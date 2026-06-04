# Manage Terraform State

## Visão Geral

O Terraform precisa armazenar o estado da infraestrutura gerenciada e de sua configuração. Esse estado é utilizado pelo Terraform para mapear recursos do mundo real à configuração declarada, rastrear metadados e melhorar o desempenho em infraestruturas de grande porte.

Por padrão, o estado é armazenado localmente em um arquivo chamado `terraform.tfstate`, mas também pode ser armazenado remotamente — o que funciona melhor em ambientes de equipe.

O Terraform usa esse estado local para criar planos e aplicar mudanças à infraestrutura. Antes de qualquer operação, o Terraform executa um refresh para atualizar o estado com a infraestrutura real.

O principal objetivo do estado do Terraform é armazenar vínculos entre objetos em um sistema remoto e instâncias de recursos declaradas na configuração. Quando o Terraform cria um objeto remoto em resposta a uma mudança de configuração, ele registra a identidade desse objeto contra uma instância de recurso específica e pode atualizar ou excluir esse objeto em resposta a mudanças futuras.

## Introdução

Neste lab, você aprende a gerenciar o estado do Terraform de diferentes formas. Serão abordados:

- A configuração de um **backend local** explícito para controlar onde o estado é salvo no filesystem.
- A migração do estado para um **backend remoto no Cloud Storage**, habilitando locking e colaboração em equipe.
- O uso de `terraform refresh` para reconciliar o estado com mudanças feitas fora do Terraform.
- A importação de infraestrutura existente (um contêiner Docker) para o controle do Terraform usando `terraform import`.
- O gerenciamento contínuo do recurso importado, incluindo alterações de configuração e destruição.

## Pré-requisitos e Variáveis

### Autenticação e configuração inicial

```bash
# Verificar conta ativa
gcloud auth list

# Verificar o projeto configurado
gcloud config list project
```

### Habilitar a API necessária (Gemini Code Assist — opcional para o lab)

```bash
gcloud services enable cloudaicompanion.googleapis.com
```

### Variáveis de ambiente utilizadas neste guia

Ajuste os valores conforme seu ambiente antes de executar os comandos:

```bash
export PROJECT_ID="SEU_PROJECT_ID"      # ID do projeto GCP
export REGION="us-central1"             # Região — ajuste conforme ambiente
export BUCKET_NAME="$PROJECT_ID"        # Nome do bucket (igual ao Project ID neste lab)
```

---

## TAREFA 1: Trabalhar com Backends

### Conceito

Um **backend** no Terraform define como o estado é carregado e como operações como `apply` são executadas. Por padrão, o Terraform usa o backend `local`, que salva o estado em um arquivo no diretório de trabalho atual.

Os principais benefícios de backends remotos são:
- Trabalho em equipe com locking de estado para evitar corrupção.
- Armazenamento do estado fora do disco local (maior segurança para dados sensíveis).
- Execução remota de operações de longa duração.

---

### Passo 1: Criar o arquivo de configuração principal

```bash
touch main.tf
```

**Explicação dos parâmetros:**
- `touch main.tf`: cria um arquivo vazio chamado `main.tf` no diretório atual.

**Resultado esperado:** arquivo `main.tf` criado no diretório de trabalho.

---

### Passo 2: Escrever a configuração do provider e do bucket no main.tf

```bash
cat > main.tf << 'EOF'
provider "google" {
  project = "SEU_PROJECT_ID"
  region  = "us-central1"
}

resource "google_storage_bucket" "test-bucket-for-state" {
  name                        = "SEU_PROJECT_ID"
  location                    = "US"
  uniform_bucket_level_access = true
}

terraform {
  backend "local" {
    path = "terraform/state/terraform.tfstate"
  }
}
EOF
```

**Explicação dos parâmetros:**
- `provider "google"`: configura o provider do Google Cloud com o projeto e a região.
- `resource "google_storage_bucket"`: declara um bucket do Cloud Storage.
  - `name`: nome único global do bucket (use o Project ID como recomendado pelo lab).
  - `location`: localização multirregional `US`.
  - `uniform_bucket_level_access`: habilita controle de acesso uniforme no bucket.
