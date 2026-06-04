# Terraform Fundamentals — Guia Prático CLI

## Visão Geral

Neste laboratório, você utiliza o Terraform para provisionar uma instância de máquina virtual (VM) no Google Cloud. O Terraform é uma ferramenta de Infrastructure as Code (IaC) que permite criar, alterar e versionar infraestrutura de forma segura e previsível, codificando APIs em arquivos declarativos de configuração.

---

## Introdução

O fluxo principal do Terraform envolve três etapas:

1. **Escrever** a configuração (arquivos `.tf`)
2. **Planejar** as mudanças (`terraform plan`)
3. **Aplicar** a infraestrutura (`terraform apply`)

Neste lab, você provisiona uma VM Compute Engine utilizando um arquivo de configuração Terraform, ativa a API do Gemini Code Assist e inspeciona o estado resultante.

---

## Pré-requisitos e Variáveis

Antes de iniciar, exporte as variáveis do seu ambiente:

```bash
export PROJECT_ID="SEU_PROJECT_ID"   # ajuste conforme ambiente
export ZONE="us-central1-a"          # ajuste conforme ambiente
export VM_NAME="terraform"
export MACHINE_TYPE="e2-medium"
```

Verifique que você está autenticado e com o projeto correto:

```bash
gcloud auth list
gcloud config list project
```

---

## TAREFA 1: Verificar a Instalação do Terraform

### Conceito

O Terraform vem pré-instalado no Cloud Shell. Antes de criar qualquer recurso, confirme que o binário está disponível e funcional.

### Passos

```bash
terraform
```

### Resultado Esperado

A saída exibe a ajuda do Terraform com os subcomandos disponíveis (`init`, `validate`, `plan`, `apply`, `destroy`, etc.), confirmando que o binário está instalado e acessível no PATH.

---

## TAREFA 2: Construir a Infraestrutura

### Conceito

O Terraform usa arquivos `.tf` para descrever a infraestrutura desejada. O provider `google` traduz os blocos de configuração em chamadas à API do Google Cloud. O ciclo de vida básico é: `init` → `plan` → `apply`.

### Passo 1: Habilitar a API do Gemini Code Assist

```bash
gcloud services enable cloudaicompanion.googleapis.com \
    --project="${PROJECT_ID}"
```

**Explicação dos parâmetros:**

- `services enable`: ativa uma API no projeto
- `cloudaicompanion.googleapis.com`: identificador da API Gemini Code Assist
- `--project`: projeto onde a API será habilitada

**Resultado esperado:** a API é ativada e o Gemini Code Assist fica disponível no Cloud Shell Editor.

---

### Passo 2: Criar o Arquivo de Configuração Terraform

```bash
touch instance.tf
```

Abra o arquivo `instance.tf` e insira o conteúdo abaixo (substitua os placeholders):

```hcl
resource "google_compute_instance" "default" {
  project      = "SEU_PROJECT_ID"   # ajuste conforme ambiente
  zone         = "us-central1-a"    # ajuste conforme ambiente
  name         = "terraform"
  machine_type = "e2-medium"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    network = "default"
  }
}
```

**Explicação dos blocos e parâmetros:**

| Campo | Descrição |
|---|---|
| `resource "google_compute_instance" "default"` | Declara um recurso do tipo VM Compute Engine com o nome lógico `default` |
| `project` | ID do projeto GCP onde a VM será criada |
| `zone` | Zona de implantação da VM |
| `name` | Nome da instância dentro do Compute Engine |
| `machine_type` | Tipo de máquina (vCPUs e memória) |
| `boot_disk.initialize_params.image` | Imagem do sistema operacional do disco de inicialização |
| `network_interface.network` | Rede VPC à qual a VM será conectada |

Confirme que apenas um arquivo `.tf` existe no diretório:

```bash
ls
```

**Resultado esperado:** somente o arquivo `instance.tf` listado (o Terraform carrega todos os arquivos `.tf` do diretório).

---

### Passo 3: Inicializar o Terraform

```bash
terraform init
```

**Explicação:**

- Baixa e instala o plugin do provider `hashicorp/google` em `.terraform/`
- Cria o arquivo de lock `.terraform.lock.hcl` com a versão exata do provider

