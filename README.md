# databricks-appinsights

A Databricks notebook that queries an Azure Monitor Log Analytics workspace to surface observability data for Gen AI workloads instrumented with Application Insights (for example, Microsoft Foundry / Azure AI Agents).

The notebook lives at [`src/Application Insights Query.ipynb`](<src/Application Insights Query.ipynb>).

## Overview

The notebook runs four KQL queries and displays each result as a pandas DataFrame:

| Section | Table | Timespan | What it shows |
|---|---|---|---|
| Content traces | `AppGenAIContent` | 7 days | The 100 most recent rows: agent names, model identifiers, input/output messages, tool calls, and evaluation explanations. |
| Daily tool-call performance by agent | `AppDependencies` | 31 days | Per day, agent, operation, tool, and app role: tool-call count, failed calls, failure rate (%), average and p95 latency (ms), and sample result codes and targets. |
| Daily token usage | `AppDependencies` | 31 days | Daily input, output, and total tokens per model. |
| Daily token usage by model and operation | `AppDependencies` | 31 days | Daily call count and input, output, and total tokens by model, provider, and operation type. |

Gen AI attributes (such as `gen_ai.operation.name`, `gen_ai.agent.name`, `gen_ai.tool.name`, `gen_ai.request.model`, and `gen_ai.usage.*_tokens`) are read from the `Properties` column of `AppDependencies`. Rows are filtered to Microsoft Foundry / Azure AI Agents spans, tool-call spans, or dependencies that report a model or token usage.

## Process

1. **Configure access**
   - In **Application Insights → Overview**, find the linked Log Analytics workspace and copy its **Workspace ID** (not the Azure resource ID or the instrumentation key).
   - Grant a Microsoft Entra service principal the **Log Analytics Reader** role on that workspace.
   - Store the service principal's tenant ID, client ID, and client secret as secrets in Azure Key Vault, then create a [Key Vault-backed secret scope](https://learn.microsoft.com/azure/databricks/security/secrets/secret-scopes#create-an-azure-key-vault-backed-secret-scope) in Databricks.
2. **Install the SDK** – the notebook runs `%pip install azure-monitor-query azure-identity` and then restarts Python.
3. **Authenticate** – each query cell builds a `ClientSecretCredential` from the secret scope via `dbutils.secrets.get(...)`. Your Databricks login does not grant access to Azure Monitor on its own.
4. **Query** – each cell uses `LogsQueryClient.query_workspace(...)` with the workspace ID, a KQL query, and a timespan. It raises an error if the result is partial.
5. **Display** – rows are loaded into a pandas DataFrame and shown with `display(...)`. If no rows match, a message is printed instead.

## Configuration to adjust

Update these values in each code cell before running:

- `key_vault_name` – name of your Key Vault-backed Databricks secret scope.
- Secret keys `SPTenantID`, `SPDatabricksAppID`, and `SPDatabricks` – the tenant ID, client ID, and client secret.
- `workspace_id` – your Log Analytics Workspace ID.
- `timespan` – the lookback window for each query.

## Requirements

- A Databricks workspace with a Key Vault-backed secret scope
- Application Insights linked to a Log Analytics workspace with Gen AI telemetry
- A service principal with Log Analytics Reader on that workspace
- Python packages `azure-monitor-query` and `azure-identity` (installed by the notebook)