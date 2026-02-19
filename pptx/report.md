# 2. 本番環境／開発環境／PoC 環境の 3 環境の準備

> **出典:** `抜粋.pptx` スライド 1 — VDC (Virtual Data Center) 環境における 3 環境構成のアーキテクチャ概要図

---

## 全体構成概要

本スライドは、エンタープライズ向け VDC（Virtual Data Center）環境において、**本番環境**・**開発環境**・**PoC 環境** の 3 環境をどのように分離・管理するかを示すアーキテクチャ図です。

### 環境ゾーン区分

スライド上部には 3 つのゾーンが左右に配置されています。

```mermaid
graph LR
    subgraph zone1["オンプレ環境"]
        onprem["オンプレミス<br/>データセンター"]
    end
    subgraph zone2["VDC 環境"]
        vdc["仮想データセンター<br/>（本番・開発）"]
    end
    subgraph zone3["PoC / Coding 環境"]
        poc["PoC 環境<br/>（社外開発用）"]
    end
    zone1 --- zone2 --- zone3
```

---

## Azure AD テナント構成

各環境は異なる Azure AD テナントに所属し、アイデンティティが分離されています。

```mermaid
graph TB
    subgraph AAD["Azure AD テナント構成"]
        aad1["O365 用 AAD"]
        aad2["本番系 VDC 管理用 AAD"]
        aad3["開発系 VDC 管理用 AAD"]
        aad4["VDC 管理用 Azure AD<br/>（PoC 側）"]
    end

    aad1 -->|"エンドユーザ<br/>認証"| prod["本番環境"]
    aad2 -->|"基盤担当<br/>管理"| prod
    aad3 -->|"開発チーム<br/>管理"| dev["開発環境"]
    aad4 -->|"社外チーム<br/>管理"| pocenv["PoC 環境"]
```

---

## ネットワーク構成図

本番ネットワークと開発ネットワークは「**開発／運用分離境界**」で分離され、それぞれに Hub-Spoke トポロジが展開されています。

```mermaid
graph TB
    subgraph boundary_prod["本番ネットワーク"]
        dc1["DC<br/>（データセンター）"]
        erpp1["ER PP<br/>（ExpressRoute Private Peering）"]
        hub1["Hub<br/>（仮想ネットワーク）"]
        spoke1a["Spoke<br/>（本番ワークロード）"]
        spoke1b["Spoke<br/>（追加 Spoke）"]
        prod_env["本番環境<br/>🖥️ VM / PaaS リソース"]

        dc1 --- erpp1
        erpp1 --- hub1
        hub1 --- spoke1a
        hub1 --- spoke1b
        spoke1a --- prod_env
    end

    subgraph separation["🔴 開発／運用分離境界"]
        sep_text["開発と運用の<br/>アクセス制御を分離"]
    end

    subgraph boundary_dev["開発ネットワーク"]
        dc2["DC<br/>（データセンター）"]
        erpp2["ER PP<br/>（ExpressRoute Private Peering）"]
        hub2["Hub<br/>（仮想ネットワーク）"]
        spoke2a["Spoke<br/>（開発ワークロード）"]
        spoke2b["Spoke<br/>（追加 Spoke）"]
        dev_env["開発環境<br/>🖥️ VM / PaaS リソース"]

        dc2 --- erpp2
        erpp2 --- hub2
        hub2 --- spoke2a
        hub2 --- spoke2b
        spoke2a --- dev_env
    end

    boundary_prod ~~~ separation
    separation ~~~ boundary_dev
```

---

## 環境構築・デプロイフロー

各環境へのデプロイは IaC（Infrastructure as Code）スクリプトによって行われ、Immutable Infrastructure の原則が適用されています。

```mermaid
flowchart LR
    subgraph build_prod["本番環境 構築フロー"]
        iac_prod["IaC スクリプト<br/>📜 PowerShell"]
        review_prod["レビューして<br/>持ち込み"]
        deploy_prod["構築 ➡️<br/>本番環境"]
        immutable_prod["💚 Immutable"]
    end

    subgraph build_dev["開発環境 構築フロー"]
        iac_dev["IaC スクリプト<br/>📜 PowerShell"]
        review_dev["レビューして<br/>持ち込み"]
        deploy_dev["構築 ➡️<br/>開発環境"]
        immutable_dev["💚 Immutable"]
    end

    subgraph build_poc["PoC 環境 構築フロー"]
        iac_poc["IaC スクリプト<br/>📜 PowerShell"]
        deploy_poc["構築 ➡️<br/>PoC 環境"]
        free_poc["💚 自由にいじれる<br/>閉域化不要<br/>データ持ち込み不可"]
    end

    iac_prod --> review_prod --> deploy_prod
    iac_dev --> review_dev --> deploy_dev
    iac_poc --> deploy_poc
```

### 環境ごとのリソース配置フロー

