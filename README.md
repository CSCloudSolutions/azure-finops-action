# Azure FinOps Cost Estimator & Budget Gate

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Azure%20FinOps%20Estimator-blue?logo=github)](https://github.com/marketplace/actions/azure-finops-cost-estimator)
[![Version](https://img.shields.io/badge/version-v1.0.0-green.svg)](https://github.com/manny864/azure-finops-action/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Powered by CSCloudSolutions](https://img.shields.io/badge/Platform-CSCloudSolutions%20FinOps-0078D4?logo=microsoftazure)](https://finops.cscloudsolutions.com.ar)
[![Powered by CSCloudSolutions](https://img.shields.io/badge/Website-CSCloudSolutions%20FinOps-0078D4?logo=microsoftazure)](https://cscloudsolutions.com.ar)
Estimate the monthly Azure cost impact of your **Terraform**, **OpenTofu**, and **Azure Bicep** changes **directly inside your Pull Requests**, and optionally block expensive deployments with automated **Budget Gates**.

---

## 🎯 Why Azure FinOps Cost Estimator?

- **Prevent Cloud Bill Shock:** Know if a change adds \$50/month or \$5,000/month *before* merging into production.
- **Native Azure Support:** Full pricing support for Azure resources including VMs, Disks, AKS, SQL, App Service Plans, Redis, Azure Firewall, Bastion, VPN Gateways, Public IPs, and more.
- **Both Terraform & Bicep:** Works with `terraform plan -out=...` and Azure Bicep / ARM `az deployment group what-if`.
- **Zero Tenant Permissions in CI:** Your CI runner only sends sanitized resource descriptors (type, region, SKU, quantities). **Passes zero secrets, passwords, or credentials.**
- **Sticky Pull Request Comments:** Automatically updates the existing comment on every push so your PR timeline stays clean.
- **Automated Budget Gates:** Fail the CI build if the infrastructure cost delta exceeds your predefined threshold.

---

## ⚡ Quick Start

### 1. Generate an API Key
Log in to your [CSCloudSolutions FinOps](https://finops.cscloudsolutions.com.ar) dashboard, go to **Integrations & API → Public API Keys**, and generate an API key with the `write:shift-left` scope. Save it in your repository secrets as `FINOPS_API_KEY`.

### 2. Add Workflow to your Repository

Create `.github/workflows/finops-cost.yml`:

```yaml
name: Azure FinOps Cost Estimator
on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write   # Required to post/update the cost summary comment

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

      - name: Run Azure FinOps Cost Estimator
        uses: cscloudsolutions/azure-finops-action@v1
        with:
          api-key: ${{ secrets.FINOPS_API_KEY }}
          plan-file: tfplan.bin
          # threshold-usd: '500' # Optional: fail if monthly cost delta exceeds $500 USD
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

When the action runs, it posts an interactive, detailed breakdown:

```markdown
<!-- finops-shift-left -->
## 🔺 Impacto en el costo mensual de Azure

| | USD/mes |
|---|---:|
| Antes | $612.50 |
| Después | $3,789.41 |
| **Delta** | **+$3,176.91** |
| Delta a precio de lista | +$3,971.14 |

> Precios con tu descuento del 20% aplicado.
> ↩️ Push anterior: +$2,800.00 → ahora +$3,176.91 (sube).
**Presupuesto de «DevOps Hub»:** este cambio equivale al 21.18% del presupuesto mensual ($15,000.00); quedan $4,200.00 del mes.
✅ **Budget Gate:** dentro del umbral.
> ⚠️ 2 recurso(s) no se pudieron tarifar o se cobran por consumo (1 por consumo): el delta es un piso, no el total.

<details><summary>Desglose por recurso</summary>

| Recurso | Acción | Antes | Después | Notas |
|---|---|---:|---:|---|
| `azurerm_firewall.hub` | create | $0.00 | $912.50 | no incluye los datos procesados |
| `azurerm_linux_virtual_machine.app` | update | $140.00 | $280.00 | precio Spot: varía con la demanda |
| `azurerm_storage_account.logs` | create | $0.00 | $0.00 | ⚠️ se cobra por consumo |

</details>

<sub>Calculado por [CSCloudSolutions FinOps](https://finops.cscloudsolutions.com.ar)</sub>
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

Developed by [CSCloudSolutions](https://finops.cscloudsolutions.com.ar).
