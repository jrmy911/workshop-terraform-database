# Atelier Terraform sur Azure

**Objectif :** découvrir les ressources, références, variables, fichiers `.tfvars`, outputs et le cycle `plan → apply → destroy`.

**Format :** 2 heures. Vous créez seulement trois ressources : un groupe de ressources, un serveur logique Azure SQL et une base de données. Le sujet est Terraform, pas l’administration SQL. Aucun client SQL, aucune clé SSH et aucune connexion à la base ne sont nécessaires.

| Étape | Durée indicative |
|---|---|
| Préparation | 15 min |
| 1. Première ressource | 15 min |
| 2. Variables et `.tfvars` | 20 min |
| 3. Serveur SQL et références | 15 min |
| 3.1 Base de données | 15 min |
| 4. Outputs, state et modifications | 30 min |
| 5. Nettoyage | 10 min |

## Préparation

Terraform, Azure CLI et un éditeur doivent être installés. Vérifiez avec `terraform version` et `az version`. Travaillez dans ce dossier, séparément de l’ancien atelier : ne réutilisez pas son state.

Azure CLI (`az`) fonctionne dans les deux terminaux. Connectez-vous et sélectionnez la sandbox avec le bloc adapté à votre terminal.

**Azure CLI dans Bash (Linux/macOS/WSL) :**

```bash
az login
az account set --subscription "<id-souscription-sandbox>"
az account show --output table
export ARM_SUBSCRIPTION_ID="$(az account show --query id --output tsv)"
```

**Azure CLI dans PowerShell (Windows) :**

```powershell
az login
az account set --subscription "<id-souscription-sandbox>"
az account show --output table
$env:ARM_SUBSCRIPTION_ID = (az account show --query id --output tsv)
```

Utilisez votre compte utilisateur Microsoft Entra avec `az login`, pas un service principal. Le formateur vérifie au préalable les droits de déploiement, la configuration de l’administrateur Entra et la disponibilité d’Azure SQL dans cette sandbox. Une autre souscription n’est pas nécessaire.

Remplacez `jdoe` par votre abréviation dans les noms Azure : **`rg-terraform-db-workshop-jdoe`** pour John Doe. Pour le serveur SQL, remplacez aussi `a1b2` par un suffixe personnel unique composé de lettres minuscules et de chiffres. Gardez ces mêmes valeurs pendant tout l’atelier.

## 1. Première ressource

**Objectif :** configurer Terraform et créer un groupe de ressources pour toutes les ressources de l’atelier.

- Créez `versions.tf` : exigez Terraform `>= 1.6, < 2.0` et le provider `hashicorp/azurerm`, version `~> 4.81.0`.
- Créez `providers.tf` : configurez le provider `azurerm` avec un bloc `features {}` vide.
- Dans `main.tf`, créez un groupe de ressources de type `azurerm_resource_group`, avec le label Terraform `workshop`, le nom `rg-terraform-db-workshop-jdoe` et la région `westeurope`.
- Ajoutez les tags `environment = "acc"` et `workshop = "terraform"`. À l’étape suivante, vous remplacerez la région et la valeur d’environnement par des variables.

Dans **Bash ou PowerShell** :