```mermaid
flowchart TD
    subgraph prod_deploy["本番環境への配置"]
        p_worker["👷 社内開発者"]
        p_ps["📜 PowerShell"]
        p_arm["📦 ARM テンプレート"]
        p_deploy["配置 ➡️"]
        p_target["本番 Spoke"]

        p_worker --> p_ps --> p_arm --> p_deploy --> p_target
    end

    subgraph dev_deploy["開発環境への配置"]
        d_worker["👷 社内開発者"]
        d_ps["📜 PowerShell"]
        d_arm["📦 ARM テンプレート"]
        d_deploy1["配置 ➡️"]
        d_deploy2["配置 ➡️"]
        d_target1["開発 Spoke 1"]
        d_target2["開発 Spoke 2"]

        d_worker --> d_ps --> d_arm
        d_arm --> d_deploy1 --> d_target1
        d_arm --> d_deploy2 --> d_target2
    end

    subgraph poc_deploy["PoC 環境への配置"]
        poc_worker["👷 社外開発者"]
        poc_ps["📜 PowerShell"]
        poc_target["PoC Spoke"]

        poc_worker --> poc_ps --> poc_target
    end
```

---

## CI/CD パイプライン構成

社外開発チームは Azure DevOps を活用して開発を行い、社内の Azure DevOps Server とは「**社内／社外分離境界**」で分離されています。

```mermaid
flowchart LR
    subgraph external["社外 Azure DevOps"]
        repos["📁 ソースコード<br/>レポジトリ（Repos）"]
        ci["⚙️ 自動ビルド（CI）"]
        artifact["📦 成果物<br/>バイナリ（Artifact）"]
        pipeline["🚀 リリースシステム<br/>（Pipeline）"]

        repos --> ci
        ci --> artifact
        artifact --> pipeline
    end

    subgraph internal["社内 Azure DevOps Server"]
        int_devops["Azure DevOps Server"]
    end

    subgraph sep_boundary["🔴 社内／社外 分離境界"]
        sep["アクセス制御"]
    end

    ext_dev["👤 社外アプリ開発チーム<br/>💻 PC"] --> repos
    ext_infra["👤 社外インフラ開発チーム<br/>💻 PC"] --> repos

    pipeline -->|"レビューして持ち込み"| dev_env["開発環境"]
    pipeline -->|"レビューして持ち込み"| prod_env["本番環境"]

    int_dev["👤 社内アプリ開発チーム<br/>💻 PC"] --> internal
    internal -->|"配置"| dev_env
```

---

## アプリ開発パターン

```mermaid
flowchart TB
    subgraph pattern1["アプリ開発パターン① — 社外 SIer が社外で開発"]
        ext_sier["👤 社外 SIer"]
        ext_devops["Azure DevOps<br/>（クラウド）"]
        ext_poc["PoC 環境<br/>（自由にいじれる）"]
        ext_review["レビューして持ち込み"]
        ext_target["開発・本番環境"]

        ext_sier --> ext_devops
        ext_sier --> ext_poc
        ext_devops --> ext_review --> ext_target
    end

    subgraph pattern2["アプリ開発パターン② — 常駐型・閉域での開発"]
        int_dev["👤 社内アプリ開発チーム"]
        int_devops["Azure DevOps Server<br/>（オンプレ）"]
        int_target["開発環境<br/>（閉域性維持が必要）"]

        int_dev --> int_devops --> int_target
    end
```

---

## セキュリティ・監視構成

```mermaid
flowchart TB
    subgraph security["セキュリティチェック体制"]
        subgraph infra_sec["インフラ（基盤）のセキュリティチェック 🔴"]
            cspm["Azure Security Center<br/>/ CSPM"]
            audit["インフラ構成を監査"]
            cspm --> audit
        end

        subgraph app_sec["アプリ（コード）のセキュリティチェック 🔵"]
            code_review["コードセキュリティ<br/>レビュー"]
        end

        subgraph config_check["セキュリティ構成チェック"]
            check_tool["セキュリティ構成<br/>チェックツール<br/>（Azure Security Center / CSPM）"]
            maintain["セキュリティ構成が<br/>維持されていることをチェック"]
            check_tool --> maintain
        end
    end

    subgraph monitoring["24h 統合運用監視"]
        mon_sys["統合運用監視システム"]
        alert["アラート通知"]
        mon_sys --> alert
    end

    infra_sec -->|"構成監査"| prod["本番環境"]
    infra_sec -->|"構成監査"| dev["開発環境"]
    monitoring -->|"24時間監視"| prod
    monitoring -->|"24時間監視"| dev

    approver["👤 情報セキュリティ担当<br/>（承認者）"] --> security
```

---

## 利用者・ロール一覧

スライド右上に記載された主要ロール：

```mermaid
graph LR
    subgraph roles["より精緻化した作業ワークフローの例"]
        enduser["👥 エンドユーザ"]
        vdc_admin["🔧 VDC 基盤担当<br/>（システム ID）"]
        int_dev["👷 社内開発者"]
        ext_dev["👷 社外開発者"]
        sec_admin["🛡️ 情報セキュリティ担当<br/>（承認者）"]
    end

    enduser -->|"利用"| prod["本番環境"]
    vdc_admin -->|"管理"| all["全環境"]
    int_dev -->|"開発"| dev["開発環境"]
    ext_dev -->|"開発"| poc["PoC 環境"]
    sec_admin -->|"承認・監査"| all
```

