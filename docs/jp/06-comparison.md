# 7. ステップ 3: 比較と推奨

## 7.1 横並び比較

評価: ●●● = 最も強い / 作業量が最も少ない、●○○ = 最も弱い / 作業量が最も多い。評価は *当社* 環境における 3 つの設計間の相対評価であり、一般的なランキングではない。

| 観点 | AWS | Azure | GCP |
|---|---|---|---|
| ***本パターン* に対する公式リファレンスの成熟度** | ●●● AWS の公式ガイダンス リポジトリが **LiteLLM** そのものをデプロイする（Terraform、ECS/EKS、Private VPC シナリオ、Okta JWT） | ●●○ 成熟した *APIM* の AI ゲートウェイ リファレンスとランディング ゾーン アクセラレーターがある。LiteLLM に置き換える必要がある。 | ●●○ 成熟した *Apigee* の AI ゲートウェイ ポリシーとサンプルがある。LiteLLM に置き換える必要がある。 |
| **マネージド サービスの成熟度（周辺サービス）** | ●●● Aurora、ElastiCache、ALB mTLS、Network Firewall、PrivateLink はいずれも GA で、知見も豊富 | ●●● PostgreSQL Flexible、Managed Redis、App Gateway mTLS、Azure Firewall、Private Link は GA。Container Apps は成熟しているが AKS より柔軟性に劣る。 | ●●● Cloud SQL、Memorystore、LB mTLS、Cloud Armor、PSC、VPC-SC は GA。Cloud Run は運用負荷が非常に小さい。 |
| **運用負荷（ゲートウェイの運用）** | ●●○ ECS Fargate は運用負荷が小さい。ネットワークの構成要素が多い（TGW + 検査用 VPC + サービスごとのエンドポイント）。 | ●●○ ACA は運用負荷が小さい。ハブのファイアウォールとプライベート DNS ゾーンの乱立には、ランディング ゾーンの規律が必要。 | ●●● Cloud Run、全 Google API を 1 つでカバーする PSC エンドポイント、VPC-SC により、管理対象が最も小さい。 |
| **ネイティブ モデルへのアクセス** | **Bedrock**: Anthropic Claude（AWS のファーストパーティ）、Amazon Nova、Llama、Mistral、DeepSeek、Cohere など。クロスリージョン推論。Guardrails。 | **Azure OpenAI / Foundry**: OpenAI の GPT と推論モデル（Azure のファーストパーティ）と、Foundry カタログのモデル（パートナー、オープン）。PTU。組み込みのコンテンツ フィルター。 | **Vertex AI**: Gemini（ファーストパーティ）と、Model Garden のパートナー モデル（Claude、Llama、Mistral など）。プロビジョニング スループット。 |
| **Entra ID との ID 統合** | ●●○ LiteLLM ↔ Entra はクラウドに依存しない。AWS 運用者の SSO は IAM Identity Center + SCIM。AWS Verified Access に Intune をネイティブに扱うデバイス信頼プロバイダーがないため、mTLS を使う。 | ●●● 同一テナント: CA、Intune、マネージド ID、Entra Private Access、Sentinel。統合の継ぎ目が最も少ない。 | ●●○ LiteLLM ↔ Entra はクラウドに依存しない。運用者は Workforce Identity Federation。Intune シグナルは Chrome Enterprise Premium（追加ライセンス）経由で利用可能。 |
| **ネットワークの複雑さ** | ●●○ サービスごとのインターフェイス エンドポイントと Resolver ルールが多い。TGW 経由の Network Firewall ルーティング。リポジトリに既製の Private VPC シナリオがある。 | ●●○ リンクとフォワードが必要なプライベート DNS ゾーンが約 10 個。Firewall の UDR。ACA サブネットのサイジング。ExpressRoute プライベート ピアリングは Microsoft 中心のネットワーク チームにはなじみがある。 | ●●● 1 つの PSC エンドポイントで全 Google API をカバー。共有 VPC。VPC-SC はポリシー設計の手間が増えるが、ルーティングは複雑にならない。 |
| **コスト要因（インフラ。モデル利用料が支配的）** | Fargate の vCPU/GB 時間、Aurora のインスタンス + I/O、ElastiCache のノード、**インターフェイス エンドポイント（AZ 時間 + GB 単位）**、**Network Firewall エンドポイント + NAT の GB 単位**、DX ポート時間とデータ送信、CloudWatch の取り込み。リポジトリのサンプルは堅牢化前で約 USD 380/月。Network Firewall とエンドポイントでそれ以上が加算されうる。 | ACA の専用ワークロード プロファイル、PostgreSQL HA（コンピューティング 2 倍）、Managed Redis、**App Gateway WAF v2 の容量ユニット**、**Azure Firewall Premium**（固定の時間課金 + GB 単位）、プライベート エンドポイント（時間 + GB 単位）、ER 回線 + ゲートウェイ、**Log Analytics の取り込み**（最大の想定外コストになりやすい）。 | Cloud Run（インスタンス ベース、最小インスタンスあり）、Cloud SQL Enterprise Plus HA、Memorystore、LB の転送ルール + データ、Cloud Armor（ポリシー + リクエスト、Enterprise は任意）、Cloud NGFW、Interconnect の VLAN アタッチメント + 送信、Cloud Logging の取り込み。Apigee を追加する場合は大きな固定費になる。 |
| **モデルのコスト配賦の精度** | ●●○ 部門/チームごとのタグ付きアプリケーション推論プロファイル（粒度が粗く、管理するプロファイルが増える）+ LiteLLM | ●○○ 請求はリソース単位のみ。チーム単位には LiteLLM（または APIM のメトリクス）か BU ごとのリソースが必要。 | ●●● Vertex のリクエスト単位の請求ラベル（従量課金）+ LiteLLM |
| **ベンダー ロックイン** | 低〜中: LiteLLM、Postgres、Redis は可搬性がある。ロックインは Bedrock 固有の機能（Guardrails、推論プロファイル）とミドルウェアから生じる。 | 低〜中: 可搬性は同様。Azure OpenAI のデプロイと PTU の概念、Content Safety は Azure 固有。 | 低〜中: 可搬性は同様。VPC-SC、Model Armor、ラベルは GCP 固有。 |
| **既存環境との適合性** | DX は設置済み。Claude on Bedrock が主要なモデル ファミリーなら有力。 | ER、M365、Entra、Intune、Sentinel は導入済み。ID、セキュリティ運用、調達（MACC）の面で最も適合性が高い。 | Interconnect は設置済み。Gemini が主要モデル、または BigQuery が財務分析基盤なら最も有力。 |

