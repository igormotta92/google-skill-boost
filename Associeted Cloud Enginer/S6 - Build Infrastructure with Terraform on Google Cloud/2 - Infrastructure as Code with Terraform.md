# Infrastructure as Code with Terraform

## Visão Geral

O Terraform é a solução de infraestrutura como código (IaC) da HashiCorp. É uma ferramenta para criar, alterar e gerenciar infraestrutura de forma segura e repetível. Operadores e equipes de infraestrutura podem usar o Terraform para gerenciar ambientes com uma linguagem de configuração chamada HashiCorp Configuration Language (HCL), projetada para ser legível por humanos e permitir implantações automatizadas.

Infraestrutura como código é o processo de gerenciar infraestrutura em arquivos de configuração, em vez de configurar recursos manualmente por meio de uma interface gráfica. Um recurso, nesse contexto, é qualquer componente de infraestrutura em um determinado ambiente, como uma máquina virtual, grupo de segurança, interface de rede, etc.

O fluxo de trabalho básico de implantação com Terraform segue estas etapas:

- **Escopo** — Confirmar quais recursos precisam ser criados para o projeto.
- **Autoria** — Criar o arquivo de configuração em HCL com os parâmetros definidos.
- **Inicialização** — Executar `terraform init` no diretório do projeto para baixar os plug-ins do provider correto.
- **Planejar e Aplicar** — Executar `terraform plan` para verificar o processo de criação e, em seguida, `terraform apply` para provisionar os recursos reais e gerar o arquivo de estado.

## Introdução

Neste lab, você aprenderá a criar, modificar, destruir e provisionar infraestrutura no Google Cloud usando o Terraform. Serão explorados conceitos fundamentais como dependências implícitas e explícitas entre recursos, uso de IPs estáticos, buckets do Cloud Storage e provisionadores locais.

O Terraform já vem pré-instalado no Cloud Shell do Google Cloud, portanto nenhuma instalação adicional é necessária.

## Pré-requisitos e Variáveis

### Variáveis de ambiente

Configure as variáveis abaixo antes de executar os comandos. Substitua os valores conforme o seu ambiente:

```bash
export PROJECT_ID="qwiklabs-gcp-04-edc4d999c468"   # ajuste conforme ambiente
export REGION="europe-west1"          # ajuste conforme ambiente
export ZONE="europe-west1-c"          # ajuste conforme ambiente
export BUCKET_NAME="qwiklabs-gcp-04-edc4d999c468"  # deve ser globalmente único
```

### Autenticação e projeto

```bash
gcloud auth list
gcloud config list project
gcloud config set project $PROJECT_ID
```

**Explicação dos parâmetros:**
- `auth list`: exibe a conta ativa autenticada no Cloud Shell.
- `config list project`: exibe o Project ID configurado na sessão atual.
- `config set project`: define explicitamente o projeto ativo para os comandos subsequentes.

### Diretório de trabalho

```bash
mkdir -p ~/terraform-lab && cd ~/terraform-lab
```

---

## TAREFA 1: Construir a Infraestrutura Base

### Conceito

O arquivo `main.tf` é o ponto de entrada de uma configuração Terraform. Ele contém três blocos principais:

- **`terraform {}`**: declara os providers necessários e suas versões.
- **`provider "google" {}`**: configura o provider do Google Cloud com projeto, região e zona.
- **`resource "tipo" "nome" {}`**: define os recursos que serão criados.

### Passo 1: Criar o arquivo de configuração principal

```bash
touch ~/terraform-lab/main.tf
```

**Resultado esperado:** arquivo `main.tf` criado no diretório de trabalho.

### Passo 2: Adicionar o conteúdo do main.tf

Abra o arquivo `~/terraform-lab/main.tf` em um editor de texto e adicione o seguinte conteúdo, substituindo os valores de `project`, `region` e `zone`:

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "3.5.0"
    }
  }
}

provider "google" {
  project = "SEU_PROJECT_ID"
  region  = "us-central1"
  zone    = "us-central1-a"
}

