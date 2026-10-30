# 7. ステップ 3: 比較と推奨

## 7.1 横並び比較

評価: ●●● = 最も強い / 作業量が最も少ない、●○○ = 最も弱い / 作業量が最も多い。評価は **日本国内のみという所在要件**（[§2.5](01-common-design.md#25-日本国内データ所在全設計に適用)）の下での、*当社* 環境における 3 つの設計間の相対評価であり、一般的なランキングではない。

| 観点 | AWS（東京 + 大阪） | Azure（Japan East + Japan West） | GCP（`asia-northeast1` + `asia-northeast2`） |
|---|---|---|---|
| **日本国内で処理されるモデル** *（決め手）* | ●●● `jp.` プロファイルで現行のフロンティア Claude モデル（Opus 5.5、Haiku 5.5、Opus 4.8、Sonnet 4.6、Haiku 4.5）と Nova 2 Lite を利用できる。東京の In-Region のモデルもある（Mistral Large 3、gpt-oss-120b、埋め込み）。最新モデルの一部（例: Claude Sonnet 5.5）は Global のみのため禁止となる。 | ●●○ 使えるのは Japan East の `Standard` と `ProvisionedManaged` だけである。Microsoft は、ジオグラフィ単位の種類は「最後に提供される」もので「提供日の保証はない」としている。そのため Global より対象モデルが少なく古いと想定する。 | ●○○ 日本国内処理として掲載されているのは少数の Gemini モデル（3.5 Flash、2.5 Flash 128k、2.5 Pro 64k）と埋め込みだけである。パートナー モデル（Claude、Mistral）はなく、大阪の記載もない。 |
| **日本国内で処理されるガードレール** | ●○○ 日本語コンテンツについて、Bedrock Guardrails は日本国内では使えない。Standard 階層は `apac` 経由でルーティングされ、Classic 階層は日本語に対応していない。VPC 内でセルフホストのガードレールが必要になる（構築・運用の作業が増える）。 | ●●● Japan East で Content Safety と Prompt Shields を利用でき、有害性分類器は日本語で学習されている。Japan West にはない。 | ●●○ Model Armor は `asia-northeast1` にあり、`asia-northeast2` にはない。 |
| **日本国内での DR（モデル層）** | ●●● `jp.` プロファイルは東京・大阪間でルーティングし、どちらからも呼び出せる。Aurora Global Database 東京 → 大阪。 | ●●○ Standard デプロイは日本のどちらのリージョンでも処理されうる。Content Safety と一部のモデルは Japan East にしかない。 | ●○○ ゲートウェイ インフラは大阪にフェイルオーバーできるが、Vertex には日本国内のモデル DR がなく、Model Armor は東京のみである。 |
| **所在要件の強制にかかる作業量** | ●●○ リージョンを拒否する SCP と、`jp.`/In-Region ARN の IAM 許可リスト。プロファイルごとに `GetInferenceProfile` で送信先を確認する必要がある。 | ●●● 許可されている場所のポリシーと、デプロイ SKU を拒否する Azure Policy の文書化された方法で足りる。Standard 回線の到達範囲は、もともと日本の地政学的リージョン内に限られる。 | ●●○ ロケーションの組織ポリシーに **加えて**、ログバケットの既定ロケーションをプロジェクト作成 *前* に設定する必要がある。Vertex の `global` エンドポイントは誤設定しやすい。 |
| ***本パターン* に対する公式リファレンスの成熟度** | ●●● AWS の公式ガイダンス リポジトリが **LiteLLM** そのものをデプロイする（Terraform、ECS/EKS、Private VPC シナリオ、Okta JWT） | ●●○ 成熟した *APIM* の AI ゲートウェイ リファレンスとランディング ゾーン アクセラレーターがある。LiteLLM に置き換える必要がある。 | ●●○ 成熟した *Apigee* の AI ゲートウェイ ポリシーとサンプルがある。LiteLLM に置き換える必要がある。 |
| **マネージド サービスの成熟度（周辺サービス）** | ●●● Aurora、ElastiCache、ALB mTLS、Network Firewall、PrivateLink はいずれも GA で、知見も豊富 | ●●● PostgreSQL Flexible、Managed Redis、App Gateway mTLS、Azure Firewall、Private Link は GA。Container Apps は成熟しているが AKS より柔軟性に劣る。 | ●●● Cloud SQL、Memorystore、LB mTLS、Cloud Armor、PSC、VPC-SC は GA。Cloud Run は運用負荷が非常に小さい。 |
| **運用負荷（ゲートウェイの運用）** | ●●○ ECS Fargate は運用負荷が小さい。ネットワークの構成要素が多く（TGW + 検査用 VPC + サービスごとのエンドポイント）、**さらにセルフホストのガードレール サービスがある**。 | ●●○ ACA は運用負荷が小さい。ハブのファイアウォールとプライベート DNS ゾーンの乱立には、ランディング ゾーンの規律が必要。 | ●●● Cloud Run、全 Google API を 1 つでカバーする PSC エンドポイント、VPC-SC により、管理対象が最も小さい。 |
| **Entra ID との ID 統合** | ●●○ LiteLLM ↔ Entra はクラウドに依存しない。AWS 運用者の SSO は IAM Identity Center + SCIM。AWS Verified Access に Intune をネイティブに扱うデバイス信頼プロバイダーがないため、mTLS を使う。 | ●●● 同一テナント: CA、Intune、マネージド ID、Entra Private Access、Sentinel。統合の継ぎ目が最も少ない。 | ●●○ LiteLLM ↔ Entra はクラウドに依存しない。運用者は Workforce Identity Federation。Intune シグナルは Chrome Enterprise Premium（追加ライセンス）経由で利用可能。 |
| **ネットワークの複雑さ** | ●●○ サービスごとのインターフェイス エンドポイントと Resolver ルールが多い。TGW 経由の Network Firewall ルーティング。リポジトリに既製の Private VPC シナリオがある。 | ●●○ リンクとフォワードが必要なプライベート DNS ゾーンが約 10 個。Firewall の UDR。ACA サブネットのサイジング。ExpressRoute プライベート ピアリングは Microsoft 中心のネットワーク チームにはなじみがある。 | ●●● 1 つの PSC エンドポイントで全 Google API をカバー。共有 VPC。VPC-SC はポリシー設計の手間が増えるが、ルーティングは複雑にならない。グローバル LB の代わりにリージョン LB を使うため、フェイルオーバーは DNS で行う。 |
| **コスト要因（ゲートウェイ インフラ。モデル利用料が支配的）** | Fargate の vCPU/GB 時間（ゲートウェイ **と** ガードレール サービス）、Aurora のインスタンス + I/O、ElastiCache のノード、**インターフェイス エンドポイント（AZ 時間 + GB 単位）**、**Network Firewall エンドポイント + NAT の GB 単位**、CloudWatch の取り込み。リポジトリのサンプルは堅牢化前で約 USD 380/月。Network Firewall とエンドポイントでそれ以上が加算されうる。 | ACA の専用ワークロード プロファイル、PostgreSQL HA（コンピューティング 2 倍）、Managed Redis、**App Gateway WAF v2 の容量ユニット**、**Azure Firewall Premium**（固定の時間課金 + GB 単位）、プライベート エンドポイント（時間 + GB 単位）、Content Safety の呼び出し、**Log Analytics の取り込み**（最大の想定外コストになりやすい）。 | Cloud Run（インスタンス ベース、最小インスタンスあり）、Cloud SQL Enterprise Plus HA、Memorystore、リージョン LB の転送ルール + データ、Cloud Armor（ポリシー + リクエスト、Enterprise は任意）、Model Armor の呼び出し、Cloud NGFW、Cloud Logging の取り込み。Apigee を追加する場合は大きな固定費になる。 |
| **閉域接続のコスト（DX / ER / Interconnect）** *（詳細と試算: [§8.9](07-private-connectivity.md#89-閉域接続のコスト日本)）* | 日本の専用ポート: **USD 0.285/時（1 Gbps）**、**USD 2.142/時（10 Gbps）**。東京／大阪から日本の DX ロケーションへのデータ **送信**: **USD 0.041/GB**。受信は無料。DX Gateway の料金はない。TGW のアタッチメント時間と GB 単位の処理料金が加わる。 | 回線の月額料金（Local / Standard、従量制または無制限）。東京と大阪は **Zone 2** で、従量制の送信は **USD 0.05/GB**。**Local** 回線（東京 → Japan East、大阪 → Japan West）は送信料金込み。Zone 2 の ER Direct 10 Gbps ポート ペア: **USD 7,000/月**。SKU ごとの ER ゲートウェイ時間料金が加わる。 | Dedicated 10 Gbps: **USD 2.328/時**。10 Gbps 以下の VLAN アタッチメント: **USD 0.10/時**。Partner 1 Gbps アタッチメント: **USD 0.2778/時**。Interconnect 経由のアジア → アジアの送信: **USD 0.042/GiB**。受信は無料。 |
| **モデルのコスト配賦の精度** | ●●○ 部門/チームごとのタグ付きアプリケーション推論プロファイル（粒度が粗く、管理するプロファイルが増える）+ LiteLLM | ●○○ 請求はリソース単位のみ。チーム単位には LiteLLM（または APIM のメトリクス）か BU ごとのリソースが必要。 | ●●● Vertex のリクエスト単位の請求ラベル（従量課金）+ LiteLLM |
| **ベンダー ロックイン** | 低〜中: LiteLLM、Postgres、Redis は可搬性がある。ロックインは Bedrock 固有の機能（推論プロファイル）とミドルウェアから生じる。 | 低〜中: 可搬性は同様。Azure OpenAI のデプロイと PTU の概念、Content Safety は Azure 固有。 | 低〜中: 可搬性は同様。VPC-SC、Model Armor、ラベルは GCP 固有。 |
| **既存環境との適合性** | DX は設置済み。Claude が主要なモデル ファミリーなら最も有力で、現行のフロンティア Claude モデルを日本国内で処理できる唯一のクラウドである。 | ER、M365、Entra、Intune、Sentinel は導入済み。必要な OpenAI モデルが Japan East で Standard として提供されて *いれば*、ID、セキュリティ運用、調達（MACC）の面で最も適合性が高い。 | Interconnect は設置済み。日本向けに掲載された Gemini モデルで足りる場合にのみ適する。 |

## 7.2 推奨

1. **クラウド間で可搬性があり、日本のリージョンに限定した LiteLLM ゲートウェイ プラットフォームの設計を 1 つ作る。** コンテナー + PostgreSQL + Redis、Entra OIDC/JWT、[§2.4](01-common-design.md#24-litellm-テナント-モデル全クラウド共通) の LiteLLM テナント モデル、[§2.5](01-common-design.md#25-日本国内データ所在全設計に適用) の所在ルール、クラウドごとの IaC モジュールで構成する。**プライマリのホスティング クラウドを 1 つ** 選び、他クラウドの **日本リージョン** のモデルには、閉域網経由で LiteLLM のバックエンドとしてアクセスする。
2. **プライマリのホスティングは AWS 東京とし、大阪を DR とする。** 日本国内処理という条件の下では、AWS は日本国内で処理されるフロンティア モデルの品ぞろえが最も広く、大阪からもモデル層が動き続け、LiteLLM を基盤とする唯一の公式リファレンスがある。[§4](03-aws.md#4-ステップ-2a-aws-設計) の変更を適用する。
   - OIDC IdP を Okta ではなく Entra にする。
   - CloudFront 既定構成の代わりにリージョン WAF + ALB mTLS を使う。
   - 送信は Network Firewall を通し、モデルの FQDN は一切許可しない。
   - Bedrock には VPC エンドポイント経由でのみ接続する。
   - `jp.` プロファイルからコピーしたアプリケーション推論プロファイルにタグを付ける。
   - リージョンを拒否する SCP を追加する。
   - ガードレールは VPC 内で実行する。日本語コンテンツでは Bedrock Guardrails が日本国内に閉じないためである。その構築・運用の予算を確保する。
3. **OpenAI モデルが必要なら、Azure OpenAI Japan East（Standard / Regional Provisioned）を 2 つ目のバックエンドとして追加する。** Direct Connect → 東京のコロケーションのルーター → ExpressRoute プライベート ピアリングの経路で接続し、トラフィックを日本国内に留める。そのガードレールには Japan East の Azure Content Safety リソースを使う。OpenAI が主要なモデル ファミリーで、必要なモデルが Japan East で既に Standard として提供されている場合は、**代わりに Azure をプライマリのホストに選ぶ。** その場合 Azure は、日本国内に閉じるガードレールの面でも、Entra/Intune との統合の面でも最良である。
4. **GCP は、日本向けに掲載されたモデルで足りる限定的な Gemini 用途にのみ使う。** Vertex 上の Claude とほとんどの Gemini モデルは日本国内で処理されず、Vertex には日本国内のモデル DR もない。
5. **既定では API 管理製品を重ねない。** LiteLLM の前段に APIM や Apigee を置くと、トークン計上が重複する。LiteLLM 以外の利用者との PTU 共有プールや、全社 API ポータルの義務化など、特定の要件がある場合にのみ追加する。
6. **LiteLLM の利用額を配賦キー、クラウド請求をコストの正として扱う。** どのクラウドでも日本国内で月次の突合ジョブを実行する。閉域網コストのうちゲートウェイの負担分（[§8.9](07-private-connectivity.md#89-閉域接続のコスト日本)）はプラットフォームの共通費として配賦する。
7. **日本のモデル スナップショットは四半期ごと、およびモデルを導入する前に再確認する。** リージョンごと・デプロイの種類ごとのモデル提供状況は、本書の更新よりも速く変わる。

## 7.3 主なトレードオフ

| トレードオフ | 影響 |
|---|---|
| 日本国内処理か、最新モデルか | 新しいモデルは、まず Global や複数国にまたがる方式で提供されることが多い。たとえば Claude Sonnet 5.5 は東京からは Global のみであり、Azure のジオグラフィ単位のデプロイは「最後に提供される」。チームは数か月モデルを待つことがある。`global.` プロファイル、Global デプロイ、公開 API による例外は認めず、要望は所在要件の責任者に回す。 |
| セルフホストのガードレール（AWS）か、マネージドのガードレール（Azure / GCP）か | AWS ではガードレールのモデル、その日本語での精度、スケーリング、パッチ適用を当社が担う。Japan East の Azure Content Safety と東京の Model Armor はマネージドだが、プライマリ リージョンにしかない。 |
| 単一のポリシー点としての LiteLLM か、クラウド ネイティブのゲートウェイ（APIM/Apigee）か | LiteLLM なら、すべてのプロバイダーにまたがる単一のユーザー/チーム/予算モデルが得られる。オープンソースだが、エンタープライズ機能にはライセンスが必要。パッチ適用とスケーリングは当社チームの責任となるため、バージョンを固定し、LiteLLM のセキュリティ アドバイザリを追跡する。 |
| 仮想キー（シンプル）か Entra JWT（ゼロトラスト）か | 仮想キーは開発者にとって扱いやすいが、使用時にデバイス チェックのないベアラー シークレットである。JWT はトークン有効期間ごとに条件付きアクセスを再適用するが、クライアントに MSAL が必要。JWT を既定とし、仮想キーは有効期限付きで、mTLS または閉域経路の背後でのみ発行する。 |
| 単一クラウドでのホスティングか、マルチクラウドのアクティブ/アクティブか | マルチクラウドのホスティングはプロバイダー障害への耐性を得られるが、運用が 2 倍になり、クラウド間で予算の整合性を保つ必要がある（Postgres の共有は現実的でない）。東京と大阪にまたがる単一クラウドでの HA と、第 2 クラウドの日本リージョンでのコールド スタンバイ IaC を推奨する。 |
| 内容のログ記録かプライバシーか | プロンプトを記録すると不正利用の調査や品質改善に役立つが、これも日本国内に置かなければならない機密データの保存先が生まれる。既定はメタデータのみ [A-10] とし、チーム単位のオプトインで、専用のキーと保持期間を設ける。 |
| マネージドのサーバーレス コンテナーか Kubernetes か | Fargate、ACA、Cloud Run は運用を最小化する。サイドカー（Presidio）、カスタム受信、GPU でホストするガードレール モデル、セルフホスト モデルには Kubernetes が必要になる。 |


---

[← 前へ: ステップ 2c: GCP](05-gcp.md) · [目次](../../README.md) · [次へ →: ステップ 4: 閉域接続](07-private-connectivity.md)