---

## 環境比較表

スライド内のテーブルから抽出した環境比較情報：

| 環境分類 | 本番系 | 開発系 | PoC 環境 |
|:---|:---|:---|:---|
| **所属 AAD テナント** | 本番系 VDC 管理用 AAD | 開発系 VDC 管理用 AAD | PoC 管理用 AAD |
| **環境利用者** | エンドユーザ | 社内アプリ開発チーム | 社外インフラ・アプリ開発チーム |
| **環境管理者** | VDC 基盤担当 | VDC 基盤担当 | VDC 基盤担当 |
| **利用目的** | 本番運用 | 社内での開発 | 社外での開発 |
| **本番データ** | 利用可 | 利用可 | 利用不可（ダミー利用） |
| **監査対象** | Yes | Yes | No |

---

## 全体アーキテクチャ（統合図）

```mermaid
graph TB
    subgraph title["2. 本番環境／開発環境／PoC 環境の 3 環境の準備"]
        direction TB

        subgraph aad_layer["Azure AD テナント層"]
            o365_aad["O365 用 AAD"]
            prod_aad["本番系 VDC<br/>管理用 AAD"]
            dev_aad["開発系 VDC<br/>管理用 AAD"]
            poc_aad["VDC 管理用<br/>Azure AD（PoC）"]
        end

        subgraph onprem_zone["オンプレ環境"]
            subgraph prod_nw["本番ネットワーク"]
                dc1["DC"] --> erpp1["ER PP"] --> hub1["Hub"]
                hub1 --> spoke1a["Spoke"]
                hub1 --> spoke1b["Spoke"]
            end

            subgraph dev_nw["開発ネットワーク"]
                dc2["DC"] --> erpp2["ER PP"] --> hub2["Hub"]
                hub2 --> spoke2a["Spoke"]
                hub2 --> spoke2b["Spoke"]
            end
        end

        subgraph vdc_zone["VDC 環境"]
            prod_env["本番環境<br/>🖥️ VM群"]
            dev_env["開発環境<br/>🖥️ VM群"]
        end

        subgraph poc_zone["PoC / Coding 環境"]
            poc_env["PoC 環境<br/>🖥️ VM群"]
            devops_cloud["Azure DevOps<br/>（Repos/CI/Artifact/Pipeline）"]
        end

        subgraph monitoring_zone["24h 統合運用監視"]
            mon["統合運用監視システム"]
            cspm["Azure Security Center<br/>/ CSPM"]
        end

        spoke1a --> prod_env
        spoke2a --> dev_env

        prod_aad --> prod_env
        dev_aad --> dev_env
        poc_aad --> poc_env

        devops_cloud -->|"レビューして持ち込み"| prod_env
        devops_cloud -->|"レビューして持ち込み"| dev_env

        mon -->|"アラート通知"| prod_env
        mon -->|"アラート通知"| dev_env
        cspm -->|"セキュリティ構成チェック"| prod_env
        cspm -->|"セキュリティ構成チェック"| dev_env
    end

    style prod_nw fill:#e6f3ff,stroke:#0066cc
    style dev_nw fill:#fff3e6,stroke:#cc6600
    style poc_zone fill:#e6ffe6,stroke:#009900
    style monitoring_zone fill:#ffe6eb,stroke:#cc0000
```

---

## 注釈（吹き出し情報）

スライド上の緑色の吹き出し（💚）は以下のポイントを強調しています：

| 対象 | 注釈内容 |
|:---|:---|
| 本番環境 | **Immutable** — 不変インフラストラクチャとして管理 |
| 開発環境 | **Immutable** — 同様に不変インフラストラクチャ |
| 本番環境への持ち込み | **レビューして持ち込み** — 変更は必ずレビューを経由 |
| 開発環境への持ち込み | **レビューして持ち込み** — 同上 |
| PoC 環境 | **自由にいじれる / 閉域化不要 / データ持ち込み不可** |
| 開発環境 | **閉域性維持が必要** |
| インフラ構築 | **IaC スクリプト** による自動構築 |
| Azure DevOps → 本番 | **レビューして持ち込み** |
| セキュリティ構成 | **セキュリティ構成が維持されていることをチェック** |
| セキュリティ監査 | **インフラ構成を監査** |

---

## 分離境界

スライド上のピンク色（🔴）で示された分離境界：

1. **開発／運用分離境界** — 本番ネットワークと開発ネットワークの間に位置し、運用環境と開発環境のアクセスを分離
2. **社内／社外分離境界** — 社内開発チーム（Azure DevOps Server 利用）と社外開発チーム（Azure DevOps クラウド利用）の間に位置

---

## セキュリティチェック体制

上部に2つのセキュリティチェック領域が配置されています：

| カテゴリ | 色 | 内容 |
|:---|:---|:---|
| **インフラ（基盤）のセキュリティチェック** | 🔴 ピンク | Azure Security Center / CSPM による構成監査 |
| **アプリ（コード）のセキュリティチェック** | 🔵 水色 | コードレベルのセキュリティ検証 |