resource "google_compute_network" "vpc_network" {
  name = "terraform-network"
}
```

**Explicação dos parâmetros:**
- `required_providers`: declara o provider `google` da HashiCorp no registro oficial.
- `version = "3.5.0"`: trava a versão do provider para evitar quebras por atualizações automáticas.
- `provider "google"`: configura credenciais e escopo do provider.
- `resource "google_compute_network"`: declara uma rede VPC no Google Cloud.
- `name = "terraform-network"`: nome da rede VPC que será criada.

### Passo 3: Inicializar o Terraform

```bash
cd ~/terraform-lab && terraform init
```

**Explicação dos parâmetros:**
- `init`: inicializa o diretório de trabalho, baixa o plug-in do provider `google` na versão especificada e prepara o backend de estado local.

**Resultado esperado:**
```
Terraform has been successfully initialized!
```

### Passo 4: Aplicar a configuração e criar a rede VPC

```bash
terraform apply
```

Quando solicitado, confirme digitando `yes` e pressionando ENTER.

**Resultado esperado:**
```
google_compute_network.vpc_network: Creation complete after 58s [id=terraform-network]
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

### Passo 5: Inspecionar o estado atual

```bash
terraform show
```

**Explicação dos parâmetros:**
- `show`: exibe o estado atual da infraestrutura gerenciada pelo Terraform, com todos os atributos dos recursos criados.

**Resultado esperado:** lista detalhada dos atributos da rede `terraform-network`, como `id`, `self_link`, `subnetworks`, etc.

---

## TAREFA 2: Alterar a Infraestrutura

### Conceito

O Terraform detecta diferenças entre a configuração declarada e o estado atual dos recursos. Quando você modifica o `main.tf` e executa `terraform apply`, ele calcula apenas o delta necessário e aplica as mudanças mínimas — sem recriar tudo do zero, a menos que a mudança exija isso (mudança destrutiva).

Existem dois tipos de alterações:
- **In-place (`~`)**: o recurso é atualizado sem ser destruído (ex.: adicionar tags).
- **Destrutiva (`-/+`)**: o recurso é destruído e recriado (ex.: trocar a imagem de disco).

### Passo 1: Adicionar uma instância de VM ao main.tf

Adicione o bloco abaixo ao final do arquivo `main.tf`:

```hcl
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "e2-micro"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
}
```

**Explicação dos parâmetros:**
- `machine_type = "e2-micro"`: tipo de máquina de baixo custo, adequado para testes.
- `image = "debian-cloud/debian-12"`: imagem do sistema operacional Debian 12.
- `network = google_compute_network.vpc_network.name`: referência implícita à rede VPC criada anteriormente — o Terraform infere a dependência automaticamente.
- `access_config {}`: bloco vazio que garante que a instância tenha acesso à internet (IP externo efêmero).

```bash
terraform apply
```

Confirme com `yes`.

**Resultado esperado:** instância `terraform-instance` criada com IP externo efêmero na rede `terraform-network`.

### Passo 2: Adicionar tags à instância (alteração in-place)

Modifique o bloco `resource "google_compute_instance" "vm_instance"` no `main.tf` para incluir o argumento `tags`:

```hcl
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "e2-micro"
  tags         = ["web", "dev"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
}
```

```bash
terraform apply
```

Confirme com `yes`.

**Explicação dos parâmetros:**
- `tags = ["web", "dev"]`: rótulos de rede aplicados à instância, usados para regras de firewall.

**Resultado esperado:** a instância recebe as tags `web` e `dev` sem ser destruída (prefixo `~` no plano).

### Passo 3: Trocar a imagem de disco (alteração destrutiva)

Modifique o bloco `boot_disk` dentro de `vm_instance` no `main.tf`:

```hcl
  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }
```

```bash
terraform apply
```

Confirme com `yes`.

**Explicação dos parâmetros:**
- `image = "cos-cloud/cos-stable"`: Container-Optimized OS do Google — troca de imagem exige destruição e recriação da instância.

**Resultado esperado:** a instância é destruída e recriada com a nova imagem (prefixo `-/+` no plano).

### Passo 4: Destruir toda a infraestrutura

```bash
terraform destroy
```

Confirme com `yes`.

**Explicação dos parâmetros:**
- `destroy`: remove todos os recursos gerenciados pelo Terraform, na ordem correta de dependências (a VM é destruída antes da rede VPC).

**Resultado esperado:**
```
Destroy complete! Resources: 2 destroyed.
```

---

## TAREFA 3: Criar Dependências entre Recursos

### Conceito

O Terraform suporta dois tipos de dependências:

