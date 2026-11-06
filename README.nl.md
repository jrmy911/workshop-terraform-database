# Terraform-workshop op Azure

**Doel:** kennismaken met resources, verwijzingen, variabelen, `.tfvars`-bestanden, outputs en de cyclus `plan → apply → destroy`.

**Opzet:** 2 uur. Je maakt slechts drie resources: een resourcegroep, een logische Azure SQL-server en een database. Het onderwerp is Terraform, niet SQL-beheer. Je hebt geen SQL-client, SSH-sleutel of verbinding met de database nodig.

| Stap | Richttijd |
|---|---|
| Voorbereiding | 15 min |
| 1. Eerste resource | 15 min |
| 2. Variabelen en `.tfvars` | 20 min |
| 3. SQL-server en verwijzingen | 15 min |
| 3.1 Database | 15 min |
| 4. Outputs, state en wijzigingen | 30 min |
| 5. Opruimen | 10 min |

## Voorbereiding

Terraform, Azure CLI en een editor moeten geïnstalleerd zijn. Controleer met `terraform version` en `az version`. Werk in deze map, los van de oorspronkelijke workshop: gebruik de state daarvan niet opnieuw.

Azure CLI (`az`) werkt in beide terminals. Meld je aan en selecteer de sandbox met het blok voor jouw terminal.

**Azure CLI in Bash (Linux/macOS/WSL):**

```bash
az login
az account set --subscription "<sandbox-subscriptie-id>"
az account show --output table
export ARM_SUBSCRIPTION_ID="$(az account show --query id --output tsv)"
```

**Azure CLI in PowerShell (Windows):**

```powershell
az login
az account set --subscription "<sandbox-subscriptie-id>"
az account show --output table
$env:ARM_SUBSCRIPTION_ID = (az account show --query id --output tsv)
```

Gebruik je Microsoft Entra-gebruikersaccount met `az login`, geen service principal. De begeleider controleert vooraf de deploymentrechten, het instellen van de Entra-beheerder en de beschikbaarheid van Azure SQL in deze sandbox. Een andere subscription is niet nodig.

Vervang `jdoe` door je eigen afkorting in de Azure-namen: **`rg-terraform-db-workshop-jdoe`** voor John Doe. Vervang voor de SQL-server ook `a1b2` door een persoonlijke, unieke suffix van kleine letters en cijfers. Houd dezelfde waarden gedurende de hele workshop aan.

## 1. Eerste resource

**Doel:** configureer Terraform en maak een resourcegroep waarin je alle workshopresources verzamelt.

- Maak `versions.tf`: vereis Terraform `>= 1.6, < 2.0` en provider `hashicorp/azurerm` met versie `~> 4.81.0`.
- Maak `providers.tf`: configureer de provider `azurerm` met een leeg `features {}`-blok.
- Maak in `main.tf` een resourcegroep van type `azurerm_resource_group`, met Terraform-label `workshop`, naam `rg-terraform-db-workshop-jdoe` en regio `westeurope`.
- Voeg de tags `environment = "acc"` en `workshop = "terraform"` toe. In de volgende stap vervang je de regio en het environment-label door variabelen.

In **Bash of PowerShell**:

