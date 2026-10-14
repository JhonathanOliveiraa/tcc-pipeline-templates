# tcc-pipeline-templates

Templates compartilhados de GitHub Actions das aplicações do TCC — o equivalente, no GitHub, a um
repositório de templates `extends` de Azure Pipelines. Cada aplicação mantém só um `ci.yml` curto que
chama um destes workflows reutilizáveis.

| Template | Para |
|---|---|
| `.github/workflows/backend-service.yml` | APIs e serviços .NET em contêiner |
| `.github/workflows/frontend-service.yml` | Front-ends Node/Angular servidos por contêiner |

Os dois seguem a mesma esteira: **testes → Trivy no código → build `linux/arm64` → push no GHCR →
Trivy na imagem → commit da nova tag no overlay da aplicação no `tcc-platform`**. A esteira nunca
acessa o cluster; quem aplica é o Argo CD, a partir do Git.

## Como consumir

`.github/workflows/ci.yml` na aplicação:

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  pipeline:
    uses: <owner>/tcc-pipeline-templates/.github/workflows/backend-service.yml@main
    with:
      platform-repository: <owner>/tcc-platform
      overlay-path: apps/catalogo-api
      enable-deploy: ${{ github.event_name != 'pull_request' }}
    secrets:
      platform-token: ${{ secrets.PLATFORM_REPO_TOKEN }}
```

## Parâmetros

Comuns: `platform-repository`, `overlay-path`, `dockerfile`, `build-context`, `docker-build-args`,
`enable-trivy-filesystem-scan`, `enable-trivy-image-scan`, `enable-deploy`.

- Backend: `dotnet-version`, `test-command` (vazio pula os testes).
- Frontend: `node-version`, `working-directory`, `install-command`, `test-command`, `build-command`.

Imagem: `ghcr.io/<owner>/<repositório>` com tag imutável `sha-<12 primeiros do commit>`.

## Pré-requisitos

1. **Este repositório** (privado): *Settings → Actions → General → Access* →
   "Accessible from repositories owned by the user" — sem isso, os outros repositórios privados não
   conseguem chamar os templates.
2. **Em cada aplicação:** secret `PLATFORM_REPO_TOKEN` com um *fine-grained token* restrito ao
   repositório `tcc-platform`, permissão **Contents: Read and write**.
3. **No `tcc-platform`:** o overlay `apps/<app>/kustomization.yaml` precisa existir, com o bloco
   `images:` contendo `name: ghcr.io/<owner>/<app>` e `newTag:`.
4. **Dockerfile** da aplicação gerando `linux/arm64` (a VM é Ampere/ARM64); compile com
   `FROM --platform=$BUILDPLATFORM` para não emular o build inteiro.

Regra: se uma aplicação precisar de algo muito fora do padrão, crie um terceiro template em vez de
encher os dois templates-base de exceções.