- **Implícita**: criada automaticamente quando um recurso referencia atributos de outro via interpolação (ex.: `google_compute_address.vm_static_ip.address`). O Terraform detecta e respeita a ordem de criação.
- **Explícita**: declarada com o argumento `depends_on`, usada quando a dependência existe na lógica da aplicação mas não é visível na configuração do Terraform.

### Passo 1: Recriar a rede e a instância base

```bash
terraform apply
```

Confirme com `yes`.

**Resultado esperado:** rede VPC `terraform-network` e instância `terraform-instance` recriadas.

### Passo 2: Adicionar um IP estático ao main.tf

Adicione o seguinte bloco ao `main.tf`:

```hcl
resource "google_compute_address" "vm_static_ip" {
  name = "terraform-static-ip"
}
```

**Explicação dos parâmetros:**
- `google_compute_address`: reserva um endereço IP externo estático no projeto.
- `name = "terraform-static-ip"`: nome do recurso de IP reservado.

### Passo 3: Vincular o IP estático à instância

Atualize o bloco `network_interface` da instância `vm_instance` no `main.tf`:

```hcl
  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
      nat_ip = google_compute_address.vm_static_ip.address
    }
  }
```

**Explicação dos parâmetros:**
- `self_link`: referência completa ao recurso de rede (URL da API), mais robusta que o `name`.
- `nat_ip = google_compute_address.vm_static_ip.address`: cria uma dependência implícita — o Terraform garante que o IP seja criado antes da VM ser atualizada.

### Passo 4: Planejar e salvar o plano de execução

```bash
terraform plan -out static_ip
```

**Explicação dos parâmetros:**
- `-out static_ip`: salva o plano de execução no arquivo `static_ip`. Garante que exatamente as mudanças planejadas serão aplicadas na próxima execução.

**Resultado esperado:** plano mostrando criação de `google_compute_address.vm_static_ip` e atualização de `google_compute_instance.vm_instance`.

### Passo 5: Aplicar o plano salvo

```bash
terraform apply "static_ip"
```

**Resultado esperado:** IP estático criado primeiro, depois a VM atualizada com o `nat_ip` apontando para o IP reservado.

### Passo 6: Adicionar bucket do Cloud Storage e instância com dependência explícita

Adicione ao `main.tf` (substitua `SEU-NOME-UNICO-DE-BUCKET` por um nome globalmente único):

```hcl
resource "google_storage_bucket" "example_bucket" {
  name     = "SEU-NOME-UNICO-DE-BUCKET"
  location = "US"

  website {
    main_page_suffix = "index.html"
    not_found_page   = "404.html"
  }
}

resource "google_compute_instance" "another_instance" {
  depends_on = [google_storage_bucket.example_bucket]

  name         = "terraform-instance-2"
  machine_type = "e2-micro"

  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }

  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
    }
  }
}
```

**Explicação dos parâmetros:**
- `google_storage_bucket`: cria um bucket do Cloud Storage.
- `location = "US"`: localização multi-regional do bucket.
- `website {}`: configura o bucket para hospedagem estática de sites.
- `depends_on = [google_storage_bucket.example_bucket]`: dependência explícita — garante que o bucket seja criado antes da segunda instância, mesmo sem referência direta nos atributos.

```bash
terraform plan
terraform apply
```

Confirme com `yes`.

**Resultado esperado:** bucket e segunda instância criados. A ordem de criação garante que o bucket existe antes da instância `terraform-instance-2`.

### Passo 7: Remover os recursos temporários (bucket e segunda instância)

Remova do `main.tf` os blocos `google_storage_bucket.example_bucket` e `google_compute_instance.another_instance` e execute:

```bash
terraform apply
```

Confirme com `yes`.

**Resultado esperado:** bucket e segunda instância destruídos, mantendo apenas a rede e a primeira instância.

---

## TAREFA 4: Provisionar Infraestrutura

### Conceito

Provisioners permitem executar scripts e comandos após a criação de um recurso. Existem dois tipos principais:

- **`local-exec`**: executa comandos na máquina local onde o Terraform está rodando (ex.: Cloud Shell).
- **`remote-exec`**: executa comandos dentro do recurso remoto criado (requer configuração de conexão SSH ou WinRM).

Provisioners só são executados na criação do recurso. Se você adicionar um provisioner a um recurso já existente, é necessário marcar o recurso como "tainted" para forçar sua recriação.

### Passo 1: Adicionar um provisioner local-exec à instância

