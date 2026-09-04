# JobRunner

軽量・汎用の Persistent Workflow / Job Runtime。

既存アプリケーションへ組み込み、Workflow / Job / Runner / Retry / Resume / Dynamic Job / External LLM / Human Review などを管理することを目的とする。

## Design

- [基本設計](docs/design.md)

### Detailed Design

1. [Workflow Definition](docs/detailed-design/01-workflow-definition.md)
2. [Expression / Inputs / Outputs](docs/detailed-design/02-expression-and-inputs.md)
3. [Runtime / Scheduling](docs/detailed-design/03-runtime-and-scheduling.md)
4. [Runner / IPC](docs/detailed-design/04-runner-and-ipc.md)
5. [Dynamic Jobs](docs/detailed-design/05-dynamic-jobs.md)
6. [Reusable Workflows](docs/detailed-design/06-reusable-workflows.md)
7. [External / Human Executor](docs/detailed-design/07-external-and-human.md)
8. [Persistence](docs/detailed-design/08-persistence.md)
9. [Artifact / Log / Workflow State](docs/detailed-design/09-artifacts-logs-state.md)
10. [Retry / Recovery / Cancel](docs/detailed-design/10-retry-recovery-cancel.md)
11. [Service API / MCP / HTTP](docs/detailed-design/11-service-api-and-mcp.md)
12. [Security / Secrets](docs/detailed-design/12-security-and-secrets.md)
13. [Testing](docs/detailed-design/13-testing.md)

現時点では設計フェーズ。WebUI の詳細画面設計は後続で実施する。
