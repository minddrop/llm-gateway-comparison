# 7. Step 3: Comparison and recommendation

## 7.1 Side-by-side comparison

Ratings: ●●● = strongest / lowest effort, ●○○ = weakest / highest effort. Ratings are relative to these three designs for *our* environment, not general rankings.

| Dimension | AWS | Azure | GCP |
|---|---|---|---|
| **Official reference maturity for *this* pattern** | ●●● An official AWS guidance repo deploys **LiteLLM** itself (Terraform, ECS/EKS, private-VPC scenario, Okta JWT) | ●●○ Mature *APIM* AI gateway references and landing-zone accelerators. LiteLLM must be substituted into them. | ●●○ Mature *Apigee* AI gateway policies and samples. LiteLLM must be substituted. |
| **Managed-service maturity (supporting services)** | ●●● Aurora, ElastiCache, ALB mTLS, Network Firewall, and PrivateLink are all GA and well understood | ●●● PostgreSQL Flexible, Managed Redis, App Gateway mTLS, Azure Firewall, and Private Link are GA. Container Apps is mature but less flexible than AKS. | ●●● Cloud SQL, Memorystore, LB mTLS, Cloud Armor, PSC, and VPC-SC are GA. Cloud Run is very low-ops. |
| **Operational effort (run the gateway)** | ●●○ ECS Fargate is low-ops. More moving parts in network (TGW + inspection VPC + endpoints per service). | ●●○ ACA is low-ops. Hub firewall and Private DNS zone sprawl need landing-zone discipline. | ●●● Cloud Run, a single PSC endpoint for all Google APIs, and VPC-SC make for the smallest surface area. |
| **Native model access** | **Bedrock**: Anthropic Claude (first-party on AWS), Amazon Nova, Llama, Mistral, DeepSeek, Cohere, and others. Cross-region inference. Guardrails. | **Azure OpenAI / Foundry**: OpenAI GPT and reasoning models (first-party on Azure), plus Foundry catalogue models (partner and open). PTU. Built-in content filters. | **Vertex AI**: Gemini (first-party), plus Model Garden partner models (Claude, Llama, Mistral …). Provisioned Throughput. |
| **Identity integration with Entra ID** | ●●○ LiteLLM ↔ Entra is cloud-agnostic. AWS operator SSO via IAM Identity Center + SCIM. No native Intune device-trust provider in AWS Verified Access, so use mTLS. | ●●● Same tenant: CA, Intune, managed identities, Entra Private Access, Sentinel. Fewest integration seams. | ●●○ LiteLLM ↔ Entra is cloud-agnostic. Operators via Workforce Identity Federation. Intune signals possible via Chrome Enterprise Premium (extra license). |
| **Networking complexity** | ●●○ Many per-service interface endpoints and Resolver rules. Network Firewall routing via TGW. Repo has a ready-made private-VPC scenario. | ●●○ ~10 private DNS zones to link and forward. Firewall UDRs. ACA subnet sizing. ExpressRoute private peering is familiar to Microsoft-centric network teams. | ●●● One PSC endpoint covers all Google APIs. Shared VPC. VPC-SC adds policy design effort but not routing complexity. |
| **Cost drivers (infra; model usage dominates)** | Fargate vCPU/GB-hours, Aurora instance + I/O, ElastiCache nodes, **interface endpoints (per AZ-hour + GB)**, **Network Firewall endpoints + NAT GB**, DX port-hours and DTO, CloudWatch ingestion. The repo's sample is about USD 380/month before hardening; Network Firewall and endpoints can add more than that. | ACA dedicated workload profile, PostgreSQL HA (2× compute), Managed Redis, **App Gateway WAF v2 capacity units**, **Azure Firewall Premium** (fixed hourly + GB), private endpoints (hourly + GB), ER circuit + gateway, **Log Analytics ingestion** (often the largest surprise). | Cloud Run (instance-based with min instances), Cloud SQL Enterprise Plus HA, Memorystore, LB forwarding rules + data, Cloud Armor (policy + requests; Enterprise optional), Cloud NGFW, Interconnect VLAN attachments + egress, Cloud Logging ingestion. Apigee, if added, is a large fixed cost. |
| **Model cost-allocation fidelity** | ●●○ Tagged application inference profiles per dept/team (coarse, extra profiles to manage) + LiteLLM | ●○○ Billing per resource only. Per-team needs LiteLLM (or APIM metrics), or a resource per BU. | ●●● Per-request billing labels on Vertex (PayGo) + LiteLLM |
| **Vendor lock-in** | Low–medium: LiteLLM, Postgres, and Redis are portable. Lock-in comes from Bedrock-specific features (Guardrails, inference profiles) and the middleware. | Low–medium: same portability. Azure OpenAI deployment and PTU constructs and Content Safety are Azure-specific. | Low–medium: same portability. VPC-SC, Model Armor, and labels are GCP-specific. |
| **Fit with our existing estate** | DX already in place. Strong if Claude-on-Bedrock is the primary model family. | ER, M365, Entra, Intune, and Sentinel already in place. Strongest identity, security-ops, and procurement fit (MACC). | Interconnect already in place. Strongest if Gemini is primary or BigQuery is the finance analytics platform. |