Modifique o bloco `resource "google_compute_instance" "vm_instance"` no `main.tf` para incluir o provisioner:

```hcl
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "e2-micro"
  tags         = ["web", "dev"]

  provisioner "local-exec" {
    command = "echo ${google_compute_instance.vm_instance.name}:  ${google_compute_instance.vm_instance.network_interface[0].access_config[0].nat_ip} >> ip_address.txt"
  }

  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }

  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
      nat_ip = google_compute_address.vm_static_ip.address
    }
  }
}
```

**Explicação dos parâmetros:**
- `provisioner "local-exec"`: bloco que define a execução local de um comando.
- `command`: comando shell a ser executado. Usa interpolação para obter o nome e o IP externo da instância.
- `network_interface[0].access_config[0].nat_ip`: acessa o primeiro IP NAT da primeira interface de rede (indexação começa em 0).
- `>> ip_address.txt`: redireciona a saída em modo append para o arquivo `ip_address.txt`.

### Passo 2: Aplicar a configuração (sem efeito imediato)

```bash
terraform apply
```

Confirme com `yes`.

**Resultado esperado:** Terraform não encontrará nada a fazer, pois a instância já existe e provisioners só rodam na criação. O arquivo `ip_address.txt` ainda não será gerado.

### Passo 3: Marcar a instância como tainted para forçar recriação

```bash
terraform taint google_compute_instance.vm_instance
```

**Explicação dos parâmetros:**
- `taint`: marca um recurso específico como "contaminado". Na próxima execução de `terraform apply`, o recurso será destruído e recriado, executando os provisioners novamente.
- `google_compute_instance.vm_instance`: endereço completo do recurso no formato `tipo.nome`.

**Resultado esperado:**
```
Resource instance google_compute_instance.vm_instance has been marked as tainted.
```

### Passo 4: Aplicar novamente para recriar a instância e executar o provisioner

```bash
terraform apply
```

Confirme com `yes`.

**Resultado esperado:** instância recriada e arquivo `ip_address.txt` gerado no diretório de trabalho com o conteúdo:
```
terraform-instance: <IP_EXTERNO_DA_INSTANCIA>
```

### Passo 5: Verificar o conteúdo do arquivo gerado

```bash
cat ~/terraform-lab/ip_address.txt
```

**Resultado esperado:** linha com o nome da instância e seu IP externo estático.

---

## Validação

### Verificar recursos Terraform

```bash
# Exibir estado completo de todos os recursos gerenciados
terraform show

# Listar recursos no estado do Terraform
terraform state list

# Inspecionar um recurso específico
terraform state show google_compute_instance.vm_instance
terraform state show google_compute_network.vpc_network
terraform state show google_compute_address.vm_static_ip
```

### Verificar via gcloud

```bash
# Listar redes VPC do projeto
gcloud compute networks list --filter="name=terraform-network"

# Listar instâncias de VM
gcloud compute instances list --filter="name=terraform-instance"

# Listar IPs estáticos reservados
gcloud compute addresses list --filter="name=terraform-static-ip"

# Descrever a instância de VM
gcloud compute instances describe terraform-instance --zone=$ZONE

# Verificar arquivo de IPs gerado pelo provisioner
cat ~/terraform-lab/ip_address.txt
```

**Resultado esperado de `gcloud compute instances list`:**
```
NAME                ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
terraform-instance  us-central1-a  e2-micro                   10.128.x.x   <IP_ESTATICO>  RUNNING
```

---

## Troubleshooting

| Sintoma | Causa Provável | Solução |
|---|---|---|
| `Error: Provider produced inconsistent result` | Versão de provider incompatível | Trave a versão em `required_providers` ou execute `terraform init -upgrade` |
| `Error 409: The resource 'terraform-network' already exists` | Recurso criado fora do Terraform ou estado desatualizado | Execute `terraform import` para importar o recurso ou remova-o manualmente |
| `Error: Bucket name already exists` | Nome de bucket do Cloud Storage não é globalmente único | Escolha um nome diferente — inclua seu Project ID ou timestamp no nome |
| Provisioner não executa após `terraform apply` | Recurso já existe; provisioners só rodam na criação | Execute `terraform taint <recurso>` e aplique novamente |
| `Error: googleapi: Error 403: Forbidden` | Conta sem permissões necessárias ou API não habilitada | Verifique as permissões IAM e habilite as APIs necessárias |
| `ip_address.txt` não é criado | Provisioner falhou silenciosamente ou a instância não foi recriada | Verifique os logs com `TF_LOG=DEBUG terraform apply` e use `terraform taint` |
| `Error: Failed to query available provider packages` | Sem acesso à internet ou proxy mal configurado | Verifique a conectividade do Cloud Shell com registry.terraform.io |

