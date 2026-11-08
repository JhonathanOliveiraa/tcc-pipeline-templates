# tcc-pipeline-templates

Templates compartilhados de GitHub Actions das aplicações do TCC — o equivalente, no GitHub, a um
repositório de templates `extends` de Azure Pipelines. Cada aplicação mantém só um `ci.yml` curto que
chama um destes workflows reutilizáveis.

| Template | Para |
|---|---|
| `.github/workflows/backend-service.yml` | APIs e serviços .NET em contêiner |
| `.github/workflows/frontend-service.yml` | Front-ends Node/Angular servidos por contêiner |

Os dois seguem a mesma esteira: **testes → Trivy no código → build `linux/arm64` → push no GHCR →
Trivy na imagem → commit da nova tag e do digest no overlay da aplicação no `tcc-platform`**. A esteira nunca
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
      overlay-path: apps/catalogo-api/hml   # a main publica em hml; prd é promoção no tcc-platform
      enable-deploy: ${{ github.event_name != 'pull_request' }}
    secrets:
      platform-token: ${{ secrets.PLATFORM_REPO_TOKEN }}
```

## Parâmetros

Comuns: `platform-repository`, `overlay-path`, `dockerfile`, `build-context`, `docker-build-args`,
`enable-trivy-filesystem-scan`, `enable-trivy-image-scan`, `enable-deploy`.

- Backend: `dotnet-version`, `test-command` (vazio pula os testes).
- Frontend: `node-version`, `working-directory`, `install-command`, `test-command`, `build-command`.

Imagem: `ghcr.io/<owner>/<repositório>:sha-<12 primeiros do commit>`. Tags do GHCR são mutáveis,
então a esteira grava no overlay **tag e digest** (`newTag` + `digest`); o Kustomize renderiza
`<imagem>:<tag>@sha256:…` e o nó baixa exatamente o conteúdo construído. A etapa que grava o overlay
recusa o commit se qualquer arquivo além de `<overlay-path>/kustomization.yaml` mudar.

## Política do Trivy

- **Código** (`fs`, vulnerabilidades e segredos) e **imagem**: falha só em `CRITICAL` **com correção
  disponível** (`ignore-unfixed: true`). Uma CVE crítica sem correção na imagem base não bloqueia a
  esteira indefinidamente; ela aparece no log do job e volta a ser avaliada a cada build.
- Achados `HIGH` e abaixo não bloqueiam.

## Build ARM64

Os runners são `ubuntu-latest` (x64). O Dockerfile das aplicações compila no estágio
`FROM --platform=$BUILDPLATFORM` (nativo, sem emulação) e publica para `$TARGETARCH`; a imagem final
arm64 não tem `RUN`, então o QEMU instalado pelo template quase não é exercitado. Assim o tempo de build
é próximo ao de um build x64 comum.

## Pré-requisitos

1. **Este repositório** (privado): *Settings → Actions → General → Access* →
   "Accessible from repositories owned by the user" — sem isso, os outros repositórios privados não
   conseguem chamar os templates.
2. **Em cada aplicação:** secret `PLATFORM_REPO_TOKEN` com um *fine-grained token* restrito ao
   repositório `tcc-platform`, permissão **Contents: Read and write**.
3. **No `tcc-platform`:** o overlay `apps/<app>/<amb>/kustomization.yaml` precisa existir, com o bloco
   `images:` contendo `name: ghcr.io/<owner>/<app>` e `newTag:`.
4. **Dockerfile** da aplicação gerando `linux/arm64` (a VM é Ampere/ARM64); compile com
   `FROM --platform=$BUILDPLATFORM` para não emular o build inteiro.

Regra: se uma aplicação precisar de algo muito fora do padrão, crie um terceiro template em vez de
encher os dois templates-base de exceções.
