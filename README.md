# LLM Gateway Architecture Comparison (AWS / Azure / GCP)

Architecture documentation for an internal **LiteLLM**-based LLM gateway platform covering user and team management, cost allocation, and token controls. It is designed for our existing environment: Microsoft Entra ID for identity, Okta Verify for MFA, Intune for device trust, and the existing ExpressRoute / Direct Connect / Cloud Interconnect links.

The English version is authoritative. The Japanese translation (`docs/jp/`) uses the same filenames as the English version (`docs/en/`).

LiteLLM を基盤とする社内 LLM ゲートウェイのアーキテクチャ ドキュメント。英語版を正とし、日本語訳（`docs/jp/`）は英語版（`docs/en/`）と同じファイル名で対応している。

| # | English | 日本語 |
|---|---|---|
| 0–1 | [Overview & assumptions](docs/en/00-overview.md) | [概要と前提条件](docs/jp/00-overview.md) |
| 2 | [Common design: identity, MFA, device trust, LiteLLM tenancy](docs/en/01-common-design.md) | [共通設計: ID、MFA、デバイス信頼、LiteLLM テナント モデル](docs/jp/01-common-design.md) |
| 3 | [Step 1: Official reference architectures](docs/en/02-reference-architectures.md) | [ステップ 1: 公式リファレンス アーキテクチャ](docs/jp/02-reference-architectures.md) |
| 4 | [Step 2a: AWS design](docs/en/03-aws.md) | [ステップ 2a: AWS 設計](docs/jp/03-aws.md) |
| 5 | [Step 2b: Azure design](docs/en/04-azure.md) | [ステップ 2b: Azure 設計](docs/jp/04-azure.md) |
| 6 | [Step 2c: GCP design](docs/en/05-gcp.md) | [ステップ 2c: GCP 設計](docs/jp/05-gcp.md) |
| 7 | [Step 3: Comparison and recommendation](docs/en/06-comparison.md) | [ステップ 3: 比較と推奨](docs/jp/06-comparison.md) |
| 8 | [Step 4: Private connectivity, mTLS, zero-trust posture](docs/en/07-private-connectivity.md) | [ステップ 4: 閉域接続、mTLS、ゼロトラスト](docs/jp/07-private-connectivity.md) |
| 9 | [Reference link index](docs/en/08-references.md) | [参照リンク一覧](docs/jp/08-references.md) |

Section numbers (§N.M) are kept across files, and cross-references link to the relevant file and heading. Diagrams are Mermaid and render on GitHub.

When you change an English file, update the Japanese file with the same name in the same PR.
