# 8. Step 4: Private connectivity (dedicated line) option

**The proposal:** management wants gateway and AI-agent traffic to ride the existing ExpressRoute, Direct Connect, and Cloud Interconnect circuits, the same as other internal traffic. That would replace internet-based access secured by site-to-site VPN, mTLS, and Intune device identity.

**Summary position:** do it, but **as an additional layer, not a replacement.** Make the dedicated line the *default path* for on-premises users and agents. Make the gateway *internal-only* where possible. Keep every identity, device, and encryption control from the Step 2 designs.

## 8.1 What changes in every design

| Element | Step 2 baseline (internet zero trust) | Step 4 private option |
|---|---|---|
| Gateway ingress | Public LB + WAF + mTLS, plus a private frontend via VPN | **Internal LB only** (ALB internal / App Gateway private frontend / regional internal ALB). Public listener removed, or kept only for a remote-worker ZTNA connector. |
| On-premises → cloud path | Site-to-site IPsec VPN over the internet | DX / ER / Interconnect private routing (BGP), optionally with **MACsec** or **IPsec over the circuit** |
| DNS | Public DNS for `llm.corp.example` | Split-horizon: on-premises resolvers forward `llm.corp.example` and the cloud private-link zones to cloud inbound resolvers |
| Model endpoints | Private endpoints from the gateway VPC (already private) | Unchanged. The gateway still calls models through PrivateLink, Private Endpoint, or PSC. On-premises clients never call models directly, so all usage stays metered. |
| Entra / Okta / Intune | Internet (SaaS) | **Still internet (SaaS).** Unchanged. See 8.5. |
| External model providers | Cloud egress firewall → internet | Unchanged: cloud egress firewall → internet. Optionally hairpin to the on-premises internet breakout, which is not recommended. |
| Remote workers | Internet → public WAF/mTLS edge | ZTNA (Entra Private Access) or corporate VPN → DC → dedicated line. See 8.6. |

## 8.2 AWS: Direct Connect overlay

```mermaid
flowchart LR
    subgraph ONP["On-premises data center"]
        direction TB
        UOP["Employees on corp LAN<br/>(Intune-compliant, device cert)"]
        AGOP["AI agents / apps<br/>(workload cert / Entra WI)"]
        ODNS["Corp DNS<br/>conditional forwarders:<br/>llm.corp.example, *.amazonaws.com"]
        CER["Customer edge routers<br/>(MACsec-capable)"]
        INET["Corp internet breakout<br/>+ secure web gateway"]
    end

    subgraph DX["AWS Direct Connect (2 locations × 2 connections)"]
        direction TB
        DXC["Dedicated connections<br/>MACsec (802.1AE) where supported<br/>or Private IP VPN over DX"]
        DXGW["Direct Connect Gateway<br/>(transit VIF)"]
    end

    subgraph AWS["AWS Region A"]
        direction TB
        TGW["Transit Gateway"]
        subgraph VPC["llm-gateway-prod VPC (no public subnets)"]
            R53IN["Route 53 Resolver<br/>inbound endpoints"]
            IALB["Internal ALB<br/>regional WAF + mTLS verify"]
            ECS["LiteLLM on ECS Fargate"]
            VPCE["VPC endpoints<br/>bedrock-runtime, etc."]
        end
        EGR["Egress VPC<br/>Network Firewall + NAT"]
        BR["Amazon Bedrock"]
    end

    subgraph SAAS["Internet SaaS (unchanged)"]
        ENTRA["Entra ID / Intune"]
        OKTA["Okta / Okta Verify"]
        EXT["External model APIs"]
    end

    REM["Remote worker<br/>(home / travel)"]
    ZTNA["Entra Private Access connector<br/>(EC2 in gateway VPC) or<br/>corp VPN to DC"]

    UOP -->|"P1 resolve"| ODNS
    ODNS -->|"P2 forward"| DXC
    DXC --> DXGW --> TGW
    TGW -->|"P2"| R53IN
    UOP -->|"P3 HTTPS + mTLS over DX"| CER
    AGOP -->|"P3"| CER
    CER --> DXC
    TGW -->|"P4"| IALB --> ECS
    ECS -->|"P5 PrivateLink"| VPCE --> BR
    ECS -->|"P6 internet-bound"| TGW
    TGW --> EGR -->|"P6"| EXT
    UOP -.->|"I1 sign-in / MFA (internet)"| INET
    INET -.-> ENTRA
    INET -.-> OKTA
    REM -.->|"R1 ZTNA + CA (compliant device)"| ZTNA
    ZTNA -.-> IALB
```

