# Interact with Terraform Modules

## Visão Geral

À medida que você gerencia sua infraestrutura com Terraform, configurações cada vez mais complexas são criadas. Não há limite intrínseco para a complexidade de um único arquivo ou diretório de configuração Terraform, mas ao continuar escrevendo tudo em um único lugar, você pode enfrentar os seguintes problemas:

- Navegar e entender os arquivos de configuração se torna cada vez mais difícil.
- Atualizar a configuração se torna mais arriscado, pois uma mudança em um bloco pode causar consequências indesejadas em outros blocos.
- A duplicação de blocos similares aumenta (ex.: ambientes dev/staging/produção separados), gerando sobrecarga ao atualizar essas partes.
- Compartilhar partes da configuração entre projetos e equipes via cópia manual é propenso a erros e difícil de manter.

Neste lab, você aprende como módulos resolvem esses problemas, a estrutura de um módulo Terraform e as boas práticas ao usar e criar módulos.

## Introdução

Este lab aborda dois fluxos principais: o uso de um módulo público do Terraform Registry para provisionar uma rede VPC no Google Cloud, e a criação de um módulo local para gerenciar buckets do Cloud Storage configurados para hospedagem de sites estáticos. Ao final, você terá praticado o ciclo completo de consumo e autoria de módulos Terraform.

## Pré-requisitos e Variáveis

### Autenticação e configuração inicial

```bash
# Verificar conta ativa
gcloud auth list

# Verificar projeto configurado
gcloud config list project
```

### Variáveis de ambiente utilizadas neste guia

Defina as variáveis abaixo antes de executar os comandos. Ajuste conforme seu ambiente.

```bash
export PROJECT_ID="SEU_PROJECT_ID"   # ajuste conforme ambiente
export REGION="SUA_REGIAO"           # ex.: us-east1, us-central1
```

---

## TAREFA 1: Usar módulos do Registry

### Conceito

O Terraform Registry disponibiliza módulos públicos e verificados que encapsulam padrões de infraestrutura reutilizáveis. Ao referenciar um módulo do Registry, você especifica o caminho `<namespace>/<nome>/<provider>` no argumento `source` e opcionalmente o argumento `version` para fixar uma versão. Todos os demais argumentos no bloco `module` são tratados como variáveis de entrada do módulo.

### Passo 1: Clonar o repositório de exemplo

```bash
git clone https://github.com/terraform-google-modules/terraform-google-network
cd terraform-google-network
git checkout tags/v6.0.1 -b v6.0.1
```

**Explicação dos parâmetros:**
- `git clone`: faz o download do repositório remoto para o diretório local.
- `git checkout tags/v6.0.1 -b v6.0.1`: cria uma branch local `v6.0.1` apontando para a tag `v6.0.1`, garantindo que a versão correta do módulo de exemplo seja usada.

**Resultado esperado:** repositório clonado e branch `v6.0.1` ativa.

### Passo 2: Habilitar a API do Gemini for Google Cloud (opcional)

```bash
gcloud services enable cloudaicompanion.googleapis.com
```

**Explicação dos parâmetros:**
- `services enable`: ativa uma API no projeto atual.
- `cloudaicompanion.googleapis.com`: identificador da API do Gemini Code Assist.

**Resultado esperado:** API habilitada no projeto, permitindo uso do Gemini Code Assist no Cloud Shell Editor.

### Passo 3: Editar `variables.tf` — adicionar valor padrão para `project_id`

Navegue até `terraform-google-network/examples/simple_project/` e edite o arquivo `variables.tf` para adicionar o argumento `default` à variável `project_id`:

```hcl
variable "project_id" {
  description = "The project ID to host the network in"
  default     = "SEU_PROJECT_ID"   # ajuste conforme ambiente
}
```

### Passo 4: Editar `variables.tf` — definir variável `network_name`

No mesmo arquivo `variables.tf`, adicione o bloco abaixo:

```hcl
variable "network_name" {
  description = "The name of the network to be created"
  default     = "example-vpc"
}
```

**Explicação dos parâmetros:**
- `description`: texto descritivo exibido em documentação e erros.
- `default`: valor usado quando a variável não for fornecida explicitamente na chamada do módulo.

### Passo 5: Editar `main.tf` — atualizar o módulo de VPC

Edite `terraform-google-network/examples/simple_project/main.tf` para usar `var.network_name` e substituir `us-west1` pela sua região:

