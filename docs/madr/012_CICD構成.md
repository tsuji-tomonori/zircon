# Architecture Decision Record: CI/CD パイプラインの設計

- **日付**: 2025-03-12
- **ステータス**: 提案中 (Proposed)

## 1. 背景 (Context)

本プロジェクト「zircon」は、以下の要件を満たすタスク管理システムである。

- 無限に子タスクを生成できる TODO アプリ
- AIアシストによる子タスク分解機能
- 表形式・カンバン・ツリー・ガントチャートなど多彩なタスク表示
- コメント／メンション／通知機能、ステータス管理の柔軟な設定
- **ローカル環境**では Docker Compose による一括起動
- **本番環境**は AWS を利用（できる限りコストを抑えるサーバレス構成）
- リポジトリは GitHub（個人アカウント）で管理

既存の各 ADR にて、フロントエンド/バックエンド/DB やアプリケーションのアーキテクチャ選定が行われている。
本 ADR では「**CI/CD パイプライン**」をどのように設計・運用するか、特に **GitHub Actions** を用いた自動化の方針を決定する。

## 2. 課題 (Problem)

1. **開発からデプロイまでの工程を自動化したい**
   - コードが更新されるたびにビルド・テスト・静的解析を手動で行うのは非効率。
   - 一貫したパイプラインを構築し、チームで共有することでバグ検出やリリース作業を迅速化したい。

2. **ローカルと本番の差異を最小化したい**
   - ローカルは Docker Compose、本番は AWS (S3+CloudFront, Lambda, Aurora Serverlessなど)。
   - ビルドの再現性やテスト手順をパイプライン上でも確実に実行するしくみが必要。

3. **コストを抑えつつ複数環境運用をしたい**
   - ステージング環境や本番環境へのデプロイをすべて手動で行うとミスの可能性が高い。
   - ただし常時起動の大規模環境を増やすとコストがかさむため、必要に応じてステージング環境を最小限のスケールで運用する、あるいは踏み台となるテストをサーバレスに合わせて構築したい。

4. **IaC と連携してインフラも自動更新したい**
   - 既存の ADR で決定したように、インフラは AWS CDK (TypeScript) による IaC 化を予定。
   - アプリケーション（フロント／バックエンド）の更新だけでなく、インフラ設定変更も Pull Request / Merge 時に自動反映できるようにしたい。


## 3. 決定 (Decision)

### **GitHub Actions を用いた CI/CD パイプラインを構築し、以下のステップを自動化する。**

1. **ビルド・テスト**
   - プルリクエスト（PR）または push イベント時に、フロントエンド（React+Vite）・バックエンド（Hono+TypeScript）のビルドとテストを実行する。
   - それぞれ Docker ベースで行うか、pnpm / Node.js 環境をセットアップして実行するかはプロジェクトの規模に応じて選択。
   - Lint, TypeCheck, UnitTest などの静的解析・自動テストを含め、**PR の段階でバグ検出**を目指す。

2. **ステージング環境へのデプロイ**
   - メインブランチにマージされた段階（もしくは特定タグ発行）で、ステージング用の AWS リソースにデプロイを行う。
   - **フロントエンド**: S3 バケット（ステージング用） + CloudFront ディストリビューションにビルド成果物をアップロード
   - **バックエンド**: Lambda (ステージング用の関数) および Aurora Serverless (必要最小限のACU) にデプロイ
   - **IaC(CDK)** でステージング用スタックを定義し、`cdk deploy --context stage=staging` などで分ける方法を想定。

3. **承認フローを経た本番デプロイ**
   - ステージングでの動作確認が完了し、リリース担当者が承認したら GitHub Actions の **手動承認ジョブ**（workflow_dispatch など）を経由して本番デプロイを実行。
   - **フロントエンド**: 本番用 S3 バケット+CloudFront への同期とキャッシュ無効化(Invalidation)
   - **バックエンド**: 本番 Lambda 関数へのデプロイ、Aurora Serverless v2 へのマイグレーション適用(Prisma Migrate)
   - **IaC**: 同様に CDK で本番スタックを更新。

4. **コストを抑える工夫**
   - ステージング環境は最小スケール(例: Aurora Serverless の最小 ACU を小さく設定)で必要最低限の性能とし、一定時間アクセスがなければコストを大幅に削減できる。
   - Lambda もリクエスト単位課金なので、負荷が低いうちはコストを抑えられる。

## 4. 選択肢 (Alternatives)

1. **Jenkins や self-hosted CI**
   - 自前で CI サーバを構築・運用する方法。大規模・高度なカスタマイズが必要な場合に有効だが、インフラ管理コストが上昇。
   - 個人アカウントによる GitHub 運用であれば、GitHub Actions を使う方がセットアップが容易でコストが抑えられる。

2. **AWS CodePipeline + CodeBuild**
   - AWS 公式の CI/CD サービス。IAM 設定やリソース連携は CDK と相性が良いが、GitHub Actions との連携、プルリク段階でのテストなどを柔軟に行うには追加の設定が必要。
   - GitHub Actions はリポジトリ連携がシンプルで、外部サービスや Slack 通知等の統合も容易。

3. **CircleCI / GitLab CI**
   - 他のホスティングサービスや CI サービスもある。
   - 既に GitHub を使っており、GitHub Actions が無料枠も比較的充実しているため、最適解とは言えない（個人アカウントであれば Actions が使いやすい）。