**Considerations (AWS):**

| Topic | Notes |
|---|---|
| What DX does and does not protect | DX is a private circuit that is not internet-routed. It **does not encrypt** traffic by default and **does not authenticate** users or devices. Encryption options are [MACsec on dedicated connections](https://docs.aws.amazon.com/directconnect/latest/UserGuide/MACsec.html) (10/100/400 Gbps at supported locations) or a [Private IP VPN over Direct Connect](https://docs.aws.amazon.com/vpn/latest/s2svpn/private-ip-dx.html) terminating on TGW. Keep TLS 1.2+ and mTLS end-to-end regardless. |
| Private routing to PaaS/model endpoints | DX transit VIF → DX Gateway → TGW → gateway VPC. Bedrock is reached only by **LiteLLM** through [interface VPC endpoints](https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html). On-premises clients do not need Bedrock DNS. On-premises DNS forwards `llm.corp.example` (a Route 53 private hosted zone) to [Route 53 Resolver inbound endpoints](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-inbound-queries.html). Avoid a **public VIF**: it exposes AWS public endpoints but bypasses VPC endpoint policies. |
| Traffic that still uses the internet | (a) User sign-in and MFA to Entra ID, Okta, and Okta Verify push, from the user's device through the corporate internet breakout. (b) LiteLLM → Entra token endpoint, JWKS, and Graph, through Network Firewall. (c) LiteLLM → external model providers, through Network Firewall with an FQDN allowlist and a fixed NAT EIP that is registered with providers that support IP allowlisting. |
| Remote / hybrid workers | They are not on the DX path. Options: **Entra Private Access** with a connector in the VPC (CA + compliant device per app); the existing corporate VPN to the DC, then DX (hairpin latency); or keep the Step 2 public mTLS edge for remote users only. [AWS Client VPN](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/what-is.html) with Entra SAML is possible but does not evaluate Intune compliance. |
| Bandwidth / latency / redundancy / cost | LLM traffic is low-bandwidth (text) but uses many long-lived streaming connections. Model inference time (seconds) dominates DX's latency advantage (milliseconds). Use the [maximum resiliency model](https://docs.aws.amazon.com/directconnect/latest/UserGuide/resiliency_toolkit.html) (2 locations × 2 connections) for 99.99%. Incremental cost is small if circuits exist: DX data transfer out, TGW attachment and processing, Resolver endpoints, and MACsec-capable ports if upgrading. |
| Ops complexity / failure modes | BGP route withdrawal or leaks, asymmetric routing between DX and VPN backup, **DNS forwarding failure** (clients fall back to public DNS and fail closed, which is good), MTU issues with IPsec over DX, TGW route-table errors that expose the gateway to other spokes, and MACsec key (CKN/CAK) rotation. Monitor with DX CloudWatch metrics, TGW flow logs, and synthetic probes from on-premises. |

## 8.3 Azure: ExpressRoute overlay

```mermaid
flowchart LR
    subgraph ONP["On-premises data center"]
        direction TB
        UOP["Employees on corp LAN"]
        AGOP["AI agents / apps"]
        ODNS["Corp DNS<br/>forward llm.corp.example +<br/>privatelink.* → Private Resolver"]
        CER["Customer edge (MSEE-facing)<br/>MACsec on ER Direct"]
        INET["Corp internet breakout"]
    end

    subgraph ER["ExpressRoute (2 circuits / 2 peering locations)"]
        direction TB
        ERC["Private peering<br/>(+ optional IPsec VPN over ER)"]
    end

    subgraph AZ["Azure Region A"]
        direction TB
        subgraph HUB["Hub VNet / vWAN hub"]
            ERGW["ExpressRoute Gateway<br/>(zone-redundant, ErGw*AZ / scalable)"]
            AFW["Azure Firewall"]
            DNSR["DNS Private Resolver<br/>(inbound endpoint)"]
        end
        subgraph SPOKE["llm-gateway spoke"]
            AGW["App Gateway WAF v2<br/>private frontend only + mTLS"]
            ACA["LiteLLM on Container Apps (internal)"]
            PE["Private endpoints<br/>Azure OpenAI, Content Safety, KV ..."]
        end
        AOAI["Azure OpenAI / Foundry"]
    end

    subgraph SAAS["Internet SaaS (unchanged)"]
        ENTRA["Entra ID / Intune"]
        OKTA["Okta / Okta Verify"]
        EXT["External model APIs"]
    end

    REM["Remote worker"]
    GSA["Entra Private Access<br/>(Global Secure Access client<br/>+ connector in spoke)"]

    UOP -->|"P1 resolve"| ODNS -->|"P2"| ERC
    ERC --> ERGW -->|"P2"| DNSR
    UOP -->|"P3 HTTPS + mTLS"| CER
    AGOP -->|"P3"| CER
    CER --> ERC
    ERGW -->|"P4 (via hub FW if inspected)"| AFW --> AGW --> ACA
    ACA -->|"P5"| PE --> AOAI
    ACA -->|"P6 0/0 UDR"| AFW -->|"P6 allowlist"| EXT
    UOP -.->|"I1 sign-in / MFA"| INET
    INET -.-> ENTRA
    INET -.-> OKTA
    REM -.->|"R1 CA: MFA + compliant device"| GSA
    GSA -.-> AGW
```

**Considerations (Azure):**

| Topic | Notes |
|---|---|
| What ER does and does not protect | ER private peering is a private Layer 3 connection through a connectivity provider. It is **not encrypted** by default and does not identify users or devices. Encryption options are [MACsec on ExpressRoute Direct](https://learn.microsoft.com/azure/expressroute/expressroute-howto-macsec) (port pairs only, not provider circuits) or [site-to-site VPN over ExpressRoute private peering](https://learn.microsoft.com/azure/vpn-gateway/site-to-site-vpn-private-peering). |
| Private routing to PaaS/model endpoints | Use **private endpoints over private peering**. Don't use Microsoft peering to reach PaaS public endpoints, because it complicates routing and bypasses private endpoint controls. On-premises DNS must conditionally forward every `privatelink.*` zone in use to the **DNS Private Resolver inbound endpoint** ([on-premises DNS for private endpoints](https://learn.microsoft.com/azure/private-link/private-endpoint-dns-integration)). Missing zones are the most common failure. |
| Traffic that still uses the internet | Entra ID, Okta, and Intune sign-in and check-in from devices. Microsoft recommends the internet for most M365 and Entra traffic, and ER Microsoft peering for M365 needs authorization ([ExpressRoute for Microsoft 365](https://learn.microsoft.com/microsoft-365/enterprise/azure-expressroute)). LiteLLM → Entra token and Graph, and external model providers, go through Azure Firewall. |
| Remote / hybrid workers | **Entra Private Access** (Global Secure Access client) is the most natural fit. The same CA policy requires a compliant device, and the connector sits in the spoke and targets the App Gateway private frontend. No hairpin through the DC is needed. Alternative: Azure VPN Gateway P2S with Entra auth, or the corporate VPN. |
| Bandwidth / latency / redundancy / cost | Size the **ER gateway SKU** for aggregate throughput and flow count. Many long-lived SSE streams count against connection limits. [FastPath](https://learn.microsoft.com/azure/expressroute/about-fastpath) bypasses the gateway on the data path, but check its limitations with Private Link. For resiliency, use 2 circuits in 2 peering locations ([ER resiliency design](https://learn.microsoft.com/azure/expressroute/design-architecture-for-resiliency)). Cost: circuit (metered vs unlimited), gateway hours, private endpoint per-GB, and firewall processing if hub-inspected. |
| Ops complexity / failure modes | Private DNS zone sprawl and drift. Asymmetric routing when hub firewall inspection is added, so UDRs must cover both directions. ER circuit maintenance windows. Gateway SKU saturation. MACsec CAK/CKN rotation in Key Vault. BGP community and route-filter mistakes. Monitor with Connection Monitor, ER metrics, and Firewall logs in Sentinel. |

## 8.4 GCP: Cloud Interconnect overlay

```mermaid
flowchart LR
    subgraph ONP["On-premises data center"]
        direction TB
        UOP["Employees on corp LAN"]
        AGOP["AI agents / apps"]
        ODNS["Corp DNS<br/>forward llm.corp.example +<br/>googleapis.com → Cloud DNS inbound"]
        CER["Customer routers<br/>(MACsec-capable)"]
        INET["Corp internet breakout"]
    end

    subgraph IC["Cloud Interconnect (99.99% topology: 2 metros × 2)"]
        direction TB
        VLAN["VLAN attachments<br/>MACsec for Cloud Interconnect<br/>or HA VPN over Interconnect"]
    end

    subgraph GCPR["Google Cloud Region A"]
        direction TB
        subgraph HOSTP["Shared VPC host"]
            CR["Cloud Router (BGP)<br/>custom adv: PSC IP, ILB range"]
            DNSIN["Cloud DNS inbound<br/>server policy"]
            PSC["PSC endpoint<br/>Google APIs"]
            NAT["Cloud NGFW + Cloud NAT"]
        end
        subgraph SVCP["llm-gateway service project (VPC-SC)"]
            ILB["Regional internal ALB<br/>Cloud Armor + mTLS"]
            RUN["LiteLLM on Cloud Run"]
        end
        VTX["Vertex AI / Model Armor"]
    end

    subgraph SAAS["Internet SaaS (unchanged)"]
        ENTRA["Entra ID / Intune"]
        OKTA["Okta / Okta Verify"]
        EXT["External model APIs"]
    end

    REM["Remote worker"]
    ZT["Entra Private Access connector<br/>(GCE in host VPC) or corp VPN"]

    UOP -->|"P1 resolve"| ODNS -->|"P2"| VLAN
    VLAN --> CR -->|"P2"| DNSIN
    UOP -->|"P3 HTTPS + mTLS"| CER
    AGOP -->|"P3"| CER
    CER --> VLAN
    CR -->|"P4"| ILB --> RUN
    RUN -->|"P5"| PSC --> VTX
    RUN -->|"P6"| NAT -->|"P6 allowlist"| EXT
    UOP -.->|"I1 sign-in / MFA"| INET
    INET -.-> ENTRA
    INET -.-> OKTA
    REM -.->|"R1 ZTNA + CA"| ZT
    ZT -.-> ILB
```

**Considerations (GCP):**

| Topic | Notes |
|---|---|
| What Interconnect does and does not protect | Dedicated or Partner Interconnect gives private L2/L3 connectivity into the VPC. It is **not encrypted** by default and has no user or device identity. Encryption options are [MACsec for Cloud Interconnect](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/macsec-overview) or [HA VPN over Cloud Interconnect](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/ha-vpn-interconnect) (IPsec). |
| Private routing to PaaS/model endpoints | On-premises clients reach only the **internal ALB**. LiteLLM reaches Vertex AI and Model Armor through **PSC for Google APIs**. Cloud Router must **custom-advertise** the internal LB subnet, and the PSC IP if on-premises tools call Google APIs directly. On-premises DNS forwards to the **Cloud DNS inbound server policy** ([Vertex PSC from on-prem](https://cloud.google.com/vertex-ai/docs/general/vertex-psc-gen-ai), [private Google access for on-prem](https://cloud.google.com/vpc/docs/configure-private-google-access-hybrid)). VPC-SC ingress rules must allow the on-premises access level if on-premises tools call Vertex directly. That is not needed when traffic always goes through LiteLLM. |
| Traffic that still uses the internet | Entra, Okta, and Intune (device ↔ SaaS); LiteLLM → Entra and Graph; LiteLLM → external providers via Cloud NGFW FQDN rules and Cloud NAT static IPs. |
| Remote / hybrid workers | Entra Private Access connector on GCE in the host VPC (CA + compliant device), the corporate VPN, or Chrome Enterprise Premium + IAP for **UI-only** access with Intune signals. |
| Bandwidth / latency / redundancy / cost | Use the [99.99% topology](https://cloud.google.com/network-connectivity/docs/interconnect/tutorials/dedicated-creating-9999-availability) (4 connections, 2 metros, 2 Cloud Routers). VLAN attachment capacity and Cloud Router route limits apply. Egress over Interconnect is cheaper than internet egress, but LLM traffic volume is small, so fixed attachment and port costs dominate. |
| Ops complexity / failure modes | Missing custom route advertisements (the most common issue). DNS forwarding loops between on-premises and Cloud DNS. VPC-SC denials that look like 403s. MACsec key rotation. Partner Interconnect provider outages. Monitor with Interconnect metrics, Network Intelligence Center Connectivity Tests, and VPC-SC dry-run logs. |

## 8.5 What private connectivity does *not* solve (all clouds)

```mermaid
flowchart TB
    subgraph L["Layer"]
        direction LR
        N["Network path isolation"]
        E["Encryption in transit"]
        D["Device identity"]
        UI["User identity & MFA"]
        AZ["Authorization (who may use which model, how much)"]
        DATA["Data protection (PII, prompt injection)"]
    end
    DL["Dedicated line<br/>(DX / ER / IC)"] -->|"✔ provides"| N
    DL -.->|"✘ unless MACsec/IPsec"| E
    DL -.->|"✘"| D
    DL -.->|"✘"| UI
    DL -.->|"✘"| AZ
    DL -.->|"✘"| DATA
    MTLS["mTLS (device/workload certs)"] -->|"✔"| E
    MTLS -->|"✔"| D
    ENTRA["Entra ID + Okta Verify (EAM)<br/>+ Conditional Access + Intune"] -->|"✔"| UI
    ENTRA -->|"✔ device compliance"| D
    LLM["LiteLLM RBAC, budgets,<br/>model access groups"] -->|"✔"| AZ
    GR["Guardrails (Bedrock / Content Safety / Model Armor / Presidio)"] -->|"✔"| DATA
```

Key points to communicate to management:

- **A private line is a path, not an identity.** Any compromised host on the corporate network, or in any VPC, VNet, or project routed to the gateway, can reach it. Without per-request user, device, and workload authentication, a private line *widens* the trusted zone.
- **Dedicated circuits are not encrypted by default.** If our policy requires link encryption, enable MACsec (where the port type supports it) or IPsec over the circuit. In all cases, keep TLS 1.2+/1.3 end-to-end to the gateway, because TLS protects against insiders on the path, not just the provider.
- **SaaS identity stays on the internet.** Entra ID, Okta Verify push, and Intune check-ins are SaaS. The dedicated line does not carry them, and a dedicated-line outage does not affect them. An *internet* outage at the DC, however, breaks sign-in even while the circuits are up. The corporate breakout must be redundant.
- **External model providers stay on the internet.** OpenAI, the Anthropic API, and similar providers are reached from the cloud egress firewall. If a provider offers private connectivity in your cloud (for example a provider's PrivateLink offering or a cloud-hosted equivalent such as Claude on Bedrock or Vertex, or OpenAI models on Azure), prefer that.

## 8.6 Remote and hybrid workers

| Option | Device trust | User experience | Recommendation |
|---|---|---|---|
| **Entra Private Access** (Global Secure Access) with connectors in each cloud VPC/VNet | Entra CA + Intune compliance per app segment ([Private Access](https://learn.microsoft.com/entra/global-secure-access/concept-private-access)) | Transparent, per-app, no full tunnel | **Recommended**: same policy engine everywhere, and it works for all three clouds |
| Corporate full-tunnel VPN → DC → dedicated line | VPN posture checks (vendor-specific) | Hairpin latency, plus VPN capacity planning | Acceptable short-term |
| Keep the Step 2 public edge (WAF + mTLS + CA) for remote users only | Intune device cert at the edge + CA | Direct, lowest latency | Acceptable. This is the "internet zero trust" path, kept intentionally for remote users. |

## 8.7 mTLS explainer

**How it works (TLS 1.3, simplified):**

```mermaid
sequenceDiagram
    autonumber
    participant C as Client (device / agent)<br/>private key in TPM / secure enclave
    participant S as Gateway edge (ALB / App GW / Google LB)<br/>trust store = Intune CA + workload CA
    C->>S: ClientHello (SNI llm.corp.example, key share)
    S-->>C: ServerHello + server certificate (public or corporate CA) + CertificateRequest (acceptable CAs)
    C->>C: Verify server cert chain & hostname
    C->>S: Client Certificate (device cert issued by Intune SCEP/PKCS)
    C->>S: CertificateVerify (signature over handshake with device private key)
    S->>S: Validate chain to trust store, validity dates, EKU, revocation (CRL/OCSP)
    S-->>C: Finished (session keys established)
    C->>S: HTTP request + Authorization: Bearer (Entra JWT / virtual key)
    S->>S: Forward cert identity headers → LiteLLM (e.g., device ID / subject)
```

| | mTLS secures | mTLS does **not** secure / limitations |
|---|---|---|
| **Authentication** | *Mutual* authentication: the client proves possession of a private key bound to a certificate from a trusted CA (device or workload identity), and the server proves its identity to the client | It is **not user authentication or authorization**. A valid device cert says nothing about *which user* or *which models and budgets*. Entra JWT, virtual keys, and LiteLLM RBAC are still required. |
| **Encryption** | Confidentiality and integrity in transit between client and TLS terminator, with forward secrecy (TLS 1.3) | Only up to the **termination point** (LB). The LB → backend hop needs its own TLS. A corporate **TLS-inspecting proxy** breaks mTLS, so exempt `llm.corp.example` from inspection. |
| **Certificate lifecycle** | Automatable: Intune SCEP/PKCS profiles for devices; Private CA / Key Vault / CAS for workloads | Issuance, renewal, and CA rotation are operational work. Expired certs cause outages, so monitor expiry and use 30–90 day validity with auto-renewal. |
| **Revocation** | CRL/OCSP checks at the edge, where supported (ALB supports CRLs; check App Gateway and Google LB capabilities per region) | Revocation is often slow or unsupported. Short-lived certs are more reliable than CRLs. Compliance loss in Intune does not automatically revoke the cert unless that is automated. |
| **Key protection** | TPM- or Secure Enclave–bound non-exportable keys (Intune can require a TPM KSP) | Software-stored keys can be copied to an unmanaged machine. |
| **Header trust** | The edge passes cert attributes to LiteLLM | The backend must accept those headers **only** from the LB. Strip client-supplied copies. |

**mTLS vs. dedicated lines:**

| Property | Dedicated line (DX / ER / IC) | mTLS |
|---|---|---|
| Layer | L1–L3 path | L4–L7 session |
| Authenticates | Nothing (the BGP peer at most) | Each client device or workload, and the server |
| Encrypts | No (unless MACsec or IPsec added) | Yes, end-to-end to the TLS terminator |
| Protects against internet exposure / DDoS | Yes (no public endpoint) | No (endpoint may be public; WAF/DDoS still needed) |
| Protects against a compromised internal host | No | Partly. The attacker also needs a valid cert and key. |
| Works for remote workers | No (needs VPN/ZTNA) | Yes |
| Deterministic bandwidth/latency, SLA | Yes | No (rides whatever path) |
| Main operational burden | Circuits, BGP, DNS forwarding, provider SLAs | PKI, cert lifecycle, revocation, proxy exemptions |

They are **complementary**: the dedicated line reduces *exposure*, and mTLS provides *cryptographic identity and encryption*.

## 8.8 Recommended zero-trust posture (combining both)

| Layer | Control (all clouds) |
|---|---|
| **User identity** | Entra ID OIDC/JWT for UI and API. Okta Verify via **EAM** for MFA, phishing-resistant for admins. Short token lifetimes. SCIM deprovisioning. |
| **Device** | Intune compliance enforced by **Conditional Access** at token issuance, **and** an Intune-issued, TPM-bound **device certificate** checked by **mTLS** at the gateway edge. |
| **Workload** | Agents authenticate with Entra workload identity (federated credentials) plus **mTLS workload certificates**. Service-account virtual keys only as an exception, with expiry, budgets, and IP scoping. |
| **Network** | **Dedicated line is the default** for on-premises users and agents to an **internal-only** gateway. **Entra Private Access** for remote users. The public edge is either removed, or kept only with WAF + mTLS + CA. Models are reached only through private endpoints. Egress is allowlisted. |
| **Transport encryption** | TLS 1.2+ (prefer 1.3) end-to-end, including LB → LiteLLM → model. **MACsec or IPsec over the circuits** if data classification requires link-layer encryption. |
| **Authorization & usage** | LiteLLM RBAC, model access groups, budgets, and TPM/RPM per org, team, user, and key. Cloud quotas as the outer limit. |
| **Data** | Guardrails on every request (PII, prompt injection). Metadata-only logging by default. CMK encryption. Immutable audit. |
| **Detect & respond** | Entra sign-in logs, WAF, firewall, and LiteLLM audit/spend anomalies in Sentinel or the SIEM. Automated key revocation on user offboarding or device non-compliance. |

> **Bottom line for management:** routing gateway traffic over ExpressRoute, Direct Connect, and Interconnect is a good choice for exposure reduction and predictable connectivity. It is **not** a substitute for user MFA, device compliance, mTLS, or encryption. Approve the private-path design together with the identity, device, and mTLS controls, not instead of them.


---

[← Previous: Step 3: Comparison](06-comparison.md) · [Index](../../README.md) · [Next →: References](08-references.md)