```shell
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

Lees het plan voordat je met `yes` bevestigt. In deze stap mag Terraform alleen een resourcegroep maken. De bestanden `acc.tfvars` en `prd.tfvars` gebruik je nog niet.

**Documentatie:** [AzureRM-provider](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs) · [resourcegroep](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/resource_group).

## 2. Variabelen: één configuratie, verschillende waarden

**Doel:** wijzig regio, environment-label en serviceniveau van de database zonder je resourceconfiguratie te herschrijven.

Declareer in `variables.tf`:

| Variabele | Type | Standaardwaarde | Gebruik |
|---|---|---|---|
| `environment` | `string` | Geen; verplicht meegeven | Environment in SQL-namen en de tag `environment`. |
| `location` | `string` | `"westeurope"` | Regio van de resourcegroep. |
| `database_sku` | `string` | `"Basic"` | Serviceniveau van de database; SKU betekent hier de Azure-grootte/-aanbieding. |

Gebruik nu `var.location` voor de regio van de resourcegroep en `var.environment` voor de tag `environment`. Laat de tag `workshop = "terraform"` en de groepsnaam zonder environment-suffix ongewijzigd.

Gebruik de meegeleverde bestanden met deze waarden:

| Bestand | `environment` | `location` | `database_sku` |
|---|---|---|---|
| `acc.tfvars` | `"acc"` | `"westeurope"` | `"Basic"` |
| `prd.tfvars` | `"prd"` | `"westeurope"` | `"S0"` |

Gebruik vanaf nu `acc.tfvars` voor je plannen en deployments. Gebruik `prd.tfvars` alleen om later een plan te bekijken, niet om te deployen: de configuratie is geen productieontwerp en gebruikt dezelfde Terraform-state en resourcegroep.

In **Bash of PowerShell**, na elke wijziging:

```shell
terraform fmt
terraform validate
terraform plan "-var-file=acc.tfvars"
terraform apply "-var-file=acc.tfvars"
```

Zonder `-var-file` laadt Terraform `acc.tfvars` niet automatisch. Je gebruikt `database_sku` pas in stap 3.1; deze variabele verandert nu nog geen resource.

## 3. De omgeving opbouwen: de SQL-server

**Doel:** koppel een SQL-server aan de resourcegroep met Terraform-verwijzingen.

Voeg de resources stapsgewijs toe aan `main.tf`. Vervang `jdoe` overal door je eigen afkorting. `${var.environment}` betekent dat je de variabele in de naam verwerkt, bijvoorbeeld `sql-tfdb-jdoe-acc-a1b2`.

Een **logische Azure SQL-server** is een beheerpunt voor je databases, geen VM. Je installeert zelf geen SQL Server.

Maak een resource `azurerm_mssql_server` met Terraform-label `workshop`:

| Instelling | Vereiste waarde |
|---|---|
| Naam | `sql-tfdb-jdoe-${var.environment}-a1b2`; vervang `jdoe` en `a1b2` |
| Naamvereisten | Wereldwijd uniek, 1 tot 63 tekens, kleine letters, cijfers en streepjes; begint en eindigt met een letter of cijfer |
| Resourcegroep | Verwijzing naar de `name` van de bestaande groep |
| Regio | Verwijzing naar de `location` van de bestaande groep |
| Serverversie | `"12.0"` |
| Minimale TLS-versie | `minimum_tls_version = "1.2"` |
| Publieke netwerktoegang | `public_network_access_enabled = false` |

Om van deze oefening geen authenticatieles te maken, kopieer je deze data source naar `main.tf`, **buiten** de `resource`-blokken:

```hcl
data "azurerm_client_config" "current" {}
```

Kopieer vervolgens dit blok **binnen** je serverresource:

```hcl
azuread_administrator {
  login_username              = "Workshop participant"
  object_id                   = data.azurerm_client_config.current.object_id
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  azuread_authentication_only = true
}
```

De data source leest je bestaande Azure-identiteit; ze maakt geen resource aan. `object_id` identificeert je account, `login_username` is hier een label en er is geen SQL-wachtwoord nodig. Maak geen Entra-gebruiker, Azure-rol of secret aan.

Gebruik `azurerm_resource_group.workshop.name` en `azurerm_resource_group.workshop.location` om de server aan de groep te koppelen. Neem geen waarden uit de portal over. Terraform leidt hieruit de aanmaakvolgorde af: je hoeft geen `depends_on` toe te voegen.

**Publieke toegang staat uit en je maakt geen private toegang: je kunt deze database niet bevragen. Dat is bewust zo.** Terraform kan deze resources via de Azure-beheer-API beheren zonder SQL-verbinding. Controleer de deployment in de portal, niet met een SQL-client. Vraag hulp bij fouten rond rechten, namen of regionale capaciteit; open het netwerk niet om de fout te omzeilen.

Voer `fmt`, `validate`, `plan` en `apply` met `acc.tfvars` uit zoals in stap 2.

**Documentatie:** [SQL-server](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/mssql_server) · [client-configuratie](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/data-sources/client_config).

## 3.1 De database toevoegen

**Doel:** gebruik de ID van een resource en een variabele in een tweede SQL-resource.

Maak een resource `azurerm_mssql_database` met Terraform-label `workshop`:

| Instelling | Vereiste waarde |
|---|---|
| Naam | `sqldb-jdoe-${var.environment}` |
| Server | Verwijs bij `server_id` naar de `id` van je SQL-server |
| Serviceniveau | Gebruik `var.database_sku` bij `sku_name` |
| Maximale grootte | `max_size_gb = 2` |
| Back-upopslag | `storage_account_type = "Local"`; gegeven waarde voor deze sandbox, geen back-upoefening |

De database gebruikt de groep en regio van de server: voeg geen `resource_group_name` of `location` toe aan deze resource. Maak geen tabellen, gegevens, firewall of netwerk aan.

In de documentatie beschrijft **Argument Reference** de instellingen die je opgeeft. **Attributes Reference** beschrijft de waarden waarnaar je kunt verwijzen, zoals `id`.

Voer `fmt`, `validate`, `plan` en `apply` met `acc.tfvars` uit. Het aanmaken van SQL kan enkele minuten duren. Voer daarna hetzelfde plan opnieuw uit: zonder verdere wijzigingen moet het **No changes** aangeven.

Je hebt nu drie beheerde resources: groep → server → database. De data source is geen vierde aan te maken resource. De systeemdatabase `master` kan ook in de portal verschijnen; die heb je niet in Terraform gedeclareerd.

**Documentatie:** [SQL-database](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/mssql_database).

## 4. Bekijken en wijzigen

**Doel:** lees outputs, bekijk de state en onderscheid een wijziging van een vervanging.

Maak in `outputs.tf` de volgende outputs. Gebruik telkens een verwijzing naar de betreffende resource:

| Output | Waarde |
|---|---|
| `resource_group_name` | Naam van de resourcegroep |
| `sql_server_fqdn` | `fully_qualified_domain_name` van de SQL-server; de volledige hostnaam |
| `database_name` | Naam van de database |

Voer `terraform apply "-var-file=acc.tfvars"` uit om de outputs op te slaan, en daarna:

```shell
terraform output
terraform state list
```

De **state** koppelt resourceadressen in je code aan de Azure-resources. De lijst bevat de drie resources en kan ook de data source bevatten. Bewerk of verwijder het statebestand niet; deel het niet, ook niet als er geen wachtwoord in staat.

### Wijzigen zonder te vervangen

Voeg de tag `training = "database"` aan de resourcegroep toe. Voer het plan met `acc.tfvars` uit: zoek `~` voor een wijziging op de bestaande resource, zonder verwijdering van de groep. Pas deze wijziging toe en voer het plan opnieuw uit om **No changes** te zien.

### Vergelijken zonder te deployen

Voer `terraform plan "-var-file=prd.tfvars"` uit. Bekijk de wijzigingen aan tags, namen en SKU. De servernaam hangt van `environment` af: die naam wijzigen vereist vervanging en heeft ook gevolgen voor de gekoppelde database. Zoek de symbolen `+`, `-` en `-/+` of `+/-` in het plan.

**Voer dit plan niet uit.** Dit is geen tweede, geïsoleerde infrastructuur: het gebruikt dezelfde state en resourcegroep. Voer daarna opnieuw `terraform plan "-var-file=acc.tfvars"` uit om te bevestigen dat de gedeployde configuratie niet verandert.

**Leg aan je buur uit:** waarom maakt een verwijzing een afhankelijkheid? Waarom kan een `prd`-plan `acc`-resources vervangen?

## 5. Opruimen voor je vertrekt

**Doel:** verwijder de tijdelijke resources en controleer of alles is opgeruimd.

In **Bash of PowerShell**:

```shell
terraform destroy "-var-file=acc.tfvars"
terraform state list
```

Lees het verwijderingsplan voordat je bevestigt. De database veroorzaakt tijdens haar bestaan kosten, ook zonder verbinding; verwijder de database en server aan het einde van de workshop. Deze configuratie is alleen voor wegwerpresources, nooit voor echte gegevens.

Controleer dat de resourcegroep verdwenen is en de lijst met beheerde resources leeg is. Bewaar bij fouten de state en vraag hulp. Deel geen statebestanden of opgeslagen plannen.

**Onthoud:** een variabele definieert een invoer; een `.tfvars`-bestand geeft die een waarde; een verwijzing verbindt resources; de state koppelt je code aan de Azure-resources.