```hcl
module "test-vpc-module" {
  source       = "terraform-google-modules/network/google"
  version      = "~> 6.0"
  project_id   = var.project_id
  network_name = var.network_name
  mtu          = 1460

  subnets = [
    {
      subnet_name   = "subnet-01"
      subnet_ip     = "10.10.10.0/24"
      subnet_region = "SUA_REGIAO"   # ajuste conforme ambiente
    },
    {
      subnet_name           = "subnet-02"
      subnet_ip             = "10.10.20.0/24"
      subnet_region         = "SUA_REGIAO"
      subnet_private_access = "true"
      subnet_flow_logs      = "true"
    },
    {
      subnet_name               = "subnet-03"
      subnet_ip                 = "10.10.30.0/24"
      subnet_region             = "SUA_REGIAO"
      subnet_flow_logs          = "true"
      subnet_flow_logs_interval = "INTERVAL_10_MIN"
      subnet_flow_logs_sampling = 0.7
      subnet_flow_logs_metadata = "INCLUDE_ALL_METADATA"
      subnet_flow_logs_filter   = "false"
    }
  ]
}
```

**Explicação dos parâmetros:**
- `source`: endereço do módulo no Terraform Registry no formato `<namespace>/<module>/<provider>`.
- `version`: restrição de versão do módulo; `~> 6.0` aceita qualquer `6.x` mas não `7.x`.
- `project_id`: ID do projeto GCP onde a VPC será criada.
- `network_name`: nome da rede VPC a ser criada.
- `mtu`: Maximum Transmission Unit da rede em bytes (1460 é o padrão recomendado para GCP).
- `subnets`: lista de sub-redes a serem criadas; cada objeto define nome, CIDR e região.
- `subnet_private_access`: habilita acesso privado ao Google APIs sem IP público.
- `subnet_flow_logs`: habilita VPC Flow Logs para a sub-rede.
- `subnet_flow_logs_interval`: intervalo de agregação dos flow logs.
- `subnet_flow_logs_sampling`: taxa de amostragem dos flow logs (0.0 a 1.0).
- `subnet_flow_logs_metadata`: metadados incluídos nos registros de flow log.

### Passo 6: Verificar `outputs.tf` do projeto de exemplo

O arquivo `terraform-google-network/examples/simple_project/outputs.tf` deve conter as saídas mapeadas a partir dos outputs do módulo filho:

```hcl
output "network_name" {
  value       = module.test-vpc-module.network_name
  description = "The name of the VPC being created"
}

output "network_self_link" {
  value       = module.test-vpc-module.network_self_link
  description = "The URI of the VPC being created"
}

output "project_id" {
  value       = module.test-vpc-module.project_id
  description = "VPC project id"
}

output "subnets_names" {
  value       = module.test-vpc-module.subnets_names
  description = "The names of the subnets being created"
}

output "subnets_ips" {
  value       = module.test-vpc-module.subnets_ips
  description = "The IP and cidrs of the subnets being created"
}

output "subnets_regions" {
  value       = module.test-vpc-module.subnets_regions
  description = "The region where subnets will be created"
}

output "subnets_private_access" {
  value       = module.test-vpc-module.subnets_private_access
  description = "Whether the subnets will have access to Google API's without a public IP"
}

output "subnets_flow_logs" {
  value       = module.test-vpc-module.subnets_flow_logs
  description = "Whether the subnets will have VPC flow logs enabled"
}

output "subnets_secondary_ranges" {
  value       = module.test-vpc-module.subnets_secondary_ranges
  description = "The secondary ranges associated with these subnets"
}

output "route_names" {
  value       = module.test-vpc-module.route_names
  description = "The routes associated with this VPC"
}
```

**Explicação:** outputs de módulo são acessados via `module.<NOME_DO_MODULO>.<NOME_DO_OUTPUT>`. Eles expõem atributos do módulo filho para o módulo raiz.

### Passo 7: Provisionar a infraestrutura (Tarefa 1)

```bash
cd ~/terraform-google-network/examples/simple_project
terraform init
terraform apply
```

**Explicação dos comandos:**
- `terraform init`: baixa e instala o módulo referenciado em `.terraform/modules/`. Deve ser executado sempre que um novo módulo é adicionado.
- `terraform apply`: cria o plano de execução e aguarda confirmação antes de provisionar os recursos.

Quando solicitado, responda `yes`.

**Resultado esperado:**