```shell
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

Lisez le plan avant de confirmer avec `yes`. À cette étape, Terraform ne doit créer qu’un groupe de ressources. Les fichiers `acc.tfvars` et `prd.tfvars` ne sont pas encore utilisés.

**Documentation :** [provider AzureRM](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs) · [groupe de ressources](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/resource_group).

## 2. Variables : une configuration, plusieurs valeurs

**Objectif :** modifier la région, l’environnement et le niveau de service de la base sans réécrire la configuration des ressources.

Déclarez dans `variables.tf` :

| Variable | Type | Valeur par défaut | Utilisation |
|---|---|---|---|
| `environment` | `string` | Aucune ; valeur obligatoire | Environnement dans les noms SQL et le tag `environment`. |
| `location` | `string` | `"westeurope"` | Région du groupe de ressources. |
| `database_sku` | `string` | `"Basic"` | Niveau de service de la base ; SKU signifie ici la taille/offre Azure. |

Utilisez maintenant `var.location` pour la région du groupe et `var.environment` pour le tag `environment`. Conservez le tag `workshop = "terraform"` et le nom du groupe sans suffixe d’environnement.

Utilisez les fichiers fournis avec ces valeurs :

| Fichier | `environment` | `location` | `database_sku` |
|---|---|---|---|
| `acc.tfvars` | `"acc"` | `"westeurope"` | `"Basic"` |
| `prd.tfvars` | `"prd"` | `"westeurope"` | `"S0"` |

Utilisez désormais `acc.tfvars` pour vos plans et déploiements. Utilisez `prd.tfvars` uniquement pour observer un plan plus tard, sans le déployer : cette configuration n’est pas une architecture de production et utilise le même state Terraform et le même groupe de ressources.

Dans **Bash ou PowerShell**, après chaque modification :

```shell
terraform fmt
terraform validate
terraform plan "-var-file=acc.tfvars"
terraform apply "-var-file=acc.tfvars"
```

Sans `-var-file`, Terraform ne charge pas automatiquement `acc.tfvars`. La variable `database_sku` sera utilisée à l’étape 3.1 ; elle ne change encore aucune ressource.

## 3. Construire l’environnement : le serveur SQL

**Objectif :** relier un serveur SQL au groupe de ressources avec des références Terraform.

Ajoutez progressivement les ressources à `main.tf`. Remplacez partout `jdoe` par votre abréviation. `${var.environment}` signifie que vous intégrez la variable au nom, par exemple `sql-tfdb-jdoe-acc-a1b2`.

Un **serveur logique Azure SQL** est un point de gestion pour vos bases : ce n’est pas une VM. Vous n’installez pas SQL Server.

Créez une ressource `azurerm_mssql_server` avec le label Terraform `workshop` :

| Paramètre | Valeur requise |
|---|---|
| Nom | `sql-tfdb-jdoe-${var.environment}-a1b2` ; remplacez `jdoe` et `a1b2` |
| Contraintes du nom | Unique mondialement, 1 à 63 caractères, lettres minuscules, chiffres et tirets ; commence et finit par une lettre ou un chiffre |
| Groupe de ressources | Référence au `name` du groupe existant |
| Région | Référence à la `location` du groupe existant |
| Version du serveur | `"12.0"` |
| TLS minimum | `minimum_tls_version = "1.2"` |
| Accès réseau public | `public_network_access_enabled = false` |

Pour ne pas transformer l’exercice en cours d’authentification, copiez cette data source dans `main.tf`, **en dehors** des blocs `resource` :

```hcl
data "azurerm_client_config" "current" {}
```

Copiez ensuite ce bloc **à l’intérieur** de votre ressource serveur :

```hcl
azuread_administrator {
  login_username              = "Workshop participant"
  object_id                   = data.azurerm_client_config.current.object_id
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  azuread_authentication_only = true
}
```

La data source lit votre identité Azure existante ; elle ne crée pas de ressource. `object_id` identifie votre compte, `login_username` est ici un libellé et aucun mot de passe SQL n’est nécessaire. Ne créez ni utilisateur Entra, ni rôle Azure, ni secret.

Utilisez `azurerm_resource_group.workshop.name` et `azurerm_resource_group.workshop.location` pour relier le serveur au groupe. Ne recopiez pas les valeurs du portail. Terraform en déduit l’ordre de création : pas besoin d’ajouter `depends_on`.

**L’accès public est désactivé et aucun accès privé n’est créé : vous ne pourrez pas interroger cette base. C’est volontaire.** Terraform peut gérer ces ressources via l’API de gestion Azure sans connexion SQL. Vérifiez le déploiement dans le portail, pas avec un client SQL. En cas d’erreur de droits, de nom ou de capacité régionale, demandez de l’aide ; n’ouvrez pas le réseau pour contourner l’erreur.

Exécutez `fmt`, `validate`, `plan` et `apply` avec `acc.tfvars` comme à l’étape 2.

**Documentation :** [serveur SQL](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/mssql_server) · [configuration client](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/data-sources/client_config).

## 3.1 Ajouter la base de données

**Objectif :** utiliser l’ID d’une ressource et une variable dans une deuxième ressource SQL.

Créez une ressource `azurerm_mssql_database` avec le label Terraform `workshop` :

| Paramètre | Valeur requise |
|---|---|
| Nom | `sqldb-jdoe-${var.environment}` |
| Serveur | Pour `server_id`, référencez l’`id` de votre serveur SQL |
| Niveau de service | Pour `sku_name`, utilisez `var.database_sku` |
| Taille maximale | `max_size_gb = 2` |
| Stockage des sauvegardes | `storage_account_type = "Local"` ; valeur fournie pour cette sandbox, pas un exercice de sauvegarde |

La base utilise le groupe et la région de son serveur : inutile d’ajouter `resource_group_name` ou `location` à cette ressource. Ne créez ni tables, ni données, ni pare-feu, ni réseau.

Dans la documentation, **Argument Reference** décrit les valeurs à configurer ; **Attributes Reference** décrit les valeurs à référencer, par exemple `id`.

Exécutez `fmt`, `validate`, `plan` et `apply` avec `acc.tfvars`. La création SQL peut prendre plusieurs minutes. Une fois terminée, relancez le même plan : sans autre modification, il doit indiquer **No changes**.

Vous avez maintenant trois ressources gérées : groupe → serveur → base. La data source n’est pas une quatrième ressource à créer. La base système `master` peut aussi apparaître dans le portail ; vous ne l’avez pas déclarée dans Terraform.

**Documentation :** [base de données SQL](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/mssql_database).

## 4. Observer et modifier

**Objectif :** lire des outputs, inspecter le state et distinguer une modification d’un remplacement.

Dans `outputs.tf`, créez les outputs suivants. Utilisez chaque fois une référence à la ressource concernée :

| Output | Valeur |
|---|---|
| `resource_group_name` | Nom du groupe de ressources |
| `sql_server_fqdn` | `fully_qualified_domain_name` du serveur SQL ; son nom d’hôte complet |
| `database_name` | Nom de la base de données |

Exécutez `terraform apply "-var-file=acc.tfvars"` pour enregistrer les outputs, puis :

```shell
terraform output
terraform state list
```

Le **state** associe les adresses de ressources dans votre code aux ressources Azure. La liste contient les trois ressources et peut aussi contenir la data source. Ne modifiez et ne supprimez pas le fichier state ; ne le partagez pas, même sans mot de passe.

### Modifier sans remplacer

Ajoutez le tag `training = "database"` au groupe de ressources. Exécutez le plan avec `acc.tfvars` : repérez `~` pour une mise à jour sur place, sans suppression du groupe. Appliquez ce changement, puis relancez le plan pour retrouver **No changes**.

### Comparer sans déployer

Exécutez `terraform plan "-var-file=prd.tfvars"`. Observez les changements de tags, de noms et de SKU. Le nom du serveur dépend de `environment` : son changement impose un remplacement, et la base liée est également affectée. Repérez les symboles `+`, `-` et `-/+` ou `+/-` dans le plan.

**N’appliquez pas ce plan.** Ce n’est pas une deuxième infrastructure isolée : il utilise le même state et le même groupe de ressources. Relancez ensuite `terraform plan "-var-file=acc.tfvars"` pour confirmer que la configuration déployée ne change pas.

**À expliquer à votre voisin :** pourquoi une référence crée-t-elle une dépendance ? Pourquoi un plan `prd` peut-il remplacer des ressources `acc` ?

## 5. Nettoyer avant de partir

**Objectif :** supprimer les ressources temporaires et vérifier le nettoyage.

Dans **Bash ou PowerShell** :

```shell
terraform destroy "-var-file=acc.tfvars"
terraform state list
```

Lisez le plan de suppression avant de confirmer. La base génère des coûts pendant son existence, même sans connexion ; supprimez la base et le serveur à la fin de l’atelier. Cette configuration ne contient que des ressources jetables, jamais des données réelles.

Vérifiez que le groupe de ressources a disparu et que la liste des ressources suivies est vide. En cas d’erreur, conservez le state et demandez de l’aide. Ne partagez pas les fichiers state ou les plans enregistrés.

**À retenir :** une variable déclare une entrée ; un `.tfvars` lui donne une valeur ; une référence relie des ressources ; le state associe votre code aux ressources Azure.
