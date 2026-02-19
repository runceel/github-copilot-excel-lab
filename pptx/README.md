# pptx

PowerPoint ファイルの解析・変換サンプル。

## ファイル一覧

| ファイル | 説明 |
|:---|:---|
| `抜粋.pptx` | 解析対象の PowerPoint ファイル（VDC 3 環境構成のアーキテクチャ図） |
| `report.md` | `抜粋.pptx` を OpenXML SDK で解析して生成した Markdown レポート（Mermaid 図含む） |
| `chatlog.md` | 解析時の作業ログ |

## report.md について

GitHub Copilot が `抜粋.pptx` を OpenXML SDK で解析し、スライド内の図形・接続・テーブル・位置関係から意味を推測して生成した Markdown です。

含まれる内容：

- **環境構成概要** — 本番 / 開発 / PoC の 3 環境ゾーン
- **Azure AD テナント構成図**（Mermaid）
- **ネットワーク構成図**（Hub-Spoke トポロジ、Mermaid）
- **構築・デプロイフロー**（IaC / Immutable Infrastructure、Mermaid）
- **CI/CD パイプライン構成**（Azure DevOps、Mermaid）
- **セキュリティ・監視構成**（Azure Security Center / CSPM、Mermaid）
- **環境比較表**（元スライドのテーブルをそのまま抽出）