---

## Limpeza (Opcional)

Para destruir todos os recursos criados pelo Terraform neste lab:

```bash
cd ~/terraform-lab
terraform destroy
```

Confirme com `yes`.

**Resultado esperado:**
```
Destroy complete! Resources: N destroyed.
```

Para remover também o diretório de trabalho local:

```bash
rm -rf ~/terraform-lab
```

---

## Conceitos-Chave

| Conceito | Descrição |
|---|---|
| HCL (HashiCorp Configuration Language) | Linguagem declarativa usada para descrever a infraestrutura desejada em arquivos `.tf` |
| Provider | Plugin que conecta o Terraform a uma plataforma específica (Google Cloud, AWS, etc.) |
| Resource | Unidade básica de infraestrutura declarada no Terraform (VM, rede, bucket, IP, etc.) |
| `terraform init` | Inicializa o diretório de trabalho e baixa os providers necessários |
| `terraform plan` | Gera um plano de execução mostrando o que será criado, alterado ou destruído |
| `terraform apply` | Executa o plano e cria/altera/destrói os recursos no provedor de nuvem |
| `terraform destroy` | Destrói todos os recursos gerenciados pelo Terraform no projeto |
| `terraform taint` | Marca um recurso para ser destruído e recriado na próxima aplicação |
| Estado (State) | Arquivo `terraform.tfstate` que mapeia os recursos declarados aos recursos reais na nuvem |
| Dependência implícita | Dependência inferida automaticamente pelo Terraform via referência de atributos entre recursos |
| Dependência explícita | Dependência declarada manualmente com o argumento `depends_on` |
| Provisioner | Mecanismo para executar scripts ou comandos após a criação de um recurso |
| `local-exec` | Provisioner que executa comandos na máquina local onde o Terraform roda |
| Mudança destrutiva (`-/+`) | Alteração que exige destruição e recriação do recurso (ex.: troca de imagem de boot) |
| Mudança in-place (`~`) | Alteração aplicada diretamente no recurso sem destruí-lo (ex.: adição de tags) |

---

## Fluxo Final

```
terraform init
     |
     v
[Download provider hashicorp/google v3.5.0]
     |
     v
terraform apply  (Tarefa 1)
     |
     +---> google_compute_network "terraform-network"  (VPC)
     |
     v
terraform apply  (Tarefa 2 - adicionar VM)
     |
     +---> google_compute_instance "terraform-instance"
     |       |- boot_disk: debian-cloud/debian-12
     |       |- network: terraform-network
     |
     v
terraform apply  (Tarefa 2 - adicionar tags, in-place ~)
     |
     +---> vm_instance atualizada com tags ["web", "dev"]
     |
     v
terraform apply  (Tarefa 2 - trocar imagem, destrutiva -/+)
     |
     +---> vm_instance destruída e recriada com cos-cloud/cos-stable
     |
     v
terraform destroy  (Tarefa 2 - limpeza)
     |
     v
terraform apply  (Tarefa 3 - recriar base)
     |
     +---> vpc_network + vm_instance recriadas
     |
     v
terraform plan -out static_ip  &&  terraform apply "static_ip"
     |
     +---> google_compute_address "terraform-static-ip"  (IP estático)
     |       |
     |       v (dependência implícita)
     +---> vm_instance.network_interface.nat_ip = static_ip.address
     |
     v
terraform apply  (Tarefa 3 - dependência explícita)
     |
     +---> google_storage_bucket "example_bucket"
     |       |
     |       v (depends_on)
     +---> google_compute_instance "terraform-instance-2"
     |
     v
terraform apply  (Tarefa 3 - remover bucket e segunda instância)
     |
     v
terraform taint google_compute_instance.vm_instance  (Tarefa 4)
     |
     v
terraform apply  (Tarefa 4 - recriar com provisioner)
     |
     +---> vm_instance recriada
     |       |
     |       v (provisioner local-exec)
     +---> ip_address.txt gerado localmente com nome:IP
     |
     v
[Infraestrutura final: vpc_network + vm_instance + static_ip]
```
