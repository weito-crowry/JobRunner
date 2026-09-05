# JobRunner UI mockups

このディレクトリは、JobRunner の Web UI 設計検討用モックを保管する。

## 位置づけ

- `mockups/` 配下の HTML は、画面構成・情報設計・操作感を共有するための参考モックである。
- モックは実装仕様の Source of Truth ではない。
- 実装時は `docs/design.md`、`docs/detailed-design/` 配下の正式な設計、および確定した API / data contract を優先する。
- モック内の数値、Job、Workflow、Runner、ログなどは表示確認用のサンプルデータである。

## 現在の方針

JobRunner の UI は、n8n のようなビジュアルオートメーション製品よりも、GitHub Actions に近い運用コンソールを基本とする。

特に Job 詳細は `Workflow -> Run -> Job -> Step` の階層を前提とし、各 Step について次の情報を確認できる構成を目標とする。

- Step 名
- Status / Duration
- Description（その Step が何をしているか）
- Handler / Runtime
- Input / Produces
- Step 単位の Logs

Job 全体の Input / Output / Context / Runtime 情報は Job summary として別途表示する。

## Mockups

- [`mockups/jobrunner-ui-mock.html`](mockups/jobrunner-ui-mock.html) — Dashboard、Workflow、Job、Runner Pool、Settings を含む単一 HTML モック。
