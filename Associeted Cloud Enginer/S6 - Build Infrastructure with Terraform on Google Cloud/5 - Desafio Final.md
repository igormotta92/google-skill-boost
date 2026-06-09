# Build Infrastructure with Terraform on Google Cloud: Desafio Final

## Visão Geral

Em um lab de desafio você recebe um cenário e um conjunto de tarefas. Em vez de seguir instruções passo a passo, você usará as habilidades aprendidas nos labs do curso para descobrir como concluir as tarefas por conta própria. Um sistema de pontuação automatizado fornecerá feedback sobre se você completou as tarefas corretamente.

Tópicos avaliados:
- Importar infraestrutura existente para a configuração do Terraform.
- Construir e referenciar seus próprios módulos Terraform.
- Adicionar um backend remoto à sua configuração.
- Usar e implementar um módulo do Terraform Registry.
- Reprovisionar, destruir e atualizar infraestrutura.
- Testar a conectividade entre os recursos criados.

## Introdução

Neste desafio, você atua como engenheiro de nuvem em uma startup. A missão é criar infraestrutura de forma rápida e eficiente usando Terraform, além de gerar um mecanismo de rastreamento para referência e mudanças futuras.

O escopo técnico inclui:
- Importar duas VMs pré-existentes (`tf-instance-1` e `tf-instance-2`) para módulos Terraform.
- Criar um bucket do Cloud Storage como backend remoto.
- Adicionar uma terceira instância, depois destruí-la.
- Criar uma VPC com duas sub-redes via módulo do Terraform Registry.
- Conectar as instâncias às sub-redes e criar uma regra de firewall.

## Pré-requisitos e Variáveis

Antes de começar, defina as variáveis de ambiente no Cloud Shell para facilitar os comandos. Ajuste os valores conforme o seu ambiente de lab.

```bash
# Ajuste conforme ambiente
export PROJECT_ID="qwiklabs-gcp-04-349113cfcef6"
export REGION="us-west4"
export ZONE="us-west4-a"
export BUCKET_NAME="tf-bucket-630475"
export VPC_NAME="tf-vpc-637584"
export INSTANCE_3_NAME="tf-instance-232342"
```

> **Atenção:** Os valores de `BUCKET_NAME`, `VPC_NAME` e `INSTANCE_3_NAME` são fornecidos pelo sistema do lab no momento do início. Consulte o painel lateral do lab para obtê-los.

**APIs necessárias** (geralmente já habilitadas no ambiente de lab):

```bash
gcloud services enable compute.googleapis.com \
    storage.googleapis.com \
    --project=$PROJECT_ID
```

---

## TAREFA 1: Criar os Arquivos de Configuração

### Conceito

O Terraform organiza a infraestrutura em arquivos `.tf`. A separação em módulos (`modules/`) promove reutilização e manutenibilidade. O arquivo `main.tf` na raiz é o ponto de entrada; os `variables.tf` definem parâmetros reutilizáveis; os `outputs.tf` expõem valores dos recursos para uso externo.

### Passo 1: Criar a estrutura de diretórios

```bash
mkdir -p modules/instances modules/storage

touch main.tf variables.tf
touch modules/instances/instances.tf \
      modules/instances/outputs.tf \
      modules/instances/variables.tf
touch modules/storage/storage.tf \
      modules/storage/outputs.tf \
      modules/storage/variables.tf
```

**Explicação dos parâmetros:**
- `mkdir -p`: cria diretórios de forma recursiva, sem errar se já existirem.
- `touch`: cria arquivos vazios que serão preenchidos nas próximas etapas.

**Resultado esperado:** Estrutura de pastas e arquivos criada sem erros.

### Passo 2: Preencher o `variables.tf` raiz

```bash
cat > variables.tf << EOF
variable "region" {
  description = "Região do Google Cloud"
  default     = "$REGION"
}

variable "zone" {
  description = "Zona do Google Cloud"
  default     = "$ZONE"
}

variable "project_id" {
  description = "ID do projeto no Google Cloud"
  default     = "$PROJECT_ID"
}
EOF
```

**Explicação dos parâmetros:**
- `variable "region"`: declara a variável de região, usada nos recursos para evitar hardcode.
- `default`: valor padrão aplicado quando a variável não é passada explicitamente.

**Resultado esperado:** Arquivo `variables.tf` preenchido com as três variáveis.

### Passo 3: Preencher os `variables.tf` dos módulos

Os dois módulos precisam das mesmas três variáveis. Execute para cada um:

```bash
for MOD in modules/instances modules/storage; do
cat > $MOD/variables.tf << EOF
variable "region" {
  description = "Região do Google Cloud"
  default     = "$REGION"
}

variable "zone" {
  description = "Zona do Google Cloud"
  default     = "$ZONE"
}

variable "project_id" {
  description = "ID do projeto no Google Cloud"
  default     = "$PROJECT_ID"
}
EOF
done
```

**Resultado esperado:** Ambos os módulos com `variables.tf` preenchidos.

### Passo 4: Preencher o `main.tf` raiz com o bloco Terraform e Provider

```bash
cat > main.tf << 'EOF'
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}
EOF
```

**Explicação dos parâmetros:**
- `terraform.required_providers`: declara o provider necessário e sua versão mínima.
- `provider "google"`: configura o provider com projeto, região e zona usando as variáveis definidas.
- `var.project_id`, `var.region`, `var.zone`: referências às variáveis declaradas em `variables.tf`.

**Resultado esperado:** `main.tf` com bloco Terraform e provider configurados.

### Passo 5: Inicializar o Terraform

```bash
terraform init
```

**Resultado esperado:** Mensagem `Terraform has been successfully initialized!` no terminal.

---

## TAREFA 2: Importar Infraestrutura

### Conceito

`terraform import` permite trazer recursos já existentes no provedor para o controle do estado do Terraform, sem recriá-los. É necessário primeiro escrever a configuração do recurso no arquivo `.tf` e só então executar o import. O Terraform compara o estado importado com a configuração e aplica apenas as diferenças.

### Passo 1: Obter os IDs das instâncias existentes

No console do Google Cloud, acesse **Compute Engine > VM Instances**, clique em `tf-instance-1` e anote:
- Instance ID (número longo)
- Tipo de máquina (ex.: `e2-micro`)
- Imagem do disco de boot (ex.: `debian-cloud/debian-11`)

Repita para `tf-instance-2`. Alternativamente, via CLI:

```bash
gcloud compute instances describe tf-instance-1 \
    --zone=$ZONE \
    --format="value(id,machineType,disks[0].source)"

gcloud compute instances describe tf-instance-2 \
    --zone=$ZONE \
    --format="value(id,machineType,disks[0].source)"
```

**Resultado esperado:** IDs numéricos e configurações das instâncias exibidos no terminal.

### Passo 2: Adicionar referência ao módulo no `main.tf`

```bash
cat >> main.tf << 'EOF'

module "instances" {
  source     = "./modules/instances"
  project_id = var.project_id
  region     = var.region
  zone       = var.zone
}
EOF
```

**Resultado esperado:** Bloco `module "instances"` adicionado ao `main.tf`.

### Passo 3: Re-inicializar para registrar o módulo

```bash
terraform init
```

**Resultado esperado:** Terraform reconhece o novo módulo sem erros.

### Passo 4: Escrever a configuração das instâncias em `modules/instances/instances.tf`

Substitua `MACHINE_TYPE`, `IMAGE_PROJECT` e `IMAGE_FAMILY` pelos valores obtidos na consulta acima (ex.: `e2-micro`, `debian-cloud`, `debian-11`):

```bash
cat > modules/instances/instances.tf << 'EOF'
resource "google_compute_instance" "tf-instance-1" {
  name         = "tf-instance-1"
  machine_type = "e2-micro"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = "default"
  }

  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT

  allow_stopping_for_update = true
}

resource "google_compute_instance" "tf-instance-2" {
  name         = "tf-instance-2"
  machine_type = "e2-micro"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = "default"
  }

  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT

  allow_stopping_for_update = true
}
EOF
```

**Explicação dos parâmetros:**
- `machine_type`: tipo de máquina da VM (ex.: `e2-micro`). Deve corresponder ao tipo da instância existente.
- `boot_disk.initialize_params.image`: imagem do SO. Deve corresponder à imagem da instância existente.
- `network_interface.network`: rede à qual a instância está conectada.
- `metadata_startup_script`: script executado na inicialização; mantido vazio para minimizar diferenças no import.
- `allow_stopping_for_update`: permite que o Terraform pare a instância para aplicar mudanças que exigem reinicialização.

**Resultado esperado:** Arquivo `instances.tf` criado com as duas configurações de instância.

### Passo 5: Importar as instâncias para o estado do Terraform

```bash
INSTANCE_1_ID=$(gcloud compute instances describe tf-instance-1 --zone=$ZONE --format='value(id)')
INSTANCE_2_ID=$(gcloud compute instances describe tf-instance-2 --zone=$ZONE --format='value(id)')

# Por ID da instância
terraform import module.instances.google_compute_instance.tf-instance-1 $INSTANCE_1_ID
terraform import module.instances.google_compute_instance.tf-instance-2 $INSTANCE_2_ID

# Por Nome da instância
terraform import module.instances.google_compute_instance.tf-instance-1 \
    $DEVSHELL_PROJECT_ID/$ZONE/tf-instance-1

terraform import module.instances.google_compute_instance.tf-instance-2 \
    $DEVSHELL_PROJECT_ID/$ZONE/tf-instance-2
```

**Explicação dos parâmetros:**
- `module.instances.google_compute_instance.tf-instance-1`: endereço do recurso no estado Terraform seguindo o padrão `module.<nome_modulo>.<tipo_recurso>.<nome_recurso>`.
- `$DEVSHELL_PROJECT_ID/$ZONE/tf-instance-1`: identificador do recurso no Google Cloud no formato `projeto/zona/nome-da-instancia`. O formato `zona/nome` (sem projeto) causa erro no provider — o projeto é obrigatório.

**Resultado esperado:** Mensagens `Import successful!` para cada instância.

### Passo 6: Aplicar as mudanças

```bash
terraform plan
terraform apply -auto-approve
```

**Resultado esperado:** O apply atualiza as instâncias in-place (sem recriar) para alinhar ao estado descrito na configuração.

---

## TAREFA 3: Configurar um Backend Remoto

### Conceito

Por padrão, o Terraform armazena o estado localmente em `terraform.tfstate`. Um backend remoto (Cloud Storage) centraliza o estado, permite colaboração entre equipes e protege contra perda de dados. O prefixo `terraform/state` organiza os arquivos de estado dentro do bucket.

### Passo 1: Escrever o recurso do bucket em `modules/storage/storage.tf`

Substitua `$BUCKET_NAME` pelo nome fornecido pelo lab:

```bash
cat > modules/storage/storage.tf << EOF
resource "google_storage_bucket" "storage" {
  name                        = "$BUCKET_NAME"
  location                    = "US"
  force_destroy               = true
  uniform_bucket_level_access = true
}
EOF
```

**Explicação dos parâmetros:**
- `name`: nome globalmente único do bucket. Usar o valor fornecido pelo lab.
- `location`: região multirregional (`US`) onde o bucket será criado.
- `force_destroy`: permite destruir o bucket mesmo que contenha objetos.
- `uniform_bucket_level_access`: habilita controle de acesso uniforme no nível do bucket (IAM puro, sem ACLs legadas).

**Resultado esperado:** Arquivo `storage.tf` criado com o recurso do bucket.

### Passo 2: Adicionar output opcional em `modules/storage/outputs.tf`

```bash
cat > modules/storage/outputs.tf << 'EOF'
output "bucket_name" {
  description = "Nome do bucket criado"
  value       = google_storage_bucket.storage.name
}
EOF
```

### Passo 3: Adicionar referência ao módulo de storage no `main.tf`

```bash
cat >> main.tf << 'EOF'

module "storage" {
  source     = "./modules/storage"
  project_id = var.project_id
  region     = var.region
  zone       = var.zone
}
EOF
```

### Passo 4: Inicializar e aplicar para criar o bucket

```bash
terraform init
terraform apply -auto-approve
```

**Resultado esperado:** Bucket criado no Cloud Storage. Verificável em **Storage > Browser** no console.

### Passo 5: Configurar o backend remoto no `main.tf`

Edite o bloco `terraform` no início do `main.tf` para adicionar o backend. Substitua `NOME_DO_BUCKET_FORNECIDO_PELO_LAB` pelo nome real:

```bash
# Reescreve o bloco terraform com o backend incluído
# Edite manualmente o main.tf inserindo o bloco backend dentro do bloco terraform existente:
```

O bloco `terraform` deve ficar assim:

```hcl
terraform {
  backend "gcs" {
    bucket = "NOME_DO_BUCKET_FORNECIDO_PELO_LAB"
    prefix = "terraform/state"
  }

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
  }
}
```

**Explicação dos parâmetros:**
- `backend "gcs"`: define o Cloud Storage como backend de estado remoto.
- `bucket`: nome do bucket onde o estado será armazenado.
- `prefix`: caminho dentro do bucket. Deve ser `terraform/state` para avaliação correta do lab.

### Passo 6: Re-inicializar para migrar o estado para o backend remoto

```bash
terraform init
```

Quando solicitado `Do you want to copy existing state to the new backend?`, digite:

```
yes
```

**Resultado esperado:** Estado migrado para o Cloud Storage. O arquivo `terraform.tfstate` local fica vazio ou é removido.

---

## TAREFA 4: Modificar e Atualizar Infraestrutura

### Conceito

O Terraform aplica mudanças incrementais: ao alterar um atributo como `machine_type`, ele para a instância, aplica a mudança e a reinicia (comportamento habilitado pelo `allow_stopping_for_update = true`). Adicionar um novo bloco `resource` ao módulo cria um novo recurso no apply seguinte.

### Passo 1: Atualizar `modules/instances/instances.tf`

Modifique as instâncias existentes para `e2-standard-2` e adicione a terceira instância. Substitua `NOME_DA_INSTANCIA_FORNECIDO_PELO_LAB` pelo valor do lab:

```bash
cat > modules/instances/instances.tf << EOF
resource "google_compute_instance" "tf-instance-1" {
  name         = "tf-instance-1"
  machine_type = "e2-standard-2"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = "default"
  }

  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT

  allow_stopping_for_update = true
}

resource "google_compute_instance" "tf-instance-2" {
  name         = "tf-instance-2"
  machine_type = "e2-standard-2"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = "default"
  }

  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT

  allow_stopping_for_update = true
}

resource "google_compute_instance" "tf-instance-3" {
  name         = "$INSTANCE_3_NAME"
  machine_type = "e2-standard-2"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = "default"
  }

  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT

  allow_stopping_for_update = true
}
EOF
```

**Explicação dos parâmetros:**
- `machine_type = "e2-standard-2"`: tipo de máquina atualizado para as três instâncias. A família `e2-standard` oferece 2 vCPUs e 8 GB de RAM.

### Passo 2: Aplicar as mudanças

```bash
terraform init
terraform apply -auto-approve
```

**Resultado esperado:** As duas instâncias existentes têm o tipo de máquina atualizado in-place e a terceira instância é criada.

---

## TAREFA 5: Destruir Recursos

### Conceito

No Terraform, a forma correta de destruir um recurso específico é remover seu bloco `resource` do arquivo de configuração e executar `terraform apply`. O Terraform detecta que o recurso não existe mais na configuração e o remove do provedor. Isso é preferível a `terraform destroy` completo, que destrói toda a infraestrutura gerenciada.

### Passo 1: Remover a terceira instância de `modules/instances/instances.tf`

Edite o arquivo e delete o bloco `resource "google_compute_instance" "tf-instance-3"` inteiro (as últimas ~20 linhas do arquivo). O arquivo deve conter apenas os recursos `tf-instance-1` e `tf-instance-2`.

Para verificar o conteúdo atual antes de editar:

```bash
terraform state list
```

**Resultado esperado:** Lista com `module.instances.google_compute_instance.tf-instance-1`, `module.instances.google_compute_instance.tf-instance-2` e `module.instances.google_compute_instance.tf-instance-3`.

### Passo 2: Aplicar a remoção

```bash
terraform init
terraform apply -auto-approve
```

**Resultado esperado:** Terraform exibe `Destroy complete! Resources: 1 destroyed.` e a terceira instância é removida do Compute Engine.

---

## TAREFA 6: Usar um Módulo do Terraform Registry

### Conceito

O Terraform Registry hospeda módulos comunitários e oficiais reutilizáveis. O módulo `terraform-google-modules/network/google` abstrai a criação de VPCs, sub-redes, rotas e outras configurações de rede no Google Cloud, reduzindo o código necessário e seguindo boas práticas.

### Passo 1: Adicionar o módulo de rede ao `main.tf`

Substitua `NOME_DA_VPC_FORNECIDO_PELO_LAB` pelo nome fornecido pelo lab:

```bash
cat >> main.tf << EOF

module "vpc" {
  source  = "terraform-google-modules/network/google"
  version = "10.0.0"

  project_id   = var.project_id
  network_name = "$VPC_NAME"
  routing_mode = "GLOBAL"

  subnets = [
    {
      subnet_name   = "subnet-01"
      subnet_ip     = "10.10.10.0/24"
      subnet_region = var.region
    },
    {
      subnet_name   = "subnet-02"
      subnet_ip     = "10.10.20.0/24"
      subnet_region = var.region
    }
  ]
}
EOF
```

**Explicação dos parâmetros:**
- `source`: endereço do módulo no Terraform Registry.
- `version = "10.0.0"`: versão específica para garantir compatibilidade com o lab.
- `network_name`: nome da VPC a ser criada.
- `routing_mode = "GLOBAL"`: modo de roteamento global permite que rotas sejam propagadas para todas as regiões.
- `subnets`: lista de sub-redes; cada uma com nome, bloco CIDR e região.

**Resultado esperado:** Bloco do módulo VPC adicionado ao `main.tf`.

### Passo 2: Inicializar e aplicar para criar a VPC

```bash
terraform init
terraform apply -auto-approve
```

**Resultado esperado:** VPC com duas sub-redes criada. Verificável em **VPC Network > VPC networks** no console.

### Passo 3: Conectar as instâncias às sub-redes

Atualize `modules/instances/instances.tf` adicionando `subnetwork` e alterando `network` em cada instância. Substitua `NOME_DA_VPC_FORNECIDO_PELO_LAB` pelo nome real:

```bash
cat > modules/instances/instances.tf << EOF
resource "google_compute_instance" "tf-instance-1" {
  name         = "tf-instance-1"
  machine_type = "e2-standard-2"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network    = "$VPC_NAME"
    subnetwork = "subnet-01"
  }

  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT

  allow_stopping_for_update = true
}

resource "google_compute_instance" "tf-instance-2" {
  name         = "tf-instance-2"
  machine_type = "e2-standard-2"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network    = "$VPC_NAME"
    subnetwork = "subnet-02"
  }

  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT

  allow_stopping_for_update = true
}
EOF
```

**Explicação dos parâmetros:**
- `network`: nome da VPC à qual a instância se conecta.
- `subnetwork`: nome da sub-rede específica dentro da VPC. `tf-instance-1` vai para `subnet-01` e `tf-instance-2` vai para `subnet-02`.

### Passo 4: Aplicar a conectividade das instâncias

```bash
terraform apply -auto-approve
```

**Resultado esperado:** Instâncias reconectadas às respectivas sub-redes da nova VPC.

---

## TAREFA 7: Configurar um Firewall

### Conceito

Regras de firewall no Google Cloud controlam o tráfego de entrada (ingress) e saída (egress) das instâncias de VM. São aplicadas no nível da VPC e podem ser direcionadas por tags de rede ou intervalos de IP. A regra `tf-firewall` abrirá a porta TCP 80 para toda a internet na VPC criada.

### Passo 1: Adicionar a regra de firewall ao `main.tf`

Para obter o `self_link` da rede VPC criada pelo módulo, use:

```bash
terraform state show module.vpc.module.vpc.google_compute_network.network | grep self_link
```

O valor terá o formato `projects/PROJECT_ID/global/networks/NOME_DA_VPC`. Adicione a regra ao `main.tf`:

```bash
cat >> main.tf << EOF

resource "google_compute_firewall" "tf-firewall" {
  name    = "tf-firewall"
  network = "projects/$PROJECT_ID/global/networks/$VPC_NAME"

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }

  source_ranges = ["0.0.0.0/0"]
  direction     = "INGRESS"
}
EOF
```

**Explicação dos parâmetros:**
- `name = "tf-firewall"`: nome da regra de firewall, obrigatório para avaliação do lab.
- `network`: self_link da VPC onde a regra será aplicada. Use o formato `projects/PROJECT_ID/global/networks/NOME_VPC`.
- `allow.protocol = "tcp"`: protocolo de transporte permitido.
- `allow.ports = ["80"]`: porta HTTP padrão liberada.
- `source_ranges = ["0.0.0.0/0"]`: permite conexões de qualquer endereço IP de origem.
- `direction = "INGRESS"`: regra aplicada ao tráfego de entrada nas instâncias.

**Resultado esperado:** Bloco da regra de firewall adicionado ao `main.tf`.

### Passo 2: Inicializar e aplicar a regra de firewall

```bash
terraform init
terraform apply -auto-approve
```

**Resultado esperado:** Regra `tf-firewall` criada. Verificável em **VPC Network > Firewall** no console.

---

## Validação

Execute os comandos abaixo para verificar o estado final de todos os recursos criados:

```bash
# Listar todas as instâncias de VM
gcloud compute instances list --project=$PROJECT_ID

# Verificar detalhes da VPC
gcloud compute networks describe $VPC_NAME \
    --project=$PROJECT_ID

# Listar sub-redes da VPC
gcloud compute networks subnets list \
    --filter="network:$VPC_NAME" \
    --project=$PROJECT_ID

# Verificar a regra de firewall
gcloud compute firewall-rules describe tf-firewall \
    --project=$PROJECT_ID

# Listar buckets do Cloud Storage
gsutil ls -p $PROJECT_ID

# Verificar o estado remoto do Terraform
gsutil ls gs://NOME_DO_BUCKET_FORNECIDO_PELO_LAB/terraform/state/

# Listar todos os recursos no estado Terraform
terraform state list
```

**Resultado esperado:**
- Duas instâncias (`tf-instance-1`, `tf-instance-2`) visíveis e em execução.
- VPC com `subnet-01` (10.10.10.0/24) e `subnet-02` (10.10.20.0/24) existentes.
- Regra `tf-firewall` ativa permitindo TCP/80 de 0.0.0.0/0.
- Arquivo de estado (`*.tfstate`) presente no bucket do Cloud Storage.

---

## Troubleshooting

| Sintoma | Causa Provável | Solução |
|---|---|---|
| `Error: Instance not found` no import | Zona incorreta no endereço de import | Verifique a zona da instância com `gcloud compute instances list` e use o formato `ZONA/NOME-INSTANCIA` |
| `Error: googleapi: Error 409: The resource already exists` | Recurso já existe no provedor mas não no estado | Use `terraform import` para trazer o recurso para o estado antes de aplicar |
| `terraform init` falha ao migrar estado para o GCS | Bucket não existe ainda ou permissões insuficientes | Certifique-se de que o `terraform apply` do módulo storage foi concluído antes de configurar o backend |
| `Error: Invalid value for "machine_type"` | Nome do tipo de máquina incorreto | Confirme o nome exato com `gcloud compute machine-types list --zone=$ZONE` |
| `module.vpc` não encontrado no `terraform state show` | Módulo não inicializado | Execute `terraform init` antes de `terraform apply` após adicionar o módulo |
| `Error: Error creating Firewall: googleapi: Error 403` | Permissão insuficiente na conta de serviço | Verifique se a conta tem o papel `roles/compute.securityAdmin` |
| Apply altera instâncias sem `allow_stopping_for_update` | Instância não pode ser modificada em execução | Adicione `allow_stopping_for_update = true` ao bloco de recurso |
| Estado remoto corrompido ou inacessível | Bucket deletado ou prefixo errado | Verifique `gs://BUCKET/terraform/state/` com `gsutil ls` e corrija o `prefix` no backend |

---

## Limpeza (Opcional)

> **Atenção:** Execute estes comandos somente se desejar remover toda a infraestrutura criada. Em ambientes de lab, os recursos são destruídos automaticamente ao encerrar o lab.

```bash
# Destruir todos os recursos gerenciados pelo Terraform
terraform destroy -auto-approve
```