```
Outputs:
network_name = "example-vpc"
network_self_link = "https://www.googleapis.com/compute/v1/projects/SEU_PROJECT_ID/global/networks/example-vpc"
project_id = "SEU_PROJECT_ID"
route_names = []
subnets_flow_logs = [false, true, true]
subnets_ips = ["10.10.10.0/24", "10.10.20.0/24", "10.10.30.0/24"]
subnets_names = ["subnet-01", "subnet-02", "subnet-03"]
```

### Passo 8: Destruir e limpar infraestrutura da Tarefa 1

```bash
terraform destroy
cd ~
rm -rd terraform-google-network -f
```

**Explicação dos parâmetros:**
- `terraform destroy`: reverte todos os recursos criados pelo Terraform neste diretório; aguarda confirmação `yes`.
- `rm -rd terraform-google-network -f`: remove o diretório clonado e todo o seu conteúdo de forma forçada e recursiva.

---

## TAREFA 2: Construir um módulo local

### Conceito

Além de consumir módulos públicos, é uma boa prática criar módulos locais para encapsular recursos reutilizáveis dentro de um projeto. Um módulo local é referenciado pelo caminho relativo do diretório. Terraform trata qualquer diretório com arquivos `.tf` como um módulo. O módulo chamado é referenciado como "módulo filho" e o diretório a partir do qual `terraform apply` é executado é o "módulo raiz".

### Estrutura do módulo
Terraform trata qualquer diretório local referenciado no argumento source de um bloco de módulo como um módulo. Uma estrutura de arquivo típica para um novo módulo é:

```
├── LICENSE
├── README.md
├── main.tf
├── variables.tf
├── outputs.tf
```

### Passo 1: Criar a estrutura de diretórios do módulo

```bash
cd ~
touch main.tf
mkdir -p modules/gcs-static-website-bucket
```

**Explicação dos parâmetros:**
- `touch main.tf`: cria o arquivo `main.tf` do módulo raiz no diretório home.
- `mkdir -p modules/gcs-static-website-bucket`: cria o diretório do módulo filho, incluindo o diretório `modules` intermediário caso não exista.

**Resultado esperado:** diretório `~/modules/gcs-static-website-bucket/` criado.

### Passo 2: Criar os arquivos base do módulo filho

```bash
cd modules/gcs-static-website-bucket
touch website.tf variables.tf outputs.tf
```

**Resultado esperado:** três arquivos vazios criados dentro do diretório do módulo.

### Passo 3: Criar o `README.md` do módulo

```bash
tee -a README.md <<EOF
# GCS static website bucket

This module provisions Cloud Storage buckets configured for static website hosting.
EOF
```

**Explicação:** `tee -a` grava o conteúdo no arquivo e também exibe no terminal. O bloco `<<EOF` é um heredoc que delimita o conteúdo a ser gravado.

### Passo 4: Criar o arquivo `LICENSE` do módulo

```bash
tee -a LICENSE <<EOF
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
EOF
```

**Nota:** `README.md` e `LICENSE` não são usados pelo Terraform, mas são boas práticas para módulos que serão compartilhados.

### Passo 5: Definir o recurso no `website.tf`

Conteúdo do arquivo `~/modules/gcs-static-website-bucket/website.tf`:

```hcl
resource "google_storage_bucket" "bucket" {
  name               = var.name
  project            = var.project_id
  location           = var.location
  storage_class      = var.storage_class
  labels             = var.labels
  force_destroy      = var.force_destroy
  uniform_bucket_level_access = true

  versioning {
    enabled = var.versioning
  }

  dynamic "retention_policy" {
    for_each = var.retention_policy == null ? [] : [var.retention_policy]
    content {
      is_locked        = var.retention_policy.is_locked
      retention_period = var.retention_policy.retention_period
    }
  }

  dynamic "encryption" {
    for_each = var.encryption == null ? [] : [var.encryption]
    content {
      default_kms_key_name = var.encryption.default_kms_key_name
    }
  }

  dynamic "lifecycle_rule" {
    for_each = var.lifecycle_rules
    content {
      action {
        type          = lifecycle_rule.value.action.type
        storage_class = lookup(lifecycle_rule.value.action, "storage_class", null)
      }
      condition {
        age                   = lookup(lifecycle_rule.value.condition, "age", null)
        created_before        = lookup(lifecycle_rule.value.condition, "created_before", null)
        with_state            = lookup(lifecycle_rule.value.condition, "with_state", null)
        matches_storage_class = lookup(lifecycle_rule.value.condition, "matches_storage_class", null)
        num_newer_versions    = lookup(lifecycle_rule.value.condition, "num_newer_versions", null)
      }
    }
  }
}
```

**Explicação dos parâmetros principais:**
- `name`: nome globalmente único do bucket; obrigatório.
- `project`: ID do projeto GCP onde o bucket será criado.
- `location`: região ou multi-região do bucket (ex.: `US`, `us-east1`).
- `storage_class`: classe de armazenamento (`STANDARD`, `NEARLINE`, `COLDLINE`, `ARCHIVE`).
- `uniform_bucket_level_access`: quando `true`, desabilita ACLs por objeto e exige IAM para controle de acesso.
- `force_destroy`: quando `true`, permite destruir o bucket mesmo com objetos dentro.
- `versioning.enabled`: quando `true`, mantém versões anteriores dos objetos.
- `dynamic "retention_policy"`: bloco condicional; cria a política de retenção apenas se `var.retention_policy` não for `null`.
- `dynamic "lifecycle_rule"`: itera sobre a lista `var.lifecycle_rules` para criar regras de ciclo de vida (ex.: deleção automática após X dias).
- `lookup(map, key, default)`: função Terraform que retorna o valor de uma chave em um mapa, ou o valor padrão se a chave não existir.

### Passo 6: Definir as variáveis do módulo em `variables.tf`

Conteúdo do arquivo `~/modules/gcs-static-website-bucket/variables.tf`:

```hcl
variable "name" {
  description = "The name of the bucket."
  type        = string
}

variable "project_id" {
  description = "The ID of the project to create the bucket in."
  type        = string
}

variable "location" {
  description = "The location of the bucket."
  type        = string
}

variable "storage_class" {
  description = "The Storage Class of the new bucket."
  type        = string
  default     = null
}

variable "labels" {
  description = "A set of key/value label pairs to assign to the bucket."
  type        = map(string)
  default     = null
}

variable "bucket_policy_only" {
  description = "Enables Bucket Policy Only access to a bucket."
  type        = bool
  default     = true
}

variable "versioning" {
  description = "While set to true, versioning is fully enabled for this bucket."
  type        = bool
  default     = true
}

variable "force_destroy" {
  description = "When deleting a bucket, this boolean option will delete all contained objects. If false, Terraform will fail to delete buckets which contain objects."
  type        = bool
  default     = true
}

variable "iam_members" {
  description = "The list of IAM members to grant permissions on the bucket."
  type = list(object({
    role   = string
    member = string
  }))
  default = []
}

variable "retention_policy" {
  description = "Configuration of the bucket's data retention policy for how long objects in the bucket should be retained."
  type = object({
    is_locked        = bool
    retention_period = number
  })
  default = null
}

variable "encryption" {
  description = "A Cloud KMS key that will be used to encrypt objects inserted into this bucket"
  type = object({
    default_kms_key_name = string
  })
  default = null
}

variable "lifecycle_rules" {
  description = "The bucket's Lifecycle Rules configuration."
  type = list(object({
    action    = any
    condition = any
  }))
  default = []
}
```

**Explicação:** variáveis sem `default` são obrigatórias ao chamar o módulo. Variáveis com `default = null` são opcionais e permitem blocos dinâmicos condicionais no recurso.

### Passo 7: Definir os outputs do módulo em `outputs.tf`

Conteúdo do arquivo `~/modules/gcs-static-website-bucket/outputs.tf`:

```hcl
output "bucket" {
  description = "The created storage bucket"
  value       = google_storage_bucket.bucket
}
```

**Explicação:** o output `bucket` expõe o objeto completo do recurso `google_storage_bucket.bucket`, tornando todos os seus atributos acessíveis no módulo raiz via `module.gcs-static-website-bucket.bucket`.

### Passo 8: Configurar o módulo raiz — `main.tf`

Edite o arquivo `~/main.tf` com o seguinte conteúdo:

```hcl
module "gcs-static-website-bucket" {
  source = "./modules/gcs-static-website-bucket"

  name       = var.name
  project_id = var.project_id
  location   = "SUA_REGIAO"   # ajuste conforme ambiente

  lifecycle_rules = [{
    action = {
      type = "Delete"
    }
    condition = {
      age        = 365
      with_state = "ANY"
    }
  }]
}
```

**Explicação dos parâmetros:**
- `source = "./modules/gcs-static-website-bucket"`: caminho relativo para o módulo local. O Terraform cria um symlink para este diretório em `.terraform/modules/`.
- `name`: nome do bucket, repassado à variável `var.name` do módulo filho.
- `project_id`: ID do projeto, repassado à variável `var.project_id` do módulo filho.
- `location`: localização do bucket; deve ser uma região ou multi-região válida do GCP.
- `lifecycle_rules`: lista de regras de ciclo de vida; neste caso, deleta objetos com mais de 365 dias independente do estado.

