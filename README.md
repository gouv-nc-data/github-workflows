# github-workflows

Workflows GitHub Actions réutilisables (`workflow_call`) pour les dépôts `gouv-nc-data`.

Appel depuis un dépôt :

```yaml
jobs:
  <job>:
    uses: gouv-nc-data/github-workflows/.github/workflows/<workflow>.yml@main
```

| Workflow | Usage |
|---|---|
| [`terraform.yml`](#terraformyml) | Plan / apply d'une stack Terraform |
| [`terraform-docs.yml`](#terraform-docsyml) | Doc d'un module Terraform dans le README |
| [`build-push-on-gcp.yml`](#build-push-on-gcpyml) | Build Docker → Artifact Registry |
| [`build-push-on-gcp-and-deploy-cloudrun.yml`](#build-push-on-gcp-and-deploy-cloudrunyml) | Build → Artifact Registry → Cloud Run service |
| [`build-push-on-gcp-and-deploy-cloudrun-job.yml`](#build-push-on-gcp-and-deploy-cloudrun-jobyml) | Build → Artifact Registry → Cloud Run job |
| [`build-push-on-github.yml`](#build-push-on-githubyml) | Build Docker → GHCR |

---

## terraform.yml

`fmt -check` → `init` → `state list` → `validate` → `plan` (PR) → `apply -auto-approve` (push sur `main`).
Sur PR, le résultat du plan est posté en commentaire (mis à jour à chaque push).

| | Nom | Requis | Défaut |
|---|---|---|---|
| input | `tf_working_dir` | non | `.` |
| secret | `GOOGLE_CREDENTIALS` | oui | |
| secret | `GH_TOKEN` | non | exposé en `GITHUB_TOKEN` et `TF_VAR_github_token` |

```yaml
on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  issues: write
  pull-requests: write

jobs:
  terraform:
    uses: gouv-nc-data/github-workflows/.github/workflows/terraform.yml@main
    secrets: inherit
    # with:
    #   tf_working_dir: terraform
```

## terraform-docs.yml

Injecte la doc [terraform-docs](https://terraform-docs.io) dans `README.md` et pousse le commit sur `main`.
Le README doit contenir les balises `<!-- BEGIN_TF_DOCS -->` / `<!-- END_TF_DOCS -->`.

L'input `tf_working_dir` existe mais n'est pas utilisé : la doc est toujours générée depuis la racine.

```yaml
on:
  push:
    branches: [main]
    paths: ['**/*.tf']

permissions:
  contents: write

jobs:
  docs:
    uses: gouv-nc-data/github-workflows/.github/workflows/terraform-docs.yml@main
```

## build-push-on-gcp.yml

Build du `Dockerfile` racine et push vers Artifact Registry.
Image : `<GCR_REGION>-docker.pkg.dev/<GCP_PROJECT_ID>/<GCP_REPO_NAME>/<nom du dépôt>`.
Tags : branche, semver, `sha-<court>`, `latest` sur la branche par défaut.

| Type | Nom |
|---|---|
| secret | `GCP_CREDENTIALS` (clé JSON du compte de service) |
| variable | `GCR_REGION`, `GCP_PROJECT_ID`, `GCP_REPO_NAME` |

## build-push-on-gcp-and-deploy-cloudrun.yml

Build, push puis déploiement d'un **service** Cloud Run sur l'image `sha-<commit>`.
Image : `<GCP_REGION>-docker.pkg.dev/<GCP_PROJECT_ID>/<GCP_REPOSITORY>/<PROJECT_NAME>-function`, taguée `sha-<commit>` et `latest`.

| Type | Nom |
|---|---|
| secret | `GCP_CREDENTIALS` |
| variable | `GCP_REGION`, `GCP_PROJECT_ID`, `GCP_REPOSITORY`, `PROJECT_NAME`, `GCP_CLOUD_SERVICE` |

## build-push-on-gcp-and-deploy-cloudrun-job.yml

Identique au précédent, mais met à jour un **job** Cloud Run.

| Type | Nom |
|---|---|
| secret | `GCP_CREDENTIALS` |
| variable | `GCP_REGION`, `GCP_PROJECT_ID`, `GCP_REPOSITORY`, `PROJECT_NAME`, `GCP_CR_JOB_NAME` |

Exemple (valable aussi pour les deux workflows précédents) :

```yaml
on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  build:
    uses: gouv-nc-data/github-workflows/.github/workflows/build-push-on-gcp-and-deploy-cloudrun-job.yml@main
    secrets:
      GCP_CREDENTIALS: ${{ secrets.GCP_CREDENTIALS }}
```

## build-push-on-github.yml

Build du `Dockerfile` racine et push vers `ghcr.io/<owner>/<repo>:latest`. Aucun secret : le workflow utilise `GITHUB_TOKEN`.

```yaml
permissions:  # le workflow appelé exige toutes ces permissions
  contents: read
  packages: write
  id-token: write
  checks: write
  pull-requests: write

jobs:
  build:
    uses: gouv-nc-data/github-workflows/.github/workflows/build-push-on-github.yml@main
```

---

Les variables (`vars.*`) se définissent dans *Settings → Secrets and variables → Actions* du dépôt appelant ou au niveau de l'organisation.