4. **手動デプロイ**
   - 小規模ならローカルでビルドして手動 S3 アップロード、Lambda へ手動アップロードもできるが、作業ミスや属人化のリスクが高い。
   - Pull Request ベースのチーム開発には向かない。

## 5. 根拠 (Rationale)

- **GitHub Actions** は GitHub リポジトリとの親和性が高く、プルリク段階での自動テストやデプロイが容易。
- **サーバレス構成** (Lambda, S3+CloudFront, Aurora Serverless) はリソースを動的に管理するため、CDK と組み合わせた自動デプロイが望ましい。
- **個人アカウントかつ小〜中規模** の想定では、GitHub Actions の無料枠や従量課金で十分賄える可能性が高く、コストを抑えやすい。

## 6. 結果 (Consequences)

- **ポジティブな影響**
  1. コードの更新 → テスト → ステージングデプロイ → 本番リリースという一連の流れを**自動化**でき、リリースサイクルが短縮。
  2. **Pull Request 単位**で静的解析・ユニットテストが走り、エラーを早期に検知可能。
  3. **CDK** による IaC と組み合わせることで、インフラ変更も含めた**一貫したレビュー・デプロイ**を実現し、ヒューマンエラーを削減。

- **ネガティブな影響**
  1. GitHub Actions や IaC(CDK) の学習コスト。小規模チームが短期でスパイクする場合、構築に時間が必要。
  2. Actions の無料枠を超えるほどビルドやテストが増えると、追加費用が発生する可能性がある。
  3. ステージング環境を長時間維持すると Aurora Serverless 最小 ACU 分の課金など、**開発時でも一定コスト**がかかる点に留意。

## 7. 実装方針 (Implementation Notes)

1. **リポジトリ構成例**（モノレポ）
   ```
   zircon/
   ├── .github/
   │   └── workflows/
   │       ├── ci.yaml       # CI用(プルリク時のビルド/テスト/リンティング)
   │       └── deploy.yaml   # CDパイプライン(ステージング/本番デプロイ)
   ├── zircon-frontend/      # フロントエンド (React + Vite)
   ├── zircon-backend/       # バックエンド (Hono + TypeScript)
   ├── packages/
   │   └── shared/           # 共通型定義等
   ├── infra/                # AWS CDK (IaC)
   └── ...
   ```

2. **主要ワークフロー**

   - **ci.yaml** (Pull Request/Push イベント)
     1. **チェックアウト & セットアップ Node**
        ```yaml
        - uses: actions/checkout@v3
        - uses: actions/setup-node@v3
          with:
            node-version: 18
        ```
     2. **依存インストール (pnpm など)**
        ```yaml
        - name: Install dependencies
          run: pnpm install
        ```
     3. **ビルド・テスト**
        - Lint / TypeCheck / Unit Test をフロント・バック両方に対して実行
        - 結果を GitHub 上のチェックとしてレポート
     4. **(オプション) Docker ベースのテスト**
        - 必要に応じて Docker Compose でバックエンド + DB を立ち上げ、API テストなどを実行

   - **deploy.yaml** (メインブランチへのマージ or 手動トリガー)
     1. **ci.yaml と同様のビルド・テスト** (安全のため再度)
     2. **CDK デプロイ: ステージング**
        - `cd infra && pnpm run cdk deploy --require-approval never --context stage=staging`
     3. **手動承認ステップ**
        - Approver が GitHub Actions の UI で承認するまで待機
     4. **CDK デプロイ: 本番**
        - `cd infra && pnpm run cdk deploy --require-approval never --context stage=production`
     5. **フロントエンド本番バケットへのアップロード** (S3 sync) or CDK タスク内で自動反映

3. **Secrets / Credentials**
   - GitHub Actions で使う AWS 資格情報（IAM User / OpenID Connect 連携など）は、**GitHub Secrets** に保存し、`aws-actions/configure-aws-credentials` で設定。
   - Aurora の接続情報・各種 API キーは AWS Systems Manager Parameter Store や Secrets Manager と連携し、CDK スタックで参照。

4. **コスト最適化策**
   - Pull Request 単位のテストではできるだけ**軽量なコンテナ or Node 環境のみ**を使い、Integration Test は必要最小限に抑える。
   - ステージング環境は**最小規模の Aurora Serverless** 設定 & Lambda でアイドル時課金を減らす。
   - GitHub Actions の使用量が増えた場合、**Self-Hosted Runner** or **割引プラン**を検討。

5. **運用監視**
   - デプロイ結果を Slack 通知するなど、CI/CD 成果をリアルタイムに把握。
   - デプロイ後の CloudWatch Logs, メトリクスを定期的に確認し、エラー増大時にはロールバック (前バージョンの Lambda) を検討。

## 8. 今後の展望 (Future Considerations)

1. **Blue/Green デプロイ / Canary リリース**
   - AWS Lambda の別バージョンを並行稼働させ、正常性を確認後にトラフィックを切り替える仕組みなど、追加のデプロイ戦略を検討できる。
2. **テスト自動化の拡充**
   - E2E テスト（Cypress, Playwright など）をワークフローに組み込み、UI 面まで自動検証する。
3. **マルチアカウント運用**
   - 大規模化に伴い、開発/検証/本番アカウントを分割し、IaC (CDK) で複数アカウントにデプロイするシナリオもあり得る。
4. **セキュリティ・脆弱性スキャン**
   - CI/CD パイプラインに SAST/DAST、Docker イメージスキャン などを組み込むことで、リリース前にセキュリティ課題を早期発見可能。