Se o backend remoto estiver inacessível, force a destruição local:

```bash
# Migrar estado de volta para local antes de destruir (se necessário)
terraform init -migrate-state
terraform destroy -auto-approve
```

---

## Conceitos-Chave

| Conceito | Descrição |
|---|---|
| `terraform import` | Comando que importa um recurso existente no provedor para o estado do Terraform, sem recriar o recurso |
| Módulo Terraform | Conjunto de arquivos `.tf` em um diretório que encapsulam recursos relacionados para reutilização |
| Backend remoto | Armazenamento externo (ex.: GCS) para o arquivo de estado, permitindo colaboração e durabilidade |
| `terraform.tfstate` | Arquivo JSON que mapeia os recursos da configuração aos recursos reais no provedor |
| VPC (Virtual Private Cloud) | Rede privada virtual isolada no Google Cloud que contém sub-redes, rotas e regras de firewall |
| Sub-rede | Segmento de uma VPC com um bloco CIDR específico, associado a uma região |
| Regra de firewall | Política que controla o tráfego de entrada ou saída em uma VPC com base em protocolo, porta e origem |
| `routing_mode = "GLOBAL"` | Modo de roteamento da VPC que propaga rotas aprendidas dinamicamente para todas as regiões |
| Terraform Registry | Repositório público de módulos e providers Terraform mantidos pela HashiCorp e pela comunidade |
| `allow_stopping_for_update` | Argumento que autoriza o Terraform a parar uma VM para aplicar mudanças que exigem reinicialização |
| `force_destroy` | Argumento do bucket GCS que permite a sua deleção mesmo quando contém objetos |
| `prefix` no backend GCS | Caminho virtual dentro do bucket onde os arquivos de estado são armazenados |

---

## Fluxo Final

```
Cloud Shell
    │
    ├── terraform init (Tarefa 1)
    │       Inicializa provider google e estrutura de módulos
    │
    ├── terraform import (Tarefa 2)
    │       tf-instance-1 ──► module.instances.google_compute_instance.tf-instance-1
    │       tf-instance-2 ──► module.instances.google_compute_instance.tf-instance-2
    │
    ├── terraform apply (Tarefa 3)
    │       module.storage ──► gs://BUCKET_NAME/
    │       backend "gcs"  ──► terraform/state/*.tfstate
    │
    ├── terraform apply (Tarefa 4)
    │       tf-instance-1  ──► e2-standard-2 (atualizado)
    │       tf-instance-2  ──► e2-standard-2 (atualizado)
    │       tf-instance-3  ──► e2-standard-2 (novo, depois destruído na Tarefa 5)
    │
    ├── terraform apply (Tarefa 5)
    │       tf-instance-3  ──► DESTRUÍDO
    │
    ├── terraform apply (Tarefa 6)
    │       module.vpc
    │           ├── VPC: NOME_DA_VPC
    │           ├── subnet-01: 10.10.10.0/24
    │           └── subnet-02: 10.10.20.0/24
    │       tf-instance-1  ──► conectada à subnet-01
    │       tf-instance-2  ──► conectada à subnet-02
    │
    └── terraform apply (Tarefa 7)
            google_compute_firewall.tf-firewall
                ├── Rede: NOME_DA_VPC
                ├── Protocolo: TCP porta 80
                └── Origem: 0.0.0.0/0 (INGRESS)

Estado Final:
┌─────────────────────────────────────────────────────┐
│  VPC: NOME_DA_VPC (routing: GLOBAL)                 │
│  ┌──────────────────┐  ┌──────────────────┐         │
│  │ subnet-01        │  │ subnet-02        │         │
│  │ 10.10.10.0/24    │  │ 10.10.20.0/24   │         │
│  │ [tf-instance-1]  │  │ [tf-instance-2] │         │
│  │ e2-standard-2    │  │ e2-standard-2   │         │
│  └──────────────────┘  └──────────────────┘         │
│                                                     │
│  Firewall: tf-firewall                              │
│  TCP:80 ← 0.0.0.0/0 (INGRESS)                      │
└─────────────────────────────────────────────────────┘
           Estado armazenado em:
           gs://BUCKET_NAME/terraform/state/
```