- `terraform { backend "local" }`: configura o backend local explícito.
  - `path`: caminho do arquivo de estado no filesystem local.

**Resultado esperado:** `main.tf` contendo provider, recurso de bucket e configuração de backend local.

---

### Passo 3: Inicializar o Terraform com o backend local

```bash
terraform init
```

**Explicação dos parâmetros:**
- `terraform init`: inicializa o ambiente Terraform, baixa os providers necessários e configura o backend declarado.

**Resultado esperado:** mensagem confirmando que o backend foi inicializado com sucesso e que os plugins de provider foram instalados.

---

### Passo 4: Aplicar a configuração para criar o bucket

```bash
terraform apply
```

Digite `yes` quando solicitado para confirmar.

**Explicação dos parâmetros:**
- `terraform apply`: gera um plano de execução e aplica as mudanças descritas na configuração.

**Resultado esperado:** bucket `SEU_PROJECT_ID` criado no Cloud Storage. O arquivo de estado é gravado em `terraform/state/terraform.tfstate`.

---

### Passo 5: Inspecionar o estado atual

```bash
terraform show
```

**Explicação dos parâmetros:**
- `terraform show`: exibe o estado atual ou um plano salvo de forma legível.

**Resultado esperado:** detalhes do recurso `google_storage_bucket.test-bucket-for-state` são exibidos, incluindo nome, localização e configurações.

---

### Passo 6: Migrar para o backend Cloud Storage (GCS)

Edite o bloco `terraform {}` em `main.tf`, substituindo o backend `local` pelo backend `gcs`:

```bash
cat > main.tf << 'EOF'
provider "google" {
  project = "SEU_PROJECT_ID"
  region  = "us-central1"
}

resource "google_storage_bucket" "test-bucket-for-state" {
  name                        = "SEU_PROJECT_ID"
  location                    = "US"
  uniform_bucket_level_access = true
}

terraform {
  backend "gcs" {
    bucket = "SEU_PROJECT_ID"
    prefix = "terraform/state"
  }
}
EOF
```

**Explicação dos parâmetros:**
- `backend "gcs"`: configura o backend remoto no Google Cloud Storage.
  - `bucket`: nome do bucket que armazenará o arquivo de estado.
  - `prefix`: prefixo (caminho) dentro do bucket onde o arquivo de estado será salvo.

**Resultado esperado:** `main.tf` atualizado com o backend GCS.

---

### Passo 7: Reinicializar com migração automática de estado

```bash
terraform init -migrate-state
```

Digite `yes` quando solicitado para confirmar a migração.

**Explicação dos parâmetros:**
- `-migrate-state`: instrui o Terraform a migrar automaticamente o estado existente para o novo backend configurado.

**Resultado esperado:** estado migrado do filesystem local para o objeto `terraform/state/default.tfstate` dentro do bucket no Cloud Storage.

---

### Passo 8: Atualizar o estado após mudança manual (terraform refresh)

O `terraform refresh` reconcilia o estado que o Terraform conhece com a infraestrutura real. Simula uma mudança manual adicionando um label ao bucket pelo Console e depois executa:

```bash
terraform refresh
```

```bash
terraform show
```

**Explicação dos parâmetros:**
- `terraform refresh`: consulta os providers e atualiza o arquivo de estado com o estado atual real dos recursos, sem modificar a infraestrutura.
- `terraform show`: exibe o estado atualizado após o refresh.

**Resultado esperado:** o par `"key" = "value"` adicionado manualmente ao bucket aparece no atributo `labels` exibido por `terraform show`.

---

### Passo 9: Reverter para backend local antes de destruir

```bash
cat > main.tf << 'EOF'
provider "google" {
  project = "SEU_PROJECT_ID"
  region  = "us-central1"
}

resource "google_storage_bucket" "test-bucket-for-state" {
  name                        = "SEU_PROJECT_ID"
  location                    = "US"
  uniform_bucket_level_access = true
  force_destroy               = true
}

terraform {
  backend "local" {
    path = "terraform/state/terraform.tfstate"
  }
}
EOF
```

