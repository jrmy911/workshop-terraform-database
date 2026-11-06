# Terraform on Azure — database assignment

**Two hours of hands-on Terraform**, using just a resource group, an Azure SQL logical server and one small database in the existing sandbox subscription.

- **Open [assignments.html](assignments.html)** for the self-contained French/Dutch participant guide. No web server or internet connection is needed to read it; Azure deployment and the linked provider documentation require internet access.
- Markdown instructions: [Français](README.fr.md) · [Nederlands](README.nl.md).
- `acc.tfvars`: workshop deployment (`Basic`, 2 GB).
- `prd.tfvars`: compare a plan only; **do not apply it**.

Participants create their own `versions.tf`, `providers.tf`, `variables.tf`, `main.tf` and `outputs.tf` as they work through the exercises. Completed Terraform code is intentionally not included here.

The general introduction, Terraform learning objectives and sandbox login instructions are retained from the original workshop. VM/SSH preparation and infrastructure exercises are replaced by the small database scenario.

## Scope

Practise resources, variables, `.tfvars`, references and dependencies, outputs, state, an in-place tag update, a preview of replacement, and cleanup. No SQL queries, database clients, passwords, networking, monitoring or backup/restore exercises.

Public SQL access is disabled and no private access is created. This is deliberately an infrastructure provisioning exercise, not a usable or production-ready database environment. Terraform uses Azure's management API; it does not need to connect to the database.

Use a new local state in this directory and the database-specific resource names from the guide. Never reuse the original workshop's state. Do not have this assignment and the instructor solution manage the same Azure resources from separate states.

## Instructor preparation

Before the session, confirm Terraform and Azure CLI are installed, `Microsoft.Sql` is registered, the sandbox permits the resources and settings, participants can configure the Entra SQL administrator and Entra-only authentication, and `Basic` is available in the chosen region. The supplied identity snippet assumes interactive `az login` with an existing user account, not a service principal. Resolve subscription, permissions and capacity issues before the two-hour session.

Useful read-only checks:

```shell
az account show --output table
az provider show --namespace Microsoft.Sql --query registrationState --output tsv
az sql db list-editions --location westeurope --output table
```

These checks do not prove every permission or capacity requirement; the instructor should rehearse a deployment beforehand. The assistant has not deployed this assignment to Azure.

The database incurs charges while it exists. Reserve the final ten minutes for `terraform destroy "-var-file=acc.tfvars"` and verification. Use synthetic, disposable resources only.
