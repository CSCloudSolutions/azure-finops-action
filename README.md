# Azure FinOps Cost Estimator & Budget Gate

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Azure%20FinOps%20Estimator-blue?logo=github)](https://github.com/marketplace/actions/azure-finops-cost-estimator)
[![Version](https://img.shields.io/badge/version-v1.0.0-green.svg)](https://github.com/manny864/azure-finops-action/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Powered by CSCloudSolutions](https://img.shields.io/badge/Platform-CSCloudSolutions%20FinOps-0078D4?logo=microsoftazure)](https://finops.cscloudsolutions.com.ar)
[![Powered by CSCloudSolutions](https://img.shields.io/badge/Website-CSCloudSolutions?logo=microsoftazure)](https://cscloudsolutions.com.ar)


# Azure FinOps Cost Estimator & Budget Gate

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Azure%20FinOps%20Estimator-blue?logo=github)](https://github.com/marketplace/actions/azure-finops-cost-estimator)
[![Version](https://img.shields.io/badge/version-v1.0.0-green.svg)](https://github.com/cscloudsolutions/azure-finops-action/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Powered by CSCloudSolutions](https://img.shields.io/badge/Platform-CSCloudSolutions%20FinOps-0078D4?logo=microsoftazure)](https://finops.cscloudsolutions.com.ar)

Estimate the monthly Azure cost impact of your **Terraform**, **OpenTofu**, and **Azure Bicep** changes **directly inside your Pull Requests**, and optionally block expensive deployments with automated **Budget Gates**.

---

## 🎯 Why Azure FinOps Cost Estimator?

- **Prevent Cloud Bill Shock:** Know if a change adds $50/month or $5,000/month *before* merging into production.
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

      - name: Estimate Azure Cost & Budget Gate
        uses: cscloudsolutions/azure-finops-action@v1
        with:
          api-key: ${{ secrets.FINOPS_API_KEY }}
          plan-file: tfplan.bin
          # threshold-usd: '500' # Optional: fail if monthly cost delta exceeds $500 USD