```bash
terraform init -migrate-state
```

Digite `yes` quando solicitado.

**Explicação dos parâmetros:**
- `force_destroy = true`: permite que o Terraform exclua um bucket mesmo que ele contenha objetos. Necessário para que o `terraform destroy` não falhe.

**Resultado esperado:** estado migrado de volta para o filesystem local e configuração atualizada com `force_destroy`.

---

### Passo 10: Aplicar a mudança de force_destroy e destruir o bucket

```bash
terraform apply
```

Digite `yes` quando solicitado.

```bash
terraform destroy
```

Digite `yes` quando solicitado.

**Explicação dos parâmetros:**
- `terraform destroy`: destrói todos os recursos gerenciados pela configuração atual.

**Resultado esperado:** bucket do Cloud Storage destruído com sucesso.

---

## TAREFA 2: Importar uma Configuração Terraform

### Conceito

O `terraform import` permite trazer infraestrutura existente — criada fora do Terraform — para o controle do estado do Terraform. O processo envolve:

1. Identificar o recurso existente a ser importado.
2. Declarar um recurso vazio na configuração.
3. Executar `terraform import` para vincular o recurso ao estado.
4. Completar a configuração com os atributos corretos.
5. Aplicar para sincronizar estado e configuração.

> **Atenção:** Sempre faça backup do `terraform.tfstate` e do diretório `.terraform` antes de importar recursos em projetos Terraform reais.

---

### Passo 1: Criar o contêiner Docker a ser importado

```bash
docker run --name hashicorp-learn --detach --publish 8080:80 nginx:latest
```

**Explicação dos parâmetros:**
- `--name hashicorp-learn`: nomeia o contêiner.
- `--detach`: executa o contêiner em segundo plano.
- `--publish 8080:80`: mapeia a porta 8080 do host para a porta 80 do contêiner.
- `nginx:latest`: imagem Docker a ser usada.

```bash
docker ps
```

**Resultado esperado:** contêiner `hashicorp-learn` em execução com a porta `8080->80/tcp` mapeada.

---

### Passo 2: Clonar o repositório de exemplo

```bash
git clone https://github.com/hashicorp/learn-terraform-import.git
cd learn-terraform-import
```

**Resultado esperado:** diretório `learn-terraform-import` criado com os arquivos `main.tf`, `docker.tf` e `terraform.tf`.

---

### Passo 3: Atualizar a versão do provider no terraform.tf

```bash
sed -i 's/version = "~> 3.0.2"/version = ">= 3.5"/' terraform.tf
```

**Explicação dos parâmetros:**
- `sed -i`: substitui texto in-place no arquivo.
- Atualiza a restrição de versão do provider Docker para `>= 3.5`, evitando erros de compatibilidade.

**Resultado esperado:** `terraform.tf` com restrição de versão atualizada.

---

### Passo 4: Inicializar o workspace Terraform

```bash
terraform init --upgrade
```

**Explicação dos parâmetros:**
- `--upgrade`: força a atualização dos providers para a versão mais recente que satisfaça as restrições declaradas.

**Resultado esperado:** provider Docker instalado/atualizado com sucesso.

---

### Passo 5: Ajustar o provider Docker no main.tf (comentar o host)

Edite o arquivo `main.tf` dentro de `learn-terraform-import` para comentar o argumento `host`:

```bash
cat > main.tf << 'EOF'
provider "docker" {
#   host = "npipe:////.//pipe//docker_engine"
}
EOF
```

**Explicação dos parâmetros:**
- Comentar o `host` é um workaround para um erro conhecido de inicialização do Docker no Cloud Shell.

**Resultado esperado:** `main.tf` com o argumento `host` comentado.

---

### Passo 6: Declarar o recurso vazio no docker.tf

Adicione a declaração de recurso vazio ao `docker.tf`:

```bash
cat >> docker.tf << 'EOF'

resource "docker_container" "web" {}
EOF
```