## 7.2 推奨

1. **クラウド間で可搬性のある LiteLLM ゲートウェイ プラットフォームの設計を 1 つ作る。** コンテナー + PostgreSQL + Redis、Entra OIDC/JWT、[§2.4](01-common-design.md#24-litellm-テナント-モデル全クラウド共通) の LiteLLM テナント モデル、クラウドごとの IaC モジュールで構成する。**プライマリのホスティング クラウドを 1 つ** 選び、他クラウドのモデルには送信経由で LiteLLM のバックエンドとしてアクセスする。将来的にはクラウド間の閉域接続も検討できる。
2. **プライマリのホスティング クラウドは主要なモデル ファミリーで選び、決め手がなければ ID で決める。**
   - 利用の大半が **Anthropic Claude** になる場合、または事業部門が **本番までの最短経路** を求める場合は **AWS** を選ぶ。公式ガイダンスが LiteLLM *そのもの* なので、設計とセキュリティ レビューの期間を短縮できる。[§4](03-aws.md#4-ステップ-2a-aws-設計) の変更を適用する。OIDC IdP を Okta ではなく Entra にする、CloudFront 既定構成の代わりにリージョン WAF + ALB mTLS を使う、Network Firewall による送信制御、Bedrock VPC エンドポイント、推論プロファイルのタグ付けである。
   - 利用の大半が **OpenAI モデル** で、セキュリティ チームが **単一の Microsoft コントロール プレーン**（CA + Intune + Private Access + Sentinel）を重視する場合は **Azure** を選ぶ。出発点となる LiteLLM のリファレンスがないため設計作業はやや増えるが、ID とデバイス信頼の統合作業は最も少ない。
   - Gemini が主要モデルの場合、または財務部門が BigQuery で **請求ラベル レベルのチャージバック** を求める場合は **GCP** を選ぶ。運用負荷も最も小さい。
3. **既定では API 管理製品を重ねない。** LiteLLM の前段に APIM や Apigee を置くと、トークン計上が重複する。LiteLLM 以外の利用者との PTU 共有プールや、全社 API ポータルの義務化など、特定の要件がある場合にのみ追加する。
4. **LiteLLM の利用額を配賦キー、クラウド請求をコストの正として扱う。** どのクラウドでも月次の突合ジョブを実行する。

## 7.3 主なトレードオフ

| トレードオフ | 影響 |
|---|---|
| 単一のポリシー点としての LiteLLM か、クラウド ネイティブのゲートウェイ（APIM/Apigee）か | LiteLLM なら、すべてのプロバイダーにまたがる単一のユーザー/チーム/予算モデルが得られる。オープンソースだが、エンタープライズ機能にはライセンスが必要。パッチ適用とスケーリングは当社チームの責任となるため、バージョンを固定し、LiteLLM のセキュリティ アドバイザリを追跡する。 |
| 仮想キー（シンプル）か Entra JWT（ゼロトラスト）か | 仮想キーは開発者にとって扱いやすいが、使用時にデバイス チェックのないベアラー シークレットである。JWT はトークン有効期間ごとに条件付きアクセスを再適用するが、クライアントに MSAL が必要。JWT を既定とし、仮想キーは有効期限付きで、mTLS または閉域経路の背後でのみ発行する。 |
| 単一クラウドでのホスティングか、マルチクラウドのアクティブ/アクティブか | マルチクラウドのホスティングはプロバイダー障害への耐性を得られるが、運用が 2 倍になり、クラウド間で予算の整合性を保つ必要がある（Postgres の共有は現実的でない）。単一クラウドでの HA と、第 2 クラウドでのコールド スタンバイ IaC を推奨する。 |
| 内容のログ記録かプライバシーか | プロンプトを記録すると不正利用の調査や品質改善に役立つが、機密データの保存先が生まれる。既定はメタデータのみ [A-10] とし、チーム単位のオプトインで、専用のキーと保持期間を設ける。 |
| マネージドのサーバーレス コンテナーか Kubernetes か | Fargate、ACA、Cloud Run は運用を最小化する。サイドカー（Presidio）、カスタム受信、セルフホスト モデルには Kubernetes が必要になる。 |


---

[← 前へ: ステップ 2c: GCP](05-gcp.md) · [目次](../../README.md) · [次へ →: ステップ 4: 閉域接続](07-private-connectivity.md)
