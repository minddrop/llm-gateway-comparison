# 2. Common design (shared by all three clouds)

## 2.1 Okta Verify + Entra ID: integration patterns

There are three ways to combine Okta MFA with an Entra ID–centric estate:

| Pattern | How it works | Pros | Cons |
|---|---|---|---|
| **2.1.1 Entra ID External Authentication Methods (EAM): ✅ assumed** | Entra ID remains the IdP. Okta is registered in Entra as an *external authentication method*. When a Conditional Access policy requires MFA, Entra redirects the user to Okta through an OIDC exchange. Okta performs the **Okta Verify** challenge and returns a signed id_token, which Entra accepts as satisfying MFA. | One policy engine (Entra Conditional Access) evaluates MFA, **Intune compliance**, location, and risk together. Apps (LiteLLM, AWS, GCP) trust only Entra. | Users must be enrolled in Okta Verify beforehand, because EAM does not support inline enrollment. Users must be mapped between the two directories (Entra object ID ↔ Okta user). Check whether EAM is GA in your tenant cloud. |
| 2.1.2 Federation to Okta + `federatedIdpMfaBehavior` | The M365 domain is federated to Okta (WS-Fed/SAML). Okta performs primary authentication plus Okta Verify, then sends an MFA claim. Entra is configured to accept it (`acceptIfMfaDoneByFederatedIdp` / `enforceMfaByFederatedIdp`). | Works if Okta is already the M365 IdP. | Splits policy across two engines. Intune compliance is still evaluated only by Entra Conditional Access. Domain federation is a larger blast radius to change. |
| 2.1.3 Conditional Access "custom controls" | Legacy mechanism for third-party MFA. | None for a new design. | Microsoft positions EAM as the replacement. Do not use. |

