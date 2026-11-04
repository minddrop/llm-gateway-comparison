# 4. Step 2a: AWS design

**Baseline:** the AWS sample [sample-litellm-bedrock-gateway-on-eks](https://github.com/aws-samples/sample-litellm-bedrock-gateway-on-eks) ([§3.1](02-reference-architectures.md#31-aws)) in its ECS Fargate mode (the sample defaults to EKS), with its layers L1 + L2 (Bedrock through a VPC endpoint). It is deployed in **Asia Pacific (Tokyo) `ap-northeast-1`**, with **Asia Pacific (Osaka) `ap-northeast-3`** as the DR Region. The gateway is exposed on **Route B** ([§2.6](01-common-design.md#26-ingress-model-applies-to-every-design)): an internet-facing ALB whose security group and WAF allow only our office and DC egress IPs, with Entra JWT checks in LiteLLM and, as a recommended option, mTLS with Intune certificates, all terminated in Tokyo. **Route A** is optional: Step 4 ([§8.2](07-private-connectivity.md#82-aws-direct-connect-overlay)) adds an internal ALB reached over Direct Connect. Japan residency rules from [§2.5](01-common-design.md#25-japan-data-residency-applies-to-every-design) apply throughout.

## 4.1 Architecture diagram (AWS)

```mermaid
flowchart LR
    %% ---------- Clients ----------
    subgraph CLIENTS["Clients"]
        direction TB
        U["Engineer laptop in the office<br/>browser / Claude Code / Codex CLI / IDE<br/>Intune-compliant<br/>(+ Intune cert, TPM key, if mTLS adopted)"]
        AGT["AI agents & internal apps<br/>(on-prem DC / other VPCs)"]
        OPS["Operator laptop<br/>AWS console / AWS CLI"]
    end

    %% ---------- SaaS identity ----------
    subgraph IDP["SaaS identity (internet)"]
        direction TB
        ENTRA["Microsoft Entra ID<br/>OIDC + Conditional Access"]
        INTUNE["Microsoft Intune<br/>device compliance"]
        OKTA["Okta (EAM)<br/>Okta Verify"]
    end

    %% ---------- AWS edge ----------
    subgraph EDGE["AWS edge"]
        direction TB
        R53["Route 53<br/>llm.corp.example"]
        CF["AWS WAF (regional web ACL on ALB, Tokyo)<br/>office/DC IP allowlist + rate limit<br/>+ Shield Standard<br/>(no CloudFront, see §2.6)"]
    end

    %% ---------- Operator access ----------
    subgraph OPSACC["AWS operator access (not behind the gateway)"]
        direction TB
        IDC["IAM Identity Center<br/>SAML from Entra (CA-GW-05) + SCIM<br/>permission sets: 1 h admin sessions"]
        IAM["IAM policy evaluation<br/>SCP + aws:SourceIp (office/DC)<br/>+ CloudTrail"]
    end

    %% ---------- Workload account ----------
    subgraph ACCT["AWS workload account: llm-gateway-prod (ap-northeast-1 Tokyo, 3 AZs; DR: ap-northeast-3 Osaka)"]
        direction TB
        subgraph PUB["Public subnets"]
            ALB["Internet-facing ALB (HTTPS, TLS 1.2+)<br/>SG: office/DC egress IPs only<br/>mTLS verify, recommended option<br/>(trust store + CRL)<br/>idle timeout 600 s, ACM cert"]
        end
        subgraph APP["Private app subnets"]
            ECS["ECS Fargate service<br/>LiteLLM Proxy + UI<br/>(+ Bedrock middleware)<br/>tasks across 3 AZs"]
            GRD["Guardrail service (ECS)<br/>Presidio PII + prompt-attack classifier<br/>(Japanese-capable, in VPC)"]
        end
        subgraph DATA["Private data subnets"]
            AUR[("Aurora PostgreSQL<br/>Multi-AZ<br/>keys, teams, budgets, spend")]
            REDIS[("ElastiCache Valkey/Redis<br/>Multi-AZ<br/>rate-limit & budget counters, cache")]
        end
        subgraph EP["Endpoint subnets (PrivateLink)"]
            VPCE["Interface endpoints:<br/>bedrock-runtime, bedrock,<br/>secretsmanager, kms, logs, sts,<br/>ecr.api/dkr, sagemaker.runtime<br/>+ S3 gateway endpoint"]
        end
        SM["Secrets Manager<br/>provider API keys, DB creds,<br/>LiteLLM master/salt key"]
        KMS["AWS KMS (CMKs)"]
        ECR["Amazon ECR<br/>LiteLLM images (scanned)"]
    end

    %% ---------- Egress ----------
    subgraph NET["Network account (shared)"]
        direction TB
        TGW["Transit Gateway"]
        NFW["Egress VPC:<br/>AWS Network Firewall<br/>(FQDN allowlist) + NAT GW"]
    end

    %% ---------- Models ----------
    subgraph MODELS["Model providers"]
        direction TB
        BR["Amazon Bedrock<br/>jp. profiles (Tokyo ↔ Osaka) / in-Region<br/>(app inference profiles<br/>tagged per dept/team)"]
        SMK["SageMaker AI endpoints, Tokyo<br/>(self-hosted models)"]
        EXT["Optional Japan-region backends<br/>Azure OpenAI Japan East (Standard)<br/>via private path in Japan<br/>(public model APIs: blocked)"]
    end

    %% ---------- Observability & billing ----------
    subgraph OBS["Logging, metering & billing (all in Tokyo)"]
        direction TB
        CW["CloudWatch Logs & metrics<br/>+ Managed Prometheus/Grafana"]
        S3L[("S3 log bucket<br/>(Object Lock, lifecycle)")]
        CT["CloudTrail (org trail)<br/>+ Bedrock invocation logging (opt-in)"]
        CUR["CUR 2.0 export (Tokyo bucket)<br/>+ Cost Explorer + AWS Budgets"]
        ATH["Athena / Quick Sight<br/>chargeback dashboard"]
    end

    %% ---------- Login flow ----------
    U -->|"L1 open UI"| R53
    U <-->|"L3-L5 sign-in + CA"| ENTRA
    ENTRA <-->|"L4 compliance"| INTUNE
    ENTRA <-->|"L5 MFA via EAM"| OKTA
    ECS -->|"L6b token exchange, JWKS, Graph<br/>(via egress allowlist)"| NFW

    %% ---------- API flow ----------
    U -->|"1 HTTPS + Entra JWT or key<br/>(+ Intune cert)<br/>(office breakout IP)"| R53
    R53 -->|"2 resolve"| CF
    CF -->|"3 IP allowlist, managed + rate rules"| ALB
    AGT -->|"1a DC breakout IP<br/>(+ workload cert)"| R53
    ALB -->|"4 forward (after mTLS, if adopted)"| ECS

    %% ---------- Operator flow ----------
    OPS <-->|"A2/A3 SAML sign-in + CA-GW-05"| ENTRA
    OPS -->|"A2/A3 portal, console, aws sso login"| IDC
    IDC -->|"A2/A3 temporary role credentials"| IAM
    ECS <-->|"5 auth, budget, TPM/RPM check"| REDIS
    ECS <-->|"5 key / team / budget lookup"| AUR
    ECS -->|"6 guardrail pre-call (in VPC)"| GRD
    VPCE -->|"7 InvokeModel / Converse<br/>(IAM task role)"| BR
    VPCE -->|"7"| SMK
    ECS -->|"8 optional Japan-region backend"| TGW
    TGW -->|"8 DX → Tokyo colo → ER"| EXT
    TGW --> NFW
    ECS -->|"9 spend log write"| AUR
    ECS -->|"10 logs & metrics"| CW
    ECS -->|"10 request metadata logs"| S3L
    CW --> S3L
    BR -.->|"11 tagged usage"| CUR
    AUR -.->|"12 spend export (daily)"| S3L
    S3L -.-> ATH
    CUR -.-> ATH

    %% ---------- Supporting ----------
    ECS -.->|"secrets at task start"| SM
    SM -.-> KMS
    ECS -.->|"image pull"| ECR
```

> **Diagram conventions** (matching the AWS guidance style): solid arrows show the request path with numbered steps, and dotted arrows show supporting, asynchronous, or batch flows. **L-steps** are the login flow ([§2.2](01-common-design.md#22-common-login-flow-sequence)), **numbered steps** are the API request flow ([§4.3](#43-request-flow-walkthrough-aws)), and **A2/A3** are the operator scenarios ([§2.9](01-common-design.md#29-access-scenarios-and-checkpoints)). Items in parentheses, such as the Intune certificate, apply only if mTLS is adopted.

## 4.2 Component table (AWS)

| # | AWS service | Purpose in this design | Azure equivalent | GCP equivalent |
|---|---|---|---|---|
| 1 | [Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html) (public + private hosted zones, Resolver endpoints) | DNS for `llm.corp.example`; split-horizon resolution for on-premises clients only if Route A is adopted | Azure DNS + DNS Private Resolver | Cloud DNS (+ inbound server policy) |
| 2 | [AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter.html) (regional web ACL on the ALB) + Shield Standard. [CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) is **not** used. | Office/DC IP allowlist (repeating the security group), managed rule groups tuned for LLM bodies ([§2.7](01-common-design.md#27-waf-tuning-and-timeouts-for-llm-traffic)), rate-based rules, DDoS protection. TLS terminates only in Tokyo. | App Gateway WAF (no Front Door) | Cloud Armor on the regional external ALB |
| 3 | [Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) with [mTLS verify mode](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/mutual-authentication.html) | L7 routing to LiteLLM. Security group allows only office/DC egress CIDRs (the sample's `allowlist-explicit` mode). Recommended option: validation of Intune certs and workload certs against a trust store ([§2.3.2](01-common-design.md#232-what-mtls-checks-at-the-load-balancer)). | Application Gateway v2 (mTLS) | External/Internal Application LB (mTLS) |
| 4 | [ECS on Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html) (alt: EKS) | Runs LiteLLM Proxy, the UI, and the Bedrock middleware. Stateless, autoscaled. | Azure Container Apps (alt: AKS) | Cloud Run (alt: GKE Autopilot) |
| 5 | [Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html) Multi-AZ (the EKS sample uses Aurora Serverless v2; provisioned or Serverless v2 both work) | LiteLLM DB: users, teams, keys, budgets, spend logs | Azure Database for PostgreSQL Flexible Server (zone-redundant HA) | Cloud SQL for PostgreSQL (HA) / AlloyDB |
| 6 | [ElastiCache (Valkey/Redis OSS)](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.html) Multi-AZ | Shared counters for TPM/RPM limits and budgets across tasks, plus response cache | Azure Managed Redis | Memorystore for Valkey / Redis Cluster |
| 7 | [Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) with [`jp.` geographic inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html) and In-Region models (+ [application inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-create.html) created from them) | Native foundation models processed in Japan (see the [§2.5](01-common-design.md#25-japan-data-residency-applies-to-every-design) snapshot), with tagged profiles for chargeback | Azure OpenAI Standard deployments in Japan East | Vertex AI `asia-northeast1` regional endpoint |
| 8 | In-VPC guardrail service on ECS: [Presidio PII masking](https://docs.litellm.ai/docs/proxy/guardrails/pii_masking_v2) plus a Japanese-capable prompt-attack classifier (self-hosted, optionally on SageMaker AI in Tokyo). [Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html) is used only if a Japan-only guardrail profile becomes available ([§2.5](01-common-design.md#25-japan-data-residency-applies-to-every-design)). | PII redaction, prompt-attack and jailbreak detection, content policy, called from LiteLLM | Azure AI Content Safety / Prompt Shields (Japan East) | Model Armor (`asia-northeast1`) |
| 9 | [SageMaker AI](https://docs.aws.amazon.com/sagemaker/latest/dg/whatis.html) | Optional self-hosted or fine-tuned models | Foundry managed compute / AML online endpoints | Vertex AI endpoints / GKE |
| 10 | [VPC interface endpoints (PrivateLink)](https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html) | Private access to Bedrock and AWS APIs, with endpoint policies | Private Endpoints (Private Link) | Private Service Connect (Google APIs) |
| 11 | [Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html) + [Direct Connect Gateway](https://docs.aws.amazon.com/directconnect/latest/UserGuide/direct-connect-gateways-intro.html) | Hub routing to the egress VPC; on-premises routing only for Route A and cross-cloud backends | Virtual WAN / hub VNet + ER Gateway | NCC / Shared VPC + Cloud Router |
| 12 | [AWS Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/what-is-aws-network-firewall.html) + NAT Gateway | Centralized egress with an FQDN/SNI allowlist (Entra, Graph; no public model APIs) | Azure Firewall (FQDN rules) | Secure Web Proxy / Cloud NGFW + Cloud NAT |
| 13 | [Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) + [KMS](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html) | Provider keys, DB credentials, LiteLLM master and salt keys; CMK encryption | Key Vault (secrets + keys, HSM) | Secret Manager + Cloud KMS (CMEK) |
| 14 | [ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html) (enhanced scanning) | Pinned, scanned LiteLLM images | Azure Container Registry (Defender scanning) | Artifact Registry (+ Artifact Analysis) |
| 15 | [CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html), [Managed Prometheus](https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html) / Grafana | Logs, LiteLLM Prometheus metrics, alarms | Azure Monitor / Log Analytics / App Insights / Managed Prometheus & Grafana | Cloud Logging / Cloud Monitoring (Managed Service for Prometheus) |
| 16 | [S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) (+ [Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)) | Request-metadata logs, spend exports, audit archive | Storage account (immutable blob) | Cloud Storage (Bucket Lock) |
| 17 | [CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html) + [Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) | Control-plane audit; optional content logging for regulated teams | Activity Log + diagnostic settings | Cloud Audit Logs (Admin + Data Access) |
| 18 | [CUR 2.0](https://docs.aws.amazon.com/cur/latest/userguide/what-is-cur.html), [cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html), [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) | Billing truth, tag-based allocation, budget alarms as a backstop | Cost Management exports, tags, budgets | Billing export to BigQuery, labels, budgets |
| 19 | [Athena](https://docs.aws.amazon.com/athena/latest/ug/what-is.html) + [Amazon Quick Sight](https://docs.aws.amazon.com/quick/latest/userguide/what-is.html) | Joins LiteLLM spend with CUR for chargeback/showback | Fabric / Power BI | BigQuery + Looker Studio |
| 20 | [IAM Identity Center ↔ Entra ID](https://docs.aws.amazon.com/singlesignon/latest/userguide/idp-microsoft-entra.html) | Operator access to AWS consoles and CLIs via Entra SSO (CA-GW-05) + SCIM, with 1 h admin permission sets ([§4.4](#44-identity--access-aws)) | Native (Entra) | Workforce Identity Federation |
| 21 | [AWS Private CA](https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html) (only if mTLS is adopted) | Issuing CA for workload mTLS certs (agents ↔ ALB) | Key Vault certificates / partner CA | Certificate Authority Service |
| 22 | [AWS Organizations SCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) | Residency guardrail: deny actions outside `ap-northeast-1`/`ap-northeast-3`, and deny invoking `global.`/`apac.` inference profiles | Azure Policy (allowed locations, denied deployment SKUs) | Organization policy `gcp.resourceLocations` |

## 4.3 Request flow walkthrough (AWS)

**Login (L1–L7), as in [§2.2](01-common-design.md#22-common-login-flow-sequence).** The UI is served by the same LiteLLM service. LiteLLM is registered in Entra as an OIDC web app with redirect URI `https://llm.corp.example/sso/callback`. The back-channel token exchange (L6b) and the Microsoft Graph group/team sync leave through the Network Firewall, which allows `login.microsoftonline.com` and `graph.microsoft.com`.

**API request:**

1. **Client → DNS.** Claude Code, Codex CLI, an SDK, or a chat UI on an Intune-managed laptop in the office calls `https://llm.corp.example/v1/messages`, `/v1/responses`, or `/v1/chat/completions` with an **Entra JWT** from its token command. An IDE plugin that accepts only a pasted key sends a **virtual key** instead ([§2.3.4](01-common-design.md#234-coding-agent-clients)). The request leaves through the office internet breakout, which does not TLS-inspect this hostname [A-15]. If mTLS is adopted, the client presents the user's **Intune-issued certificate**. AI agents in the on-premises DC (1a) leave through the DC breakout, presenting a **workload certificate** if mTLS is adopted. With Route A, both use Direct Connect instead ([§8.2](07-private-connectivity.md#82-aws-direct-connect-overlay)).
2. **Route 53** resolves the name to the ALB.
3. **AWS WAF (regional web ACL attached to the ALB)** allows only our office/DC egress IPs (an IP set that repeats the ALB security group). It then applies a per-IP rate limit, Amazon IP reputation lists, and the Core rule set tuned for LLM bodies ([§2.7](01-common-design.md#27-waf-tuning-and-timeouts-for-llm-traffic)), plus stricter rules on admin paths. **Design decision:** as in the EKS sample, the ALB is the only front door and CloudFront is not used ([§2.6](01-common-design.md#26-ingress-model-applies-to-every-design)). This is not about mTLS, because CloudFront can now verify client certificates itself. It is about residency (TLS termination and an edge WAF at locations not limited to Japan, and WAF management in us-east-1), CloudFront's 30 s default origin timeout, and the absence of anything to cache. Do not add CloudFront later.
4. **ALB.** The security group admits only office/DC egress CIDRs, never `0.0.0.0/0`. If mTLS is adopted, during the TLS handshake the ALB verifies the client certificate chain against its trust store (the CA that Intune deploys from, plus the workload CA), the proof of possession of the private key, the validity dates, and the CRL stored in S3 ([§2.3.2](01-common-design.md#232-what-mtls-checks-at-the-load-balancer)), and forwards the `X-Amzn-Mtls-Clientcert-*` headers so that LiteLLM can log the certificate serial. It routes to the LiteLLM target group on ECS tasks in 3 AZs. The idle timeout is 600 s ([§2.7](01-common-design.md#27-waf-tuning-and-timeouts-for-llm-traffic)).
5. **LiteLLM authn/authz.** LiteLLM validates either the Entra JWT (signature via Entra JWKS, plus `aud`, `iss`, and expiry checks) or the virtual key, by hash lookup in Aurora with a Redis cache. It does not compare the certificate's user with the token or key owner ([§2.3.3](01-common-design.md#233-certificateuser-binding-not-adopted)). LiteLLM then resolves the user, team, and org, and checks model access groups, key, user, and team **budgets**, and **TPM/RPM limits** using Redis counters. Requests over a limit get `429`, and requests over budget get `400`/`429` with a budget-exceeded error.
6. **Guardrails (pre-call).** LiteLLM calls the **in-VPC guardrail service** (Presidio for PII masking, plus a self-hosted prompt-attack classifier that handles Japanese) on ECS in Tokyo. Bedrock Guardrails is *not* used for Japanese content: its Standard tier routes checks through the `apac` guardrail profile, and its Classic tier does not support Japanese ([§2.5](01-common-design.md#25-japan-data-residency-applies-to-every-design)).
7. **Native models.** LiteLLM calls `bedrock-runtime` through the **interface VPC endpoint**, using the ECS **task IAM role** ([task roles](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)) with no static keys. The model ID is the **team's application inference profile ARN**, created by copying a **`jp.` system-defined profile** (or an In-Region model), so Bedrock usage carries `CostCenter`/`Team` tags and requests are processed only in Tokyo or Osaka. The endpoint policy and the task-role policy allow only those ARNs. The SCP blocks `global.*` and `apac.*` profiles even if someone adds them to the LiteLLM config.
8. **Other Japan-region backends (optional).** Public model APIs (OpenAI, the Anthropic API, and similar) are blocked [A-7]. If the business needs OpenAI models, LiteLLM calls an **Azure OpenAI Standard deployment in Japan East**. The path runs ECS → TGW → Direct Connect → a router in the Tokyo colocation facility → ExpressRoute private peering → a private endpoint, so traffic never leaves Japan ([§8.2](07-private-connectivity.md#82-aws-direct-connect-overlay)). Do not use the internet path for this backend. It would put a model FQDN on the internet allowlist, and internet routing is not guaranteed to stay inside Japan. Provider API keys, or Entra workload-identity credentials, come from **Secrets Manager**.
9. **Response and metering.** LiteLLM streams the response, runs post-call guardrails if configured, and writes a **spend log** row to Aurora with user, team, key, tags, model, prompt and completion tokens, cost, and latency. Budget counters are updated in Redis and persisted to Postgres asynchronously.
10. **Logs and metrics.** All of these stores are in Tokyo. Container stdout (JSON) goes to **CloudWatch Logs**. LiteLLM's Prometheus `/metrics` endpoint is scraped into **Amazon Managed Prometheus** with Grafana dashboards. The LiteLLM **S3 logging callback** writes request metadata, without content per [A-10], to the S3 log bucket ([LiteLLM logging](https://docs.litellm.ai/docs/proxy/logging)).
11. **Billing.** Bedrock usage appears in **CUR 2.0** with inference-profile tags. External-provider invoices arrive separately.
12. **Chargeback.** A daily job exports LiteLLM spend (by team, tag, and model) to S3 in Tokyo. **Athena** and **Quick Sight**, both in Tokyo, join it with the CUR 2.0 export (also delivered to a Tokyo bucket) and publish showback per department. Month-end true-up: allocate the actual invoice amount in proportion to LiteLLM-measured usage.

## 4.4 Identity & access (AWS)

| Topic | Design |
|---|---|
| SSO to LiteLLM UI | Entra **OIDC** app (`litellm-prod`) using LiteLLM's Microsoft SSO settings (`MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET`, `MICROSOFT_TENANT`). The client secret is stored in Secrets Manager. Prefer a certificate credential if your LiteLLM version supports one; otherwise rotate the secret every 90–180 days. SAML 2.0 is an alternative ([LiteLLM SAML](https://docs.litellm.ai/docs/proxy/saml_sso)). |
| SSO / auth to API | LiteLLM **JWT auth** with Entra as the issuer. Claude Code (`apiKeyHelper`) and Codex CLI (`auth.command`) run a token command that gets tokens for `api://litellm-prod/.default` through MSAL with the OS broker ([§2.3.4](01-common-design.md#234-coding-agent-clients)). `team_ids_jwt_field: groups` (or app roles) maps tokens to LiteLLM teams ([JWT auth](https://docs.litellm.ai/docs/proxy/token_auth)). Virtual keys are a fallback, with mandatory expiry, for tools that accept only a pasted key. |
| Group-to-role mapping | Entra **app roles** map to LiteLLM global roles (`proxy_admin`, `proxy_admin_viewer`, `internal_user`, `internal_user_viewer`). Entra **security groups** prefixed `llm-team-*` map to LiteLLM teams through team sync on login ([MSFT SSO tutorial](https://docs.litellm.ai/docs/tutorials/msft_sso)). **SCIM** handles deprovisioning ([SCIM](https://docs.litellm.ai/docs/tutorials/scim_litellm)). Only *assigned* groups are emitted in the token, which avoids the >200-group overage problem. |
| Intune compliance | Enforced by CA-GW-01 at Entra sign-in and at every token issuance and refresh. If mTLS is adopted, also **on every request** by ALB mTLS with Intune certificates, which is the only device check on virtual keys ([§2.3.2](01-common-design.md#232-what-mtls-checks-at-the-load-balancer)). The certificate's user is not matched to the token ([§2.3.3](01-common-design.md#233-certificateuser-binding-not-adopted)). AWS services do not verify the end user; LiteLLM does ([§2.3.1](01-common-design.md#231-who-verifies-what-on-each-request)). AWS Verified Access is *not* used. Its device-trust providers are Jamf, CrowdStrike, and JumpCloud, not Intune ([Verified Access device trust](https://docs.aws.amazon.com/verified-access/latest/ug/device-trust.html)). |
| Service-to-service | ECS task role → Bedrock, Secrets Manager, KMS, and S3, using least-privilege policies plus VPC endpoint policies. Aurora uses **IAM DB authentication** or a rotated Secrets Manager credential. Agents → LiteLLM use Entra workload identity JWTs. For agents running in AWS, use [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) with Entra federated credentials, or a team service-account virtual key in the caller's Secrets Manager, combined with an mTLS workload cert from AWS Private CA if mTLS is adopted. |
| Operators (scenarios A2, A3 in [§2.9](01-common-design.md#29-access-scenarios-and-checkpoints)) | IAM Identity Center federated to Entra with SCIM ([guide](https://docs.aws.amazon.com/singlesignon/latest/userguide/idp-microsoft-entra.html)). The Identity Center enterprise app in Entra is covered by **CA-GW-05**: phishing-resistant MFA, compliant device, office/DC named locations, and device-code flow blocked. Admin permission sets use a **1 h session duration** (the maximum is 12 h). An SCP or the permission-set policy denies calls whose `aws:SourceIp` is not an office or DC egress IP, with an `aws:ViaAWSService` exception so that AWS services calling on the operator's behalf still work. The AWS CLI signs in with `aws sso login`, which uses a browser on the same machine; do not use `--use-device-code`, which lets the credentials land on a different machine from the one that signed in. Break-glass IAM users are kept in a vault. No interactive access to Fargate tasks; ECS Exec is disabled in production. |

## 4.5 Cost allocation & token controls (AWS)

| Control | LiteLLM layer (real-time enforcement) | AWS layer (billing truth / backstop) |
|---|---|---|
| Department chargeback | LiteLLM **organization** per department, or the tag `dept:<code>` on every key and team | Bedrock **application inference profile** per department, tagged `CostCenter=<code>`. LiteLLM model aliases (for example `claude-sonnet`) resolve to the caller-team-specific profile ARN using per-team model lists or deployment aliases. Activate the cost allocation tags; activation is not retroactive. |
| Team / project | Team budget (`max_budget`, `budget_duration: 30d`), TPM/RPM, `metadata.tags: [project:x]` | Inference profiles per team only where volume justifies it, because they add operational overhead. Otherwise, allocation comes from LiteLLM spend × CUR total. |
| Per user | User budget and default personal-key budget; key expiry | IAM-principal cost allocation (April 2026) applies to *AWS principals*. All calls arrive as the single task role, so it does not identify end users. Per-user truth therefore comes from LiteLLM. |
| Token controls | TPM/RPM per key, user, and team; `max_tokens` caps per model; model access groups; [tag budgets](https://docs.litellm.ai/docs/proxy/tag_budgets) | Bedrock **service quotas** (TPM/RPM per model) set the upper bound. For `jp.` profiles, quotas are managed **per source Region**: a Tokyo quota increase does not apply to calls from Osaka ([JP CRIS launch post](https://aws.amazon.com/blogs/machine-learning/introducing-amazon-bedrock-cross-region-inference-for-claude-sonnet-4-5-and-haiku-4-5-in-japan-and-australia)). Request the DR quota in Osaka in advance. Use reserved capacity only if needed. |
| Alerts | LiteLLM budget alerts (Slack/email webhooks); Prometheus alerts on `remaining_budget` | **AWS Budgets** on the gateway account and per `CostCenter` tag, at 80/100% thresholds, with SNS notifications |
| Reporting | LiteLLM UI usage tab, `/spend/*` APIs | CUR 2.0 → Athena → Quick Sight, joined to LiteLLM spend exports on `team_id`/tag and date |

## 4.6 Networking (AWS)

- **Accounts and VPCs:** a `network-prod` account owns the Transit Gateway, the DX Gateway, and the inspection/egress VPC. The `llm-gateway-prod` account has one VPC, `10.60.0.0/20`, across 3 AZs:
  - `public` (/26 ×3): the internet-facing ALB for Route B. Omit these subnets only if policy forces a private-only deployment (Route A alone).
  - `app` (/24 ×3): Fargate tasks.
  - `data` (/26 ×3): Aurora and ElastiCache.
  - `endpoints` (/26 ×3): interface endpoints for `bedrock-runtime`, `bedrock`, `bedrock-agent-runtime` (if used), `sagemaker.runtime`, `secretsmanager`, `kms`, `logs`, `monitoring`, `sts`, `ecr.api`, `ecr.dkr`, `elasticloadbalancing`, plus the **S3 gateway endpoint**. Private DNS is enabled on the endpoints. Route 53 Resolver inbound endpoints let on-premises DNS resolve them ([Resolver](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html)).
- **Security groups:** the ALB SG allows 443 only from office and DC egress CIDRs [A-14], never `0.0.0.0/0` (the EKS sample's `allowlist-explicit` mode), and the WAF IP set repeats the list. If Route A is added, its internal ALB SG allows the corporate ranges that arrive over TGW. The tasks SG allows 4000 from the ALB SGs only. Data SG allows 5432/6379 from the tasks SG only. Endpoints SG allows 443 from the tasks SG.
- **Egress:** there is no IGW route from the app subnets. The default route goes via TGW to the **inspection VPC** with **Network Firewall** stateful rules that allowlist TLS SNI for `login.microsoftonline.com`, `graph.microsoft.com`, the LiteLLM license endpoint if required ([§2.5](01-common-design.md#25-japan-data-residency-applies-to-every-design)), and `*.okta.com` only if LiteLLM calls Okta (it normally does not). **No model provider FQDN is on the internet allowlist** [A-7]. A Japan-region Azure OpenAI backend is reached over the private path instead. Everything else is denied and logged, which also catches any attempt to reach a non-Japan endpoint.
- **Ingress paths:** (a) **Route B (primary):** internet → regional WAF → internet-facing ALB (office/DC IPs only), with mTLS if adopted. No site-to-site VPN is needed. (b) **Route A (optional, [§8.2](07-private-connectivity.md#82-aws-direct-connect-overlay)):** Direct Connect → TGW → a second, *internal* ALB in front of the same ECS service, which matches the sample's `internal` mode. Split-horizon DNS chooses the route. ALB IP addresses change over time; if the office firewall can only allow fixed destination IPs, put AWS Global Accelerator in front of the ALB or use Route A ([§2.6](01-common-design.md#26-ingress-model-applies-to-every-design)).

## 4.7 Data & logging (AWS)

| Data | Store | Protection | Retention (proposed) |
|---|---|---|---|
| LiteLLM config DB (users, teams, keys as hashes, budgets) | Aurora PostgreSQL in Tokyo, Multi-AZ, encrypted with a CMK, PITR on | SG isolation, IAM auth, TLS required (`rds.force_ssl`) | Life of the platform; PITR 35 days; daily snapshot copied to Osaka only |
| Spend logs (per request) | Aurora (`LiteLLM_SpendLogs`). At high volume, use a separate DB or partitioning, per [LiteLLM prod guide](https://docs.litellm.ai/docs/proxy/prod) | Same | 90 days hot in Postgres. Daily export to S3 Parquet, kept 7 years for finance [assumption]. |
| Rate-limit counters and cache | ElastiCache Valkey/Redis, Multi-AZ, encryption in transit and at rest, AUTH/RBAC | SG isolation | Ephemeral |
| Request metadata logs | S3 in Tokyo (LiteLLM S3 callback) → Athena | SSE-KMS, bucket policy restricted to VPC endpoint, Object Lock (governance) | 1 year hot, then Glacier until 7 years |
| Prompt/response content (opt-in teams only) | Bedrock model invocation logging → dedicated S3 bucket in Tokyo, or LiteLLM callback with a per-team flag | Separate KMS key and access role, Macie scanning | Per team's legal requirement (default 30 days) |
| Admin audit (who changed keys and budgets) | LiteLLM audit logs (enterprise) → S3; CloudTrail org trail (home Region Tokyo) | Immutable bucket in the log-archive account | 7 years |
| Infrastructure logs | CloudWatch Logs in Tokyo (ALB access logs, WAF logs, Network Firewall flow/alert logs, VPC Flow Logs) | KMS, cross-account subscription to the SIEM (SIEM storage must also be in Japan) | 90 days in CloudWatch, 1–7 years in S3 |

## 4.8 Security controls (AWS)

- **WAF:** an office/DC IP set, a per-IP rate-based rule, Amazon IP reputation, Known Bad Inputs, and the Core rule set with `SizeRestrictions_BODY` overridden to Count on the API paths. That rule blocks every body over 8 KB, which is nearly every coding-agent request. Review the body XSS/LFI/RFI rules for false positives on code ([§2.7](01-common-design.md#27-waf-tuning-and-timeouts-for-llm-traffic)). AWS WAF inspects only the first 8 KB of a body on an ALB, so it cannot enforce a total body size; LiteLLM's request-size limit does that. A separate, stricter rule set applies to the `/ui`, `/sso`, and `/key/*` admin paths. Redact `authorization` and `x-api-key` in WAF logs ([§2.8](01-common-design.md#28-who-can-see-prompts)).
- **mTLS (recommended option; required in the cases in [§2.3.2](01-common-design.md#232-what-mtls-checks-at-the-load-balancer)):** ALB [mutual authentication verify mode](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/mutual-authentication.html), with a trust store containing the Intune issuing CA and the AWS Private CA. The Intune certificate profile keeps the private key non-exportable in the TPM or Secure Enclave. A scheduled job publishes the current CRL to S3 and refreshes the trust store's revocation list. Certificate validity is 30–90 days with auto-renewal.
- **Secrets:** Secrets Manager with automatic rotation for the DB credential. Provider keys rotate on a 90-day schedule using a provider-specific rotation Lambda or a manual runbook. The LiteLLM `LITELLM_MASTER_KEY` and `LITELLM_SALT_KEY` are injected only as ECS secrets. **Never rotate the salt key without a re-encryption plan**, because it encrypts the stored provider credentials.
- **Encryption:** TLS 1.2+ everywhere (ALB security policy `ELBSecurityPolicy-TLS13-1-2-2021-06`). KMS CMKs for Aurora, ElastiCache, S3, CloudWatch Logs, and Secrets Manager.
- **Guardrails / PII:** the in-VPC guardrail service is the default pre- and post-call guardrail for all models: Presidio for PII, plus a Japanese-capable prompt-attack and content classifier. Re-evaluate Bedrock Guardrails when AWS offers a guardrail profile that stays in Japan, or Classic-tier support for Japanese. LiteLLM guardrails are configured per team or key.
- **Residency guardrails:** an SCP denies all Regions except Tokyo and Osaka (global services excepted), and denies `bedrock:InvokeModel*` on `global.*` and `apac.*` inference profiles. An AWS Config rule flags resources outside the two Regions. Every new Bedrock model is onboarded only after `GetInferenceProfile` confirms that its destinations are Tokyo and Osaka.
- **Supply chain:** pin the LiteLLM image digest, mirror it to ECR, run ECR enhanced scanning (Inspector), and promote images through dev → staging → prod.
- **Detection:** GuardDuty (incl. ECS runtime monitoring), Security Hub, Config rules, and IAM Access Analyzer.

## 4.9 Scalability & HA (AWS)

- **Compute:** ECS Fargate service with at least 3 tasks spread across 3 AZs. Target-tracking autoscaling on CPU (~60%) and ALB `RequestCountPerTarget`. Set Uvicorn workers to vCPU count. ALB idle timeout and LiteLLM `request_timeout` are both 600 s ([§2.7](01-common-design.md#27-waf-tuning-and-timeouts-for-llm-traffic)).
- **Alternative:** EKS (the sample's default; Pod Identity, HPA, PodDisruptionBudgets) if the platform team already runs EKS or needs the sample's L4 cross-account access. The sample supports either, but not both in one deployment.
- **Data tier:** Aurora Multi-AZ with a reader for the UI and analytics. ElastiCache Multi-AZ with automatic failover.
- **Model capacity:** `jp.` profiles spread load across Tokyo and Osaka. LiteLLM router **fallbacks** handle the rest, but only to other Japan-processing deployments (for example Claude Opus 5.5 `jp.` → Claude Opus 4.8 `jp.`, or → Azure OpenAI Japan East), with `cooldown` on 429/5xx. Never fall back to a `global.` profile or a public API.
- **DR:** warm standby in **Osaka**, deployed from the same IaC. `jp.` profiles can be called from Osaka as the source Region, so the model layer survives a Tokyo outage. Aurora Global Database Tokyo → Osaka (RPO about 1 s) and a Route 53 failover record. The Osaka ALB needs the same security group allowlist and WAF rules, plus the same trust store and CRL job if mTLS is adopted. Redis is rebuilt cold, which briefly resets rate-limit counters but keeps budgets in Postgres. RTO is 30–60 min.


---

[← Previous: Step 1: Reference architectures](02-reference-architectures.md) · [Index](../../README.md) · [Next →: Step 2b: Azure](04-azure.md)