**Resultado esperado:** mensagem `Terraform has been successfully initialized!` e instalação do provider Google (ex.: `Installing hashicorp/google v6.x.x...`).

---

### Passo 4: Criar o Plano de Execução

```bash
terraform plan
```

**Explicação:**

- Compara o estado atual (vazio, pois é a primeira execução) com a configuração declarada
- Exibe um diff de todos os recursos que serão criados, modificados ou destruídos
- Nenhuma alteração real é feita nesta etapa

**Resultado esperado:** saída com `Plan: 1 to add, 0 to change, 0 to destroy` e detalhes da VM a ser criada.

---

### Passo 5: Aplicar a Configuração

```bash
terraform apply
```

Quando solicitado, confirme com:

```
yes
```

**Explicação:**

- Executa as ações descritas no plano
- Provisiona a VM no Google Cloud via API
- Grava o estado resultante em `terraform.tfstate`

**Resultado esperado:** mensagem `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.` após a VM ser provisionada com sucesso.

---

## TAREFA 3: Inspecionar o Estado

### Conceito

O arquivo `terraform.tfstate` registra os IDs e atributos de todos os recursos gerenciados pelo Terraform. O comando `terraform show` exibe esse estado de forma legível.

### Passos

```bash
terraform show
```

**Resultado esperado:** saída com todos os atributos da VM criada, incluindo `id`, `instance_id`, `machine_type`, `zone`, `self_link` e detalhes de disco e interface de rede.

---

## Validação

Verifique que a VM foi criada corretamente:

```bash
# Listar instâncias do projeto na zona configurada
gcloud compute instances list \
    --project="${PROJECT_ID}" \
    --filter="name=${VM_NAME}"
```

```bash
# Descrever a instância
gcloud compute instances describe "${VM_NAME}" \
    --zone="${ZONE}" \
    --project="${PROJECT_ID}"
```

**Resultado esperado:** a VM `terraform` aparece com status `RUNNING`, tipo de máquina `e2-medium` e imagem de boot `debian-12`.

---

## Troubleshooting

| Sintoma | Causa Provável | Solução |
|---|---|---|
| `Error: google: could not find default credentials` | Provider sem credenciais configuradas | Execute `gcloud auth application-default login` |
| `Error: googleapi: Error 403: ...` | API Compute Engine não habilitada | `gcloud services enable compute.googleapis.com` |
| `Error: Error acquiring the state lock` | Processo anterior travado | `terraform force-unlock <LOCK_ID>` |
| `Plan: 0 to add` quando esperava criação | Arquivo `.tf` com erro de sintaxe ou vazio | Execute `terraform validate` para checar erros |

### Limpeza (Opcional)

Para destruir todos os recursos criados pelo Terraform:

```bash
terraform destroy
```

Confirme com `yes` quando solicitado.

---

## Conceitos-Chave

| Conceito | Descrição |
|---|---|
| **Infrastructure as Code (IaC)** | Infraestrutura descrita em arquivos versionáveis e reutilizáveis |
| **Provider** | Plugin que traduz blocos `.tf` em chamadas de API (ex.: `hashicorp/google`) |
| **Resource** | Unidade de infraestrutura gerenciada pelo Terraform (ex.: VM, rede, disco) |
| **terraform init** | Inicializa o diretório de trabalho e baixa providers |
| **terraform plan** | Gera um plano de execução sem aplicar mudanças |
| **terraform apply** | Aplica as mudanças para atingir o estado declarado |
| **terraform show** | Exibe o estado atual da infraestrutura gerenciada |
| **terraform.tfstate** | Arquivo de estado que mapeia recursos declarados para objetos reais na nuvem |
| **Execution Plan** | Descrição detalhada das ações que o Terraform executará (create/update/destroy) |

---

## Fluxo Final

```
Escrever instance.tf
        ↓
terraform init       → baixa provider hashicorp/google
        ↓
terraform plan       → exibe diff: 1 VM a criar
        ↓
terraform apply      → provisiona VM no Compute Engine
        ↓
terraform show       → inspeciona estado salvo em terraform.tfstate
        ↓
gcloud compute instances list → valida VM ativa
```