### Passo 9: Configurar outputs do módulo raiz — `outputs.tf`

```bash
cd ~
touch outputs.tf
```

Conteúdo do arquivo `~/outputs.tf`:

```hcl
output "bucket-name" {
  description = "Bucket names."
  value       = "module.gcs-static-website-bucket.bucket"
}
```

### Passo 10: Configurar variáveis do módulo raiz — `variables.tf`

```bash
touch variables.tf
```

Conteúdo do arquivo `~/variables.tf`:

```hcl
variable "project_id" {
  description = "The ID of the project in which to provision resources."
  type        = string
  default     = "SEU_PROJECT_ID"   # ajuste conforme ambiente
}

variable "name" {
  description = "Name of the buckets to create."
  type        = string
  default     = "SEU_NOME_BUCKET_UNICO"   # o nome do bucket deve ser globalmente único
}
```

**Nota:** o nome do bucket deve ser globalmente único no GCP. Uma boa estratégia é usar o ID do projeto, o nome do usuário ou a data atual como parte do nome.

### Passo 11: Inicializar e aplicar o módulo local

```bash
cd ~
terraform init
terraform apply
```

Quando solicitado, responda `yes`.

**Explicação:**
- `terraform init`: instala o módulo local criando um symlink em `.terraform/modules/gcs-static-website-bucket`.
- `terraform apply`: cria o bucket com as configurações definidas no módulo.

**Resultado esperado:** bucket do Cloud Storage criado com versioning habilitado e regra de ciclo de vida configurada.

### Passo 12: Fazer upload de arquivos para o bucket

```bash
cd ~
curl https://raw.githubusercontent.com/hashicorp/learn-terraform-modules/master/modules/aws-s3-static-website-bucket/www/index.html > index.html
curl https://raw.githubusercontent.com/hashicorp/learn-terraform-modules/blob/master/modules/aws-s3-static-website-bucket/www/error.html > error.html

gsutil cp *.html gs://SEU_NOME_BUCKET_UNICO   # ajuste conforme ambiente
```

**Explicação dos parâmetros:**
- `curl <url> > arquivo`: faz download do conteúdo da URL e salva no arquivo local.
- `gsutil cp *.html gs://NOME_BUCKET`: copia todos os arquivos `.html` do diretório atual para o bucket especificado.

**Resultado esperado:** arquivos `index.html` e `error.html` disponíveis no bucket. Acesse `https://storage.cloud.google.com/SEU_NOME_BUCKET/index.html` para verificar.

---

## Validação

### Verificar a VPC criada na Tarefa 1 (antes de destruir)

```bash
# Listar redes VPC no projeto
gcloud compute networks list --project=$PROJECT_ID

# Descrever a VPC criada
gcloud compute networks describe example-vpc --project=$PROJECT_ID

# Listar sub-redes criadas
gcloud compute networks subnets list \
  --filter="network:example-vpc" \
  --project=$PROJECT_ID
```

### Verificar o bucket criado na Tarefa 2

```bash
# Listar buckets do projeto
gsutil ls -p $PROJECT_ID

# Listar objetos dentro do bucket
gsutil ls gs://SEU_NOME_BUCKET_UNICO

# Verificar metadados do bucket
gsutil ls -L -b gs://SEU_NOME_BUCKET_UNICO
```

### Verificar outputs do Terraform

```bash
# Exibir todos os outputs do estado atual
terraform output

# Exibir um output específico
terraform output bucket-name
```

### Verificar estado do Terraform

```bash
# Listar recursos gerenciados pelo Terraform
terraform state list

# Inspecionar um recurso específico
terraform state show module.gcs-static-website-bucket.google_storage_bucket.bucket
```

---

## Troubleshooting