**Assumed pattern: EAM (2.1.1), per [A-2].** Okta's guide is [Configure Okta as an external authentication method for Microsoft Entra ID](https://help.okta.com/oie/en-us/content/topics/apps/configure-okta-as-microsoft-entra-id-eam.htm). Microsoft's guide is [Manage an external authentication method in Microsoft Entra ID](https://learn.microsoft.com/entra/identity/authentication/how-to-authentication-external-method-manage). For the federation alternative, see [federatedIdpMfaBehavior (internalDomainFederation)](https://learn.microsoft.com/graph/api/resources/internaldomainfederation) and [Satisfy Entra MFA with MFA claims from a federated IdP](https://learn.microsoft.com/entra/identity/authentication/how-to-mfa-expected-inbound-assertions). Okta documents known EAM failure modes, including the `eam_amr_unsatisfiable` and `AADSTS50012620` errors: see [Okta EAM integration failures](https://support.okta.com/help/s/article/entra-id-external-authentication-method-eam-integration-failures-with-okta).

**Conditional Access policy set for the gateway** (one Entra app registration per gateway environment, for example `litellm-prod`):

| CA policy | Target | Grant controls |
|---|---|---|
| CA-GW-01 | All users → `litellm-prod` (web/UI + API audience) | Require MFA, satisfied by Okta EAM, **and** require a device marked compliant ([Intune compliance in CA](https://learn.microsoft.com/entra/identity/conditional-access/policy-all-users-device-compliance)) |
| CA-GW-02 | Admin role holders (`proxy_admin`) | Require MFA plus a compliant device, sign-in frequency of 4–8 h, phishing-resistant MFA where available (Okta FastPass, or Windows Hello for Business as an Entra method) |
| CA-GW-03 | Workload identities (service principals that call the gateway) | Restrict to named locations (cloud egress IPs, on-premises NAT) using Conditional Access for workload identities (requires the Workload ID premium license) |
| CA-GW-04 | Block legacy authentication and device-code flow for `litellm-prod` | Block |

## 2.2 Common login flow (sequence)

All three cloud diagrams reference this sequence as **steps L1–L7**.

```mermaid
sequenceDiagram
    autonumber
    actor U as Employee (Intune-managed device)
    participant UI as LiteLLM Admin UI<br/>(behind cloud edge/WAF)
    participant E as Microsoft Entra ID<br/>(OIDC IdP + Conditional Access)
    participant I as Microsoft Intune<br/>(compliance state)
    participant O as Okta (EAM)<br/>+ Okta Verify app
    U->>UI: L1 Browse https://llm.corp.example (UI)
    UI-->>U: L2 302 → Entra /authorize (OIDC auth code + PKCE)
    U->>E: L3 Primary auth (password / WHfB / passkey)
    E->>I: L4 Evaluate CA: device compliant? (device ID from PRT / Company Portal / Platform SSO)
    I-->>E: compliant = true
    E-->>U: L5 MFA required → redirect to Okta EAM (OIDC)
    U->>O: Okta Verify push / FastPass challenge
    O-->>E: signed id_token (amr = mfa) via browser redirect
    E-->>U: L6 Auth code → LiteLLM callback
    U->>UI: code
    UI->>E: L6b Token exchange (client secret / cert, server-to-server egress)
    E-->>UI: ID token with app roles + groups (or group overage → Graph)
    UI->>UI: L7 Map app role → LiteLLM role, groups → teams, create/update user, issue session or virtual key
```

Notes:

- Steps L3–L6 happen **between the user's device and SaaS endpoints** (`login.microsoftonline.com`, `*.okta.com`), never through the gateway network. This is important for Step 4: private circuits do not carry this traffic unless you deliberately route it that way.
- Only L1, L2, L6, and L6b touch the gateway. L6b, the back-channel token exchange, and the Graph calls for team sync are outbound internet calls from the gateway subnet, so they must be on the egress allowlist.
- LiteLLM supports Entra SSO through its Microsoft SSO integration, generic OIDC, and SAML 2.0. See [Admin UI SSO](https://docs.litellm.ai/docs/proxy/admin_ui_sso), [Microsoft SSO and team sync tutorial](https://docs.litellm.ai/docs/tutorials/msft_sso), and [SAML SSO](https://docs.litellm.ai/docs/proxy/saml_sso).

## 2.3 Authentication modes for LiteLLM API traffic

| Client type | Auth to LiteLLM | How device trust is enforced | Notes |
|---|---|---|---|
| Admin UI (browser) | Entra OIDC SSO (L1–L7) | Conditional Access CA-GW-01/02 at sign-in | Roles come from Entra **app roles** (`proxy_admin`, `proxy_admin_viewer`, `internal_user`, `internal_user_viewer`). See the [MSFT SSO tutorial](https://docs.litellm.ai/docs/tutorials/msft_sso) and [Entra app roles](https://learn.microsoft.com/entra/identity-platform/howto-add-app-roles-in-apps). |
| Developer / IDE / chat front-end (for example Open WebUI) | **Preferred:** Entra access token (JWT), validated by LiteLLM [OIDC/JWT auth](https://docs.litellm.ai/docs/proxy/token_auth) (team from the `groups` claim via `team_ids_jwt_field` / alias mapping). **Fallback:** personal virtual key with a 30–90 day expiry. | CA at token issuance, with a short token lifetime (≈60–90 min). For virtual keys, use **mTLS with an Intune-issued device certificate** at the edge, or the private path only (Step 4). | JWTs can be mapped to virtual-key-equivalent controls (budgets, TPM/RPM). See [JWT → key mapping](https://docs.litellm.ai/docs/proxy/jwt_key_mapping). |
| Internal app / AI agent (workload) | Option 1: Entra **workload identity**. The client-credentials JWT, or a federated credential from AWS/GCP/K8s, is validated by LiteLLM JWT auth. Option 2: a team-scoped **service-account virtual key** stored in the caller's secret store and rotated. | Network scoping (private path, security groups, source IP allowlist) plus mTLS workload certificates | Prefer Option 1, because it removes long-lived secrets. |
| CI/automation (key management API) | Admin virtual key held only in a secrets manager, or JWT with an admin scope | Private path only | Use the LiteLLM management APIs (`/key/generate`, `/team/new`) through IaC and pipelines. Do not use ad-hoc UI changes in production. |

**Intune device certificates for mTLS.** Intune can deploy SCEP/PKCS client certificates to managed devices ([Intune certificate profiles](https://learn.microsoft.com/intune/fundamentals/certificates/overview)). The cloud edge (AWS ALB, Azure Application Gateway, Google Cloud Load Balancing) is configured with a trust store that contains the issuing CA. The edge then accepts only connections from managed devices, which binds *device identity* to API calls that use virtual keys. When a device falls out of compliance, revoke its certificate, or set short certificate validity and have Intune reissue only to compliant devices.

## 2.4 LiteLLM tenancy model (same on all clouds)

```mermaid
flowchart TB
    subgraph Entra["Entra ID (source of truth)"]
        G1["Group: dept-finance"]
        G2["Group: team-fin-analytics"]
        G3["Group: team-fin-apps"]
        AR["App roles: proxy_admin / internal_user ..."]
    end
    subgraph LiteLLM["LiteLLM hierarchy"]
        ORG["Organization = Department<br/>(max_budget, allowed models)"]
        T1["Team = Entra group<br/>(budget, TPM/RPM, model access group)"]
        T2["Team"]
        USR["Internal user<br/>(personal budget, role)"]
        K1["Virtual key<br/>(expiry, budget, TPM/RPM, tags)"]
        TAG["Tags: cost-center, project, env<br/>(tag budgets)"]
    end
    G1 -. "SCIM / SSO sync" .-> ORG
    G2 -. "team sync" .-> T1
    G3 -. "team sync" .-> T2
    AR -. "role on login" .-> USR
    ORG --> T1 & T2
    T1 --> USR --> K1
    K1 --- TAG
```

| Need | LiteLLM feature (docs) |
|---|---|
| Onboarding / offboarding | SSO auto-provisioning on first login, plus **SCIM** from Entra for deprovisioning ([SCIM](https://docs.litellm.ai/docs/tutorials/scim_litellm)). Verify in your LiteLLM version that a deprovisioned user's keys are blocked or deleted, and add an offboarding job if they are not. |
| Roles | Entra app roles → LiteLLM roles, with highest role winning ([MSFT SSO](https://docs.litellm.ai/docs/tutorials/msft_sso)) |
| Virtual API keys | [Virtual keys](https://docs.litellm.ai/docs/proxy/virtual_keys): per-key `max_budget`, `budget_duration`, `tpm_limit`, `rpm_limit`, `models`, `expires`, `metadata.tags` |
| Team budgets / quotas | [Team budgets](https://docs.litellm.ai/docs/proxy/team_budgets), [user/team rate limits & budgets](https://docs.litellm.ai/docs/proxy/users) |
| Department / project chargeback | Orgs or tags with [tag budgets](https://docs.litellm.ai/docs/proxy/tag_budgets) and [spend tracking](https://docs.litellm.ai/docs/proxy/cost_tracking) (spend per key, user, team, tag, and model) |
| Model access policies | [Model access groups](https://docs.litellm.ai/docs/proxy/model_access): for example, `tier-standard` (small models) and `tier-premium` (frontier models) granted per team |
| Metrics | [Prometheus metrics](https://docs.litellm.ai/docs/proxy/prometheus): spend, tokens, remaining budget, latency, and failures by team and key |
| Guardrails | Pre/post-call guardrails: [Bedrock Guardrails](https://docs.litellm.ai/docs/proxy/guardrails/bedrock), [Azure Content Safety / Prompt Shields](https://docs.litellm.ai/docs/proxy/guardrails/azure_content_guardrail), [Google Model Armor](https://docs.litellm.ai/docs/proxy/guardrails/model_armor), [Presidio PII masking](https://docs.litellm.ai/docs/proxy/guardrails/pii_masking_v2) |
| Production hardening | [LiteLLM production best practices](https://docs.litellm.ai/docs/proxy/prod): Redis for cross-instance rate limits and budgets, separate DB for spend logs at scale, worker count |

**Two important behaviours to design around:**

1. **LiteLLM spend is an *estimate*.** LiteLLM computes cost from its model price map, or from custom per-model pricing that we configure. It can differ from the cloud invoice: enterprise discounts, provisioned throughput (PTU) and committed-use pricing, cached-token pricing, and taxes all cause drift. The designs therefore treat **LiteLLM as the allocation key** (who used what), and **cloud billing as the cost truth** (what we paid), and reconcile the two monthly ([§4.5](03-aws.md#45-cost-allocation--token-controls-aws), [§5.5](04-azure.md#55-cost-allocation--token-controls-azure), [§6.5](05-gcp.md#65-cost-allocation--token-controls-gcp)).
2. **Budgets and rate limits are enforced from Redis and Postgres state.** If Redis is unavailable, LiteLLM's multi-instance limits degrade. Each design uses a zone-redundant managed Redis and alerts on Redis errors.

## 2.5 Japan data residency (applies to every design)

**Requirement [A-5, A-11]:** inference, LLM processing (including guardrail checks), logs, and data pipelines stay in Japan. In practice that means four rules:

1. **Regions:** only Tokyo (primary) and Osaka (DR) are used for gateway compute, databases, caches, secrets, log stores, backups, billing exports, and analytics.
2. **Models:** LiteLLM is configured only with deployments whose *processing* is in Japan. A Japanese *endpoint* is not enough, because global and multi-country routing options still process prompts outside Japan.
3. **Pipelines:** spend exports, billing exports, SIEM copies, and dashboards are built in the same Japanese regions. No copy goes to a global or US/EU analytics location.
4. **Enforcement:** cloud policy (SCP, Azure Policy, GCP organization policy) denies anything outside Japan. Documentation and convention alone are not enough.

**Where each cloud processes inference in Japan:**

| | AWS (Bedrock) | Azure (Azure OpenAI / Foundry Models) | GCP (Vertex AI) |
|---|---|---|---|
| **Allowed option** | **In-Region** model IDs in Tokyo/Osaka, and **`jp.` geographic cross-Region inference profiles**, which route only between Tokyo and Osaka. AWS says geographic profiles keep requests inside the geography, and that abuse-detection storage, where a model needs it, happens in the destination Region ([geographic CRIS](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html), [abuse detection](https://docs.aws.amazon.com/bedrock/latest/userguide/abuse-detection.html)). | **Standard** (`Standard`) and **Regional Provisioned** (`ProvisionedManaged`) deployments in Japan East. Prompts and responses are processed in the customer-specified Azure geography (Japan East + Japan West) ([deployment types](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/deployment-types)). | **Regional endpoint `asia-northeast1`** for models that Google lists with in-region ML processing in Japan ([data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency)). |
| **Blocked option** | `global.` profiles (any commercial Region) and `apac.` profiles (Seoul, Singapore, Sydney, Mumbai, and others) | `GlobalStandard`, `GlobalProvisionedManaged`, `GlobalBatch` (any Azure region), and `DataZone*` (the APAC data zone spans several countries), plus `DeveloperTier`. Batch is a Global type, so it is blocked too. | The `global` endpoint ("may be processed in any Google Cloud location"), and models that have no Japan entry in the residency table |
| **Guardrails in Japan** | ⚠️ **Gap.** Bedrock Guardrails **Standard** tier, the tier with Japanese language support, requires cross-Region inference. From Tokyo it uses `apac.guardrail.v1:0`, whose destinations include Regions outside Japan, and there is no `jp.` guardrail profile. **Classic** tier stays in-Region but supports only English, French, and Spanish ([tiers](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-tiers.html), [guardrail profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-cross-region-support.html)). Japanese content therefore needs in-VPC guardrails ([§4.8](03-aws.md#48-security-controls-aws)). | ✅ Content Safety in **Japan East** offers content harms, Prompt Shields, and protected material. The harm classifiers are trained and tested on Japanese. **Japan West has none of these.** Foundry guardrails follow the deployment type, so regional deployments keep guardrail processing in the region ([Content Safety regions](https://learn.microsoft.com/azure/ai-services/content-safety/region-availability)). | ✅ Model Armor supports `asia-northeast1`. `asia-northeast2` is not listed ([Model Armor locations](https://docs.cloud.google.com/model-armor/locations)). |
| **Policy enforcement** | An SCP denies actions outside `ap-northeast-1`/`ap-northeast-3`. IAM on the LiteLLM task role allows `bedrock:InvokeModel*` only on `jp.*` inference profiles, in-Region foundation models, and our application inference profiles in those two Regions. AWS warns not to trust the profile prefix alone, so call `GetInferenceProfile` from each source Region and check its destinations ([supported profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html)). | Azure Policy *allowed locations* is set to `japaneast`/`japanwest`. A custom policy denies the Global, Data Zone, and Developer deployment SKUs; Microsoft documents this pattern on the deployment-types page. | The organization policy `gcp.resourceLocations` is set to `in:asia-northeast1-locations` and `in:asia-northeast2-locations` ([restrict locations](https://docs.cloud.google.com/organization-policy/restrict-locations)). LiteLLM's `vertex_location` is pinned to `asia-northeast1` and must never be `global`. |
| **Logs** | CloudWatch, S3, and the CloudTrail organization trail are all in Tokyo, with replicas only in Osaka | Log Analytics and Sentinel workspaces are in Japan East. Storage GRS and PostgreSQL geo-backups go to the paired region, Japan West ([region pairs](https://learn.microsoft.com/azure/reliability/cross-region-replication-azure)). | ⚠️ `_Default` and `_Required` log buckets are created in `global` by default. Set the organization's default log storage location to `asia-northeast1` **before** creating projects, and redirect the `_Default` sink to a regional bucket ([regionalize logs](https://docs.cloud.google.com/logging/docs/regionalized-logs), [default settings](https://docs.cloud.google.com/logging/docs/default-settings)). |
| **Billing export** | The CUR 2.0 / Data Exports bucket is in Tokyo | Cost Management exports go to a storage account in Japan East | The BigQuery billing export dataset is in `asia-northeast1`, which the export supports ([export setup](https://docs.cloud.google.com/billing/docs/how-to/export-data-bigquery-setup)) |
| **Provider abuse monitoring** | Stored in the destination Region (Tokyo or Osaka) for models that need it | The abuse-monitoring store is in the resource's geography (Japan). Human review is in scope unless we are approved for **modified abuse monitoring**; Microsoft documents reviewer location only for the EEA ([data privacy](https://learn.microsoft.com/azure/foundry/responsible-ai/openai/data-privacy)) [A-12]. | Covered by the Google Cloud terms; confirm with legal [A-12] |

**Snapshot of models processed in Japan (checked 2026-10-30; re-check before onboarding each model):**

| Cloud | Processed in Japan (allowed) | Available from Tokyo but **not** processed only in Japan (blocked) |
|---|---|---|
| AWS Bedrock | `jp.` profiles: Claude Opus 5.5, Claude Haiku 5.5, Claude Opus 4.8, Claude Sonnet 4.6, Claude Haiku 4.5, Amazon Nova 2 Lite ([Sonnet 4.6 model card example](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html)). In-Region in Tokyo: Titan Text Embeddings V2, Cohere Embed v4, Mistral Large 3, gpt-oss-120b. | Global-only from Tokyo: Claude Sonnet 5.5, Claude Fable 5.1, Claude Opus 5, Claude Sonnet 5. APAC-only: Amazon Nova Pro. ([Models at a glance](https://docs.aws.amazon.com/bedrock/latest/userguide/model-cards.html) has a per-model page for each.) |
| Azure OpenAI | Whatever Japan East offers as `Standard` or `ProvisionedManaged`. Microsoft says geography-based deployment types "arrive last" and have "no guaranteed availability date", so expect fewer and older models than in Global ([deployment types](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/deployment-types), [model availability](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure)). | Everything that Japan East offers only as Global or Data Zone |
| Vertex AI | Listed for Japan (`asia-northeast1`): Gemini 3.5 Flash, Gemini 2.5 Flash (128k), Gemini 2.5 Pro (64k), text-embedding-004, text-multilingual-embedding, Chirp 3 | Every other Gemini model. **All partner models (Claude, Mistral) and open models**: the residency tables have no Japan column for them. Nothing is listed for Osaka (`asia-northeast2`). |

**LiteLLM-specific residency settings:**

- Set `LITELLM_OFFLINE_MODE=true`. This stops the default calls to `raw.githubusercontent.com` (model cost map and other files) and the release check, and the egress firewall denies the rest ([outbound requests](https://docs.litellm.ai/docs/proxy/outbound_requests)). The enterprise license check can still contact `license.litellm.ai` if the key cannot be verified locally. That call carries the license key, not prompts; allowlist it or confirm offline licensing with the vendor.
- Use only logging callbacks whose destinations are in Japan, such as S3, GCS, or Azure Blob in a Japanese region, or self-hosted Langfuse in the same VPC. **SaaS observability or alerting destinations are not allowed** unless they are hosted in Japan. Budget alerts go by email or Teams/Slack, and the alert text must contain no prompt content.
- Router fallbacks may point only to other Japan-processing deployments. A fallback to a `global.` profile or a public API would silently break residency.
- Image URLs in prompts make LiteLLM (and Bedrock) fetch the image from wherever it is hosted ([outbound requests](https://docs.litellm.ai/docs/proxy/outbound_requests)). The egress allowlist blocks arbitrary fetches, so clients must send images inline (base64).


---

[← Previous: Overview & assumptions](00-overview.md) · [Index](../../README.md) · [Next →: Step 1: Reference architectures](02-reference-architectures.md)