## 7.2 Recommendation

1. **Build one LiteLLM gateway platform design that is portable across clouds**: containers + PostgreSQL + Redis, Entra OIDC/JWT, the LiteLLM tenancy model in [§2.4](01-common-design.md#24-litellm-tenancy-model-same-on-all-clouds), and IaC modules per cloud. Pick **one primary hosting cloud** and reach the other clouds' models as LiteLLM backends over egress. Optionally, use private interconnects between clouds later.
2. **Primary hosting cloud: choose by primary model family, with identity as the tie-breaker.**
   - If most usage will be **Anthropic Claude**, or the business wants the **fastest path to production**, choose **AWS**. The official guidance *is* LiteLLM, which shortens design and security review. Apply the changes in [§4](03-aws.md#4-step-2a-aws-design): Entra instead of Okta as the OIDC IdP, regional WAF + ALB mTLS instead of the CloudFront default, Network Firewall egress, Bedrock VPC endpoints, and inference-profile tagging.
   - If most usage will be **OpenAI models** and the security team values **one Microsoft control plane** (CA + Intune + Private Access + Sentinel), choose **Azure**. Expect a little more design work, because there is no LiteLLM reference to start from. You get the lowest identity and device-trust integration effort.
   - Choose **GCP** if Gemini is primary or finance wants **billing-label-level chargeback** in BigQuery. It also has the lowest ops burden.
3. **Don't stack API managers by default.** APIM or Apigee in front of LiteLLM duplicates token accounting. Add them only for a specific need, such as shared PTU pooling across non-LiteLLM consumers, or a corporate API portal mandate.
4. **Treat LiteLLM spend as the allocation key and cloud billing as the cost truth.** Run a monthly reconciliation job on every cloud.

## 7.3 Key trade-offs

| Trade-off | Implication |
|---|---|
| LiteLLM as the single policy point vs. cloud-native gateway (APIM/Apigee) | LiteLLM gives one user/team/budget model across all providers. It is open source, and the enterprise features need a license. Our team owns patching and scaling, so pin versions and track LiteLLM security advisories. |
| Virtual keys (simple) vs. Entra JWT (zero-trust) | Virtual keys are easy for developers, but they are bearer secrets with no device check at use time. JWTs re-apply Conditional Access every token lifetime, but need MSAL in clients. Use JWT by default; issue virtual keys with expiry, and only behind mTLS or the private path. |
| Single cloud host vs. multi-cloud active/active | Multi-cloud hosting gives provider-outage resilience but doubles ops and needs cross-cloud budget consistency (shared Postgres is not practical). Prefer single-cloud HA plus a cold standby IaC on a second cloud. |
| Content logging vs. privacy | Logging prompts helps abuse investigation and quality work, but creates a sensitive data store. Default to metadata only [A-10], and allow opt-in per team with separate keys and retention. |
| Managed serverless containers vs. Kubernetes | Fargate, ACA, and Cloud Run minimize ops. Kubernetes is needed for sidecars (Presidio), custom ingress, or self-hosted models. |


---

[← Previous: Step 2c: GCP](05-gcp.md) · [Index](../../README.md) · [Next →: Step 4: Private connectivity](07-private-connectivity.md)