| Sintoma | Causa Provável | Solução |
|---|---|---|
| `Error: Module not installed` ao rodar `terraform apply` | Módulo não instalado após adição ao `main.tf` | Execute `terraform init` antes de `terraform apply` |
| `Error: Invalid module source address` | Caminho local incorreto no argumento `source` | Verifique se o diretório referenciado existe e o caminho relativo está correto |
| `Error: googleapi: Error 409: The requested bucket name is not available` | Nome do bucket já existe globalmente no GCP | Escolha um nome único; use o ID do projeto mais data ou sufixo aleatório |
| `Error: Required variable not set: project_id` | Variável `project_id` sem valor padrão e não fornecida | Adicione `default` na variável ou passe via `-var="project_id=SEU_PROJECT_ID"` |
| `Error: Error creating Network: googleapi: Error 403` | API Compute Engine não habilitada ou permissões insuficientes | Habilite a API: `gcloud services enable compute.googleapis.com` |
| `Error: Error creating Storage Bucket: googleapi: Error 403` | API Cloud Storage não habilitada ou permissões insuficientes | Habilite a API: `gcloud services enable storage.googleapis.com` |
| `terraform destroy` falha com objetos no bucket | `force_destroy = false` por padrão em alguns cenários | Certifique-se de que `force_destroy = true` está definido no módulo ou remova os objetos manualmente com `gsutil rm -r gs://BUCKET` |
| `Error: Incompatible provider version` | Versão do provider incompatível com o módulo | Execute `terraform init -upgrade` para atualizar os providers |

---

## Limpeza (Opcional)

### Destruir recursos da Tarefa 1

```bash
cd ~/terraform-google-network/examples/simple_project
terraform destroy
# Responda: yes

cd ~
rm -rd terraform-google-network -f
```

### Destruir recursos da Tarefa 2

```bash
cd ~
terraform destroy
# Responda: yes
```

**Resultado esperado:** todos os recursos criados pelo Terraform são destruídos. O estado local é atualizado para refletir a ausência dos recursos.

---

## Conceitos-Chave

| Conceito | Descrição |
|---|---|
| Módulo Terraform | Conjunto de arquivos `.tf` em um único diretório; toda configuração Terraform é um módulo |
| Módulo raiz | Diretório a partir do qual os comandos Terraform são executados diretamente |
| Módulo filho | Módulo chamado por outro módulo via bloco `module {}` |
| Módulo local | Módulo referenciado por caminho relativo no sistema de arquivos |
| Módulo remoto | Módulo carregado do Terraform Registry, GitHub, HTTP ou registros privados |
| `source` | Argumento obrigatório no bloco `module` que define a origem do módulo |
| `version` | Restrição de versão do módulo; suportada para fontes remotas; `~> 6.0` aceita `6.x` |
| `terraform init` | Instala módulos e providers; cria `.terraform/modules/` e `.terraform/providers/` |
| `terraform get` | Instala apenas módulos, sem inicializar backends ou instalar plugins |
| Variável de entrada de módulo | Argumento passado ao bloco `module {}` que configura o comportamento do módulo filho |
| Output de módulo | Valor exposto pelo módulo filho, acessado via `module.<NOME>.<OUTPUT>` |
| `dynamic` block | Bloco Terraform que gera sub-blocos condicionalmente a partir de uma coleção |
| `uniform_bucket_level_access` | Política de acesso uniforme no bucket; desabilita ACLs por objeto, exigindo IAM |
| Regra de ciclo de vida | Política automática para transição ou exclusão de objetos no Cloud Storage baseada em condições |
| `force_destroy` | Permite que o Terraform destrua um bucket mesmo que ele contenha objetos |

---

## Fluxo Final

```
TAREFA 1 — Módulo do Registry
==============================

  git clone (repo de exemplo)
          |
          v
  Editar variables.tf
  (project_id default, network_name)
          |
          v
  Editar main.tf
  (var.network_name, REGION nas subnets)
          |
          v
  terraform init
  (baixa módulo terraform-google-modules/network/google v6.x)
          |
          v
  terraform apply
          |
          v
  VPC "example-vpc" + 3 subnets criadas
          |
          v
  terraform destroy + rm -rd terraform-google-network


TAREFA 2 — Módulo Local
========================

  ~/main.tf (módulo raiz)
  ~/variables.tf
  ~/outputs.tf
          |
          v
  ~/modules/gcs-static-website-bucket/
    ├── website.tf     (recurso google_storage_bucket)
    ├── variables.tf   (name, project_id, location, ...)
    ├── outputs.tf     (output "bucket")
    ├── README.md
    └── LICENSE
          |
          v
  terraform init
  (symlink para ./modules/gcs-static-website-bucket)
          |
          v
  terraform apply
          |
          v
  Cloud Storage Bucket criado
  (versioning=true, lifecycle_rule: Delete após 365 dias)
          |
          v
  gsutil cp *.html gs://BUCKET_NAME
          |
          v
  Site estático disponível em:
  https://storage.cloud.google.com/BUCKET_NAME/index.html
          |
          v
  terraform destroy
```