**Explicação dos parâmetros:**
- `resource "docker_container" "web" {}`: declara um recurso vazio que servirá como âncora para a importação. O Terraform precisa que o recurso exista na configuração antes de aceitar o import.

**Resultado esperado:** `docker.tf` com o bloco de recurso vazio adicionado.

---

### Passo 7: Importar o contêiner Docker para o estado Terraform

```bash
terraform import docker_container.web $(docker inspect -f {{.ID}} hashicorp-learn)
```

**Explicação dos parâmetros:**
- `terraform import`: vincula um recurso real ao estado do Terraform.
- `docker_container.web`: ID do recurso Terraform que receberá o vínculo.
- `$(docker inspect -f {{.ID}} hashicorp-learn)`: obtém o SHA256 completo do contêiner para ser usado como ID de importação.

**Resultado esperado:** mensagem `Import successful!` confirmando que o contêiner foi importado para o estado.

---

### Passo 8: Verificar o estado importado

```bash
terraform show
```

**Resultado esperado:** estado completo do contêiner Docker exibido, incluindo imagem, portas, variáveis de rede e outros atributos gerenciados pelo provider.

---

### Passo 9: Gerar a configuração a partir do estado importado

```bash
terraform show -no-color > docker.tf
```

**Explicação dos parâmetros:**
- `terraform show -no-color`: exibe o estado sem códigos de cor ANSI.
- `> docker.tf`: redireciona a saída para o arquivo `docker.tf`, sobrescrevendo seu conteúdo com a configuração derivada do estado.

**Resultado esperado:** `docker.tf` contendo a configuração completa derivada do estado importado.

---

### Passo 10: Verificar o plano e corrigir atributos desnecessários

```bash
terraform plan
```

O plano pode mostrar avisos sobre atributos obsoletos (`links`) e atributos somente leitura (`ip_address`, `network_data`, `gateway`, `ip_prefix_length`, `id`). Edite `docker.tf` mantendo apenas os atributos essenciais:

```bash
cat > docker.tf << 'EOF'
resource "docker_container" "web" {
    image = "sha256:HASH_DA_IMAGEM_AQUI"
    name  = "hashicorp-learn"
    ports {
        external = 8080
        internal = 80
        ip       = "0.0.0.0"
        protocol = "tcp"
    }
}
EOF
```

> Substitua `sha256:HASH_DA_IMAGEM_AQUI` pelo hash real exibido em `terraform show`.

```bash
terraform plan
```

**Resultado esperado:** plano executado sem erros. O Terraform pode indicar adição de atributos como `attach`, `logs`, `must_run` e `start` (valores padrão que não afetam o contêiner em execução).

---

### Passo 11: Aplicar a configuração para sincronizar estado e contêiner

```bash
terraform apply
```

Digite `yes` quando solicitado.

**Resultado esperado:** estado, configuração e contêiner sincronizados. O contêiner continua em execução sem interrupções.

---

### Passo 12: Criar um recurso de imagem Docker separado

Obtenha a tag da imagem:

```bash
docker image inspect <IMAGE-ID> -f {{.RepoTags}}
```

Substitua `<IMAGE-ID>` pelo hash SHA256 exibido em `docker.tf`. Em seguida, adicione o recurso de imagem ao `docker.tf`:

```bash
cat >> docker.tf << 'EOF'

resource "docker_image" "nginx" {
  name = "nginx:latest"
}
EOF
```

```bash
terraform apply
```

Digite `yes` quando solicitado.

**Explicação dos parâmetros:**
- `resource "docker_image" "nginx"`: declara a imagem Docker como recurso Terraform, permitindo referenciá-la por nome em vez de hash.
- `name = "nginx:latest"`: tag da imagem a ser gerenciada.

**Resultado esperado:** recurso `docker_image.nginx` criado no estado Terraform.

---

### Passo 13: Referenciar a imagem pelo recurso no contêiner

Atualize `docker.tf` para referenciar a imagem pelo recurso em vez do hash:

```bash
cat > docker.tf << 'EOF'
resource "docker_container" "web" {
    image = docker_image.nginx.image_id
    name  = "hashicorp-learn"
    ports {
        external = 8080
        internal = 80
        ip       = "0.0.0.0"
        protocol = "tcp"
    }
}

resource "docker_image" "nginx" {
  name = "nginx:latest"
}
EOF
```

```bash
terraform apply
```

Digite `yes` quando solicitado.

**Explicação dos parâmetros:**
- `docker_image.nginx.image_id`: referência dinâmica ao ID da imagem gerenciada pelo recurso `docker_image.nginx`, eliminando o hash hardcoded.

**Resultado esperado:** plano sem mudanças (o hash da imagem referenciada é o mesmo já em uso). O contêiner continua em execução.

---

### Passo 14: Gerenciar o contêiner — alterar a porta externa

Atualize `docker.tf` mudando a porta externa de `8080` para `8081`:

```bash
cat > docker.tf << 'EOF'
resource "docker_container" "web" {
  image = docker_image.nginx.image_id
  name  = "hashicorp-learn"

  ports {
    external = 8081
    internal = 80
    ip       = "0.0.0.0"
    protocol = "tcp"
  }
}

resource "docker_image" "nginx" {
  name = "nginx:latest"
}
EOF
```

```bash
terraform apply
```

Digite `yes` quando solicitado.

**Resultado esperado:** Terraform destrói o contêiner antigo e cria um novo com a porta `8081` mapeada. O ID do contêiner muda.

---

### Passo 15: Verificar o novo contêiner

```bash
docker ps
```

**Resultado esperado:** contêiner `hashicorp-learn` em execução com a porta `8081->80/tcp`.

---

### Passo 16: Destruir a infraestrutura

```bash
terraform destroy
```

Digite `yes` quando solicitado.

```bash
docker ps --filter "name=hashicorp-learn"
```

**Resultado esperado:** nenhum contêiner `hashicorp-learn` listado. Contêiner e imagem foram removidos pelo Terraform.

---

## Validação

### Validar o estado do backend Cloud Storage

```bash
# Listar objetos no bucket para confirmar que o estado foi salvo remotamente
gsutil ls gs://SEU_PROJECT_ID/terraform/state/
```

**Resultado esperado:** arquivo `default.tfstate` listado no prefixo `terraform/state/`.

### Verificar o estado atual do Terraform

```bash
terraform show
```

### Listar contêineres Docker ativos

```bash
docker ps
```

### Verificar se o bucket foi destruído

```bash
gsutil ls | grep SEU_PROJECT_ID
```

**Resultado esperado:** nenhuma saída (bucket não existe mais).

---

## Troubleshooting

| Sintoma | Causa Provável | Solução |
|---|---|---|
| `Error: Failed to query available provider packages` | Versão de provider incompatível ou cache desatualizado | Execute `terraform init -upgrade` |
| `Error acquiring the state lock` | Outro processo mantém o lock do estado remoto | Verifique se há outro `terraform apply` em execução; use `terraform force-unlock <LOCK_ID>` com cautela |
| `Error: Failed to get existing workspaces` | Permissão insuficiente no bucket GCS | Conceda ao service account o papel `roles/storage.objectAdmin` no bucket |
| `Error deleting Cloud Storage Bucket` | Bucket contém objetos e `force_destroy = false` | Adicione `force_destroy = true` ao recurso e execute `terraform apply` antes do destroy |
| `docker: Error response from daemon` ao criar contêiner | Docker daemon não está em execução no Cloud Shell | Reinicie o Cloud Shell ou execute `sudo service docker start` |
| `terraform import` retorna `Error: resource address must be a resource` | Formato incorreto do endereço do recurso no comando import | Use o formato `tipo.nome`, por exemplo `docker_container.web` |
| Plano mostra destruição e recriação inesperada após import | Atributos somente leitura ou obsoletos presentes no `docker.tf` gerado | Remova atributos como `id`, `ip_address`, `network_data`, `gateway` e `links` do arquivo |
| `Error: Backend configuration changed` sem `-migrate-state` | Backend foi alterado sem reinicialização com migração | Execute `terraform init -migrate-state` |

