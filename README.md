# Azure FinOps Cost Estimator & Budget Gate

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Azure%20FinOps%20Estimator-blue?logo=github)](https://github.com/marketplace/actions/azure-finops-cost-estimator)
[![Version](https://img.shields.io/badge/version-v1.0.0-green.svg)](https://github.com/cscloudsolutions/azure-finops-action/releases)
[![Website](https://img.shields.io/badge/Website-cscloudsolutions.com.ar-0078D4?logo=googlechrome&logoColor=white)](https://cscloudsolutions.com.ar)
[![Platform](https://img.shields.io/badge/SaaS%20Platform-CSCloudSolutions%20FinOps-blueviolet?logo=microsoftazure)](https://finops.cscloudsolutions.com.ar)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Estimate the monthly Azure cost impact of your **Terraform**, **OpenTofu**, and **Azure Bicep** changes **directly inside your Pull Requests**, and optionally block expensive deployments with automated **Budget Gates**.

---

## 🎯 Why Azure FinOps Cost Estimator?

- **Prevent Cloud Bill Shock:** Know whether a pull request adds $50/month or $5,000/month *before* merging into production.
- **Native Azure Pricing Support:** Full pricing coverage for Azure resources including Virtual Machines, Disks, AKS clusters, SQL Databases, App Service Plans, Redis Cache, Azure Firewall, Bastion, VPN Gateways, Public IPs, and more.
- **Both Terraform & Bicep Support:** Works seamlessly with `terraform plan -out=...` and Azure Bicep / ARM `az deployment group what-if`.
- **Zero Tenant Permissions in CI:** Your CI runner only sends sanitized resource descriptors (type, region, SKU, and quantities). **Passes zero secrets, passwords, SSH keys, or tenant credentials.**
- **Sticky Pull Request Comments:** Automatically updates the existing comment on every new commit/push so your PR conversation stays clean and uncluttered.
- **Automated Budget Gates:** Automatically fail the CI check if the infrastructure cost delta exceeds your team's predefined budget threshold.

---

## ⚡ Quick Start

### 1. Generate an API Key
Log in to your [CSCloudSolutions FinOps](https://finops.cscloudsolutions.com.ar) dashboard, navigate to **Integrations & API → Public API Keys**, and generate an API key with the `write:shift-left` scope. Save it as a repository secret named `FINOPS_API_KEY`.

### 2. Add Workflow to your Repository

Create `.github/workflows/finops-cost.yml`:

```yaml
name: Azure FinOps Cost Estimator
on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write   # Required to post and update the cost comment

jobs:
  cost-estimate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_wrapper: false

      - name: Terraform Init & Plan
        run: |
          terraform init -input=false
          terraform plan -input=false -out=tfplan.bin

      - name: Estimate Azure Cost & Budget Gate
        uses: cscloudsolutions/azure-finops-action@v1
        with:
          api-key: ${{ secrets.FINOPS_API_KEY }}
          plan-file: tfplan.bin
          # threshold-usd: '500' # Optional: fail build if cost delta exceeds $500 USD/month
```

---

## 🚀 Azure Bicep Example

```yaml
name: Azure Bicep Cost Estimator
on:
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write        # Required for Azure OIDC login

jobs:
  estimate-bicep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Generate Bicep What-If JSON
        run: |
          az deployment group what-if \
            --resource-group rg-production \
            --template-file main.bicep \
            --no-pretty-print > whatif.json

      - name: Estimate Bicep Cost
        uses: cscloudsolutions/azure-finops-action@v1
        with:
          api-key: ${{ secrets.FINOPS_API_KEY }}
          tool: bicep
          plan-json: whatif.json
          threshold-usd: '300'
```

---

## 📊 Pull Request Comment Preview

When the action runs, it posts an interactive, detailed sticky summary:

```markdown
<!-- finops-shift-left -->
## 🔺 Monthly Azure Cost Impact

| | USD/month |
|---|---:|
| Before | $612.50 |
| After | $3,789.41 |
| **Delta** | **+$3,176.91** |
| List Price Delta | +$3,971.14 |

> Prices calculated with your 20% tenant discount applied.
> ↩️ Previous push: +$2,800.00 → now +$3,176.91 (increase).
**Cost Center «DevOps Hub» Budget:** this change equals 21.18% of monthly budget ($15,000.00); $4,200.00 remaining this month.
✅ **Budget Gate:** within allowable threshold.
> ⚠️ 2 resource(s) unpriced or billed on consumption (1 usage-based): delta represents a lower floor, not the total.

<details><summary>Resource Breakdown</summary>

| Resource | Action | Before | After | Notes |
|---|---|---:|---:|---|
| `azurerm_firewall.hub` | create | $0.00 | $912.50 | does not include processed data |
| `azurerm_linux_virtual_machine.app` | update | $140.00 | $280.00 | Spot pricing: varies with demand |
| `azurerm_storage_account.logs` | create | $0.00 | $0.00 | ⚠️ usage-based billing |

</details>

<sub>Azure Retail Prices API list pricing. Powered by [CSCloudSolutions FinOps](https://finops.cscloudsolutions.com.ar)</sub>
```

---

## ⚙️ Inputs

| Name | Required | Default | Description |
| :--- | :--- | :--- | :--- |
| `api-key` | **Yes** | — | Platform API key with `write:shift-left` scope. Store in GitHub Secrets. |
| `tool` | No | `terraform` | IaC tool used: `terraform`, `tofu`, or `bicep`. |
| `plan-file` | No | `tfplan.bin` | Binary plan file from `terraform plan -out=...`. Ignored if `plan-json` is provided. |
| `plan-json` | No | `''` | Plan already formatted as JSON, or Bicep `what-if` output. |
| `threshold-usd` | No | `''` | Workflow budget threshold in USD/month. If platform also defines one, strictest wins. |
| `comment` | No | `true` | Post sticky comment to Pull Request (`true`, `false`, `auto`). |
| `api-url` | No | `https://finops.cscloudsolutions.com.ar` | Platform endpoint URL. |
| `script-sha256`| No | `''` | Expected SHA-256 hash of calculation script. |

---

## 📤 Outputs

| Name | Description |
| :--- | :--- |
| `delta-usd` | Monthly cost delta in USD (negative if PR saves money). |
| `projected-usd` | Total projected monthly Azure cost in USD. |
| `unpriced` | Count of resources unpriced or billed strictly by consumption. |
| `gate-status` | Budget Gate evaluation: `OK`, `EXCEEDED`, or `NOT_EVALUATED`. |

---

## 🔒 Security & Privacy

1. **Client-Side Sanitization:** The plan is sanitized directly on your GitHub runner before network transit. Only resource types, regions, and SKUs are transmitted. Passwords, certificates, and metadata never leave your runner.
2. **Fail-Open Resilience:** If the FinOps platform suffers a network timeout or outage, your CI build **will not be blocked** (`exit 0`). Cost tracking never halts your development velocity.
3. **Fail-Closed Governance:** If a Budget Gate threshold is exceeded, the build is intentionally failed (`exit 1`) to prevent unauthorized cloud expenses.

---

## 📄 License

Distributed under the MIT License. See [LICENSE](LICENSE) for more information.

Developed by [CSCloudSolutions](https://cscloudsolutions.com.ar).
