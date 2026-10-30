# Multi-Cloud LLM Gateway Architecture Comparison (AWS / Azure / GCP)

> **Scope:** This document covers architecture only (no implementation code). It describes an internal LLM gateway platform built on **LiteLLM Proxy**. The platform provides user and team management, cost allocation, and token controls. Each design is built for our environment: **Microsoft Entra ID** for identity, **Okta Verify** for MFA, **Microsoft Intune** for device trust, and existing **ExpressRoute / Direct Connect / Cloud Interconnect** links.
>
> **Status:** Draft for architecture review, as of 2026-10-30. Vendor features change quickly. Every vendor claim links to official documentation, and you should check those links again before you make a final decision.

---

## 0. Executive summary

- **LiteLLM is the control point on every cloud.** It owns virtual keys, the org/team/user/key hierarchy, budgets, TPM/RPM limits, model access groups, and spend logs. The cloud-native services around it supply what LiteLLM lacks: an edge and WAF, private networking, managed Postgres and Redis, secrets, guardrails, logging, and billing reconciliation.
- **Identity is the same on all three clouds.** Entra ID is the only OIDC/SAML IdP for LiteLLM. Okta Verify is attached to Entra ID as an **External Authentication Method (EAM)**, so a single Conditional Access engine evaluates both MFA and **Intune device compliance**. The clouds differ only in how their own control planes federate with Entra ID.
- **Virtual keys are bearer secrets.** Entra Conditional Access, and therefore Intune compliance, is checked only when a user *signs in*. Nothing re-checks it when someone *uses* a LiteLLM virtual key. The designs close this gap with three controls: (a) Entra-issued JWTs (LiteLLM OIDC/JWT auth) for interactive and developer clients, (b) **mTLS with Intune-deployed device certificates** at the edge, or the dedicated-line path from Step 4, and (c) short key lifetimes plus per-key network scoping for service accounts.
- **Recommendation (detail in [§7](06-comparison.md#7-step-3-comparison-and-recommendation)):** If the deciding factor is native model access to Bedrock, put the primary gateway on **AWS**. The official guidance in the AWS repository is the closest match to what we need, and it is the only one of the three built on LiteLLM. If the model mix is mostly Azure OpenAI or Foundry models and you want the least identity and device-trust integration work, choose **Azure**. In either case, deploy LiteLLM the same way everywhere (containers, Postgres, Redis), so the gateway stays portable and other clouds' models can be reached as LiteLLM backends.
- **Dedicated lines (Step 4):** Use ExpressRoute, Direct Connect, and Interconnect as the *default path* for on-premises users and AI agents. A private path only isolates traffic. It does not authenticate users or devices, and it does not encrypt anything by itself. Keep Entra/Okta/Intune and TLS (mTLS for workloads) on top of it, and add MACsec or IPsec over the circuit if your policy requires link encryption. Remote workers still need an internet path, built on zero-trust network access (ZTNA) and Conditional Access.

---

## 1. Assumptions

Where our environment details are unknown, these designs assume the following. Each one is marked **[A-n]** and referenced throughout the document.

| ID | Assumption | Why it matters / how to validate |
|---|---|---|
| A-1 | Entra ID is the authoritative IdP for all employees, and every user has a cloud-synced UPN and group membership (synced from on-premises AD or cloud-only). | Drives OIDC claims and LiteLLM team sync. |
| A-2 | Okta is used **only as the MFA provider** (Okta Verify push, TOTP, or FastPass). It is **not** the primary federation IdP for M365. Pattern chosen: **Entra ID External Authentication Methods (EAM)** (see [§2.1](01-common-design.md#21-okta-verify--entra-id-integration-patterns)). | If our M365 domains are actually *federated to Okta*, use the alternative pattern in [§2.1.2](01-common-design.md#21-okta-verify--entra-id-integration-patterns). |
| A-3 | Entra ID P1/P2 licensing is in place (Conditional Access, group claims, app roles). Intune enforces device compliance for Windows and macOS, and for iOS/Android where needed. | Conditional Access with "require compliant device" needs P1 or higher. |
| A-4 | Users reach the LiteLLM admin UI from a browser on an Intune-managed device. API clients include IDE plugins, SDKs, internal apps, and **AI agents hosted in the on-premises DC or in the cloud**. | Determines the authentication mode per client type ([§2.3](01-common-design.md#23-authentication-modes-for-litellm-api-traffic)). |
| A-5 | One primary region per cloud plus a warm-standby DR region. Data residency is limited to a single geography (for example Japan or the EU). | Region choice limits which models are available. Check model availability per region. |
| A-6 | We buy a **LiteLLM Enterprise license**. In current docs, SSO for more than 5 users, OIDC/JWT auth, SCIM, audit logs, and some guardrail and logging integrations are enterprise features. | Without the license, SSO for more than 5 users and JWT auth are unavailable. See [LiteLLM Enterprise](https://docs.litellm.ai/docs/enterprise) and [Admin UI SSO](https://docs.litellm.ai/docs/proxy/admin_ui_sso). |
| A-7 | External model providers (for example OpenAI, Anthropic API, Mistral) are reached over the internet through an egress allowlist, because they have no private connectivity option in our region. | Affects egress design and Step 4. |
| A-8 | The existing ExpressRoute, Direct Connect, and Interconnect circuits terminate in a hub/transit network that is already peered with landing-zone spokes. Each has at least 2 circuits at 2 locations. | Affects HA claims in Step 4. Validate with the network team. |
| A-9 | Expected scale for sizing discussions: about 5,000 employees, about 300 concurrent users at peak, and roughly 50–200 RPS through the gateway with streaming responses. | Used only to discuss cost drivers. No pricing is quoted. |
| A-10 | By default, prompt and response **content** is *not* stored. Only metadata (user, team, model, tokens, cost, latency, guardrail verdict) is logged. Content logging is a per-team opt-in for compliance use cases. | Drives the privacy and retention design ([§4.7](03-aws.md#47-data--logging-aws), [§5.7](04-azure.md#57-data--logging-azure), [§6.7](05-gcp.md#67-data--logging-gcp)). |


---

[Index](../../README.md) · [Next →: Common design](01-common-design.md)