---

## Limpeza (Opcional)

Caso queira remover todos os recursos criados neste lab:

```bash
# Na pasta do lab principal (backend local restaurado)
terraform destroy

# Na pasta learn-terraform-import
cd learn-terraform-import
terraform destroy

# Remover imagens Docker locais
docker image prune -f
```

---

## Conceitos-Chave

| Conceito | Descrição |
|---|---|
| **Terraform State** | Arquivo (`.tfstate`) que mapeia recursos declarados na configuração para objetos reais na infraestrutura |
| **Backend Local** | Armazena o estado em um arquivo no filesystem local; padrão do Terraform |
| **Backend GCS** | Armazena o estado como objeto em um bucket do Cloud Storage; suporta locking e uso em equipe |
| **State Locking** | Mecanismo que impede que múltiplos usuários executem operações simultâneas que poderiam corromper o estado |
| **terraform init** | Inicializa o ambiente Terraform, configura o backend e baixa providers |
| **terraform init -migrate-state** | Inicializa e migra o estado existente para o novo backend configurado |
| **terraform refresh** | Reconcilia o estado conhecido com o estado real da infraestrutura sem modificá-la |
| **terraform import** | Importa um recurso existente (criado fora do Terraform) para o estado do Terraform |
| **terraform show** | Exibe o estado atual ou um plano salvo de forma legível |
| **terraform destroy** | Destrói todos os recursos gerenciados pela configuração atual |
| **force_destroy** | Atributo do `google_storage_bucket` que permite excluir o bucket mesmo que contenha objetos |
| **Workspace** | Instância isolada de estado associada a um backend; o workspace padrão é chamado `default` |
| **docker_image** | Recurso Terraform do provider Docker que gerencia imagens Docker como infraestrutura |
| **docker_container** | Recurso Terraform do provider Docker que gerencia contêineres Docker como infraestrutura |

---

## Fluxo Final

```
TAREFA 1 — Backends e Estado
═══════════════════════════════════════════════════════════════

  main.tf (provider + bucket + backend local)
        │
        ▼
  terraform init          ──► inicializa backend local
        │
        ▼
  terraform apply         ──► cria google_storage_bucket
        │
        ▼
  terraform show          ──► inspeciona estado local
        │
        ▼
  main.tf atualizado      ──► backend "gcs" configurado
        │
        ▼
  terraform init          ──► migra estado local ──► GCS bucket
    -migrate-state              terraform/state/default.tfstate
        │
        ▼
  [label adicionado        ──► terraform refresh ──► estado atualizado
   manualmente no Console]
        │
        ▼
  main.tf restaurado      ──► backend local + force_destroy = true
        │
        ▼
  terraform init          ──► migra estado GCS ──► local
    -migrate-state
        │
        ▼
  terraform destroy       ──► bucket destruído


TAREFA 2 — Import de Configuração Terraform
═══════════════════════════════════════════════════════════════

  docker run nginx        ──► contêiner hashicorp-learn (porta 8080)
        │
        ▼
  git clone repo          ──► learn-terraform-import/
        │
        ▼
  terraform init          ──► provider Docker instalado
  (--upgrade)
        │
        ▼
  docker.tf               ──► resource "docker_container" "web" {}
  (recurso vazio)
        │
        ▼
  terraform import        ──► vincula contêiner ao estado Terraform
  docker_container.web
        │
        ▼
  terraform show          ──► estado completo exibido
  > docker.tf             ──► configuração gerada a partir do estado
        │
        ▼
  editar docker.tf        ──► manter apenas: image, name, ports
        │
        ▼
  terraform apply         ──► estado e configuração sincronizados
        │
        ▼
  docker_image.nginx      ──► recurso de imagem adicionado ao docker.tf
  terraform apply
        │
        ▼
  image = docker_image    ──► referência dinâmica substituindo hash
  .nginx.image_id
  terraform apply
        │
        ▼
  porta 8080 ──► 8081     ──► terraform apply ──► contêiner recriado
        │
        ▼
  terraform destroy       ──► contêiner e imagem destruídos
```
