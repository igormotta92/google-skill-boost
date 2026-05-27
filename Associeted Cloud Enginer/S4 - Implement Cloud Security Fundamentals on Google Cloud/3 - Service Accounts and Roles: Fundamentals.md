# Service Accounts and Roles: Fundamentals (CLI)

## Introducao

Este guia converte o desafio para uma execucao 100% via CLI, com foco em:

- criacao e gestao de service accounts;
- atribuicao de papeis IAM;
- uso de service account em VM para consultar BigQuery.

O fluxo abaixo preserva os nomes de recursos exigidos no lab e usa placeholders onde o ambiente pode variar.

---

## Pre-requisitos e Variaveis

### Conceito

Padronizar variaveis evita erro de digitacao e facilita reaproveitar os comandos.

### Passo 1: Definir variaveis

```bash
# export PROJECT_ID="$(gcloud config get-value project)"
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export REGION="us-central1"         # ajuste conforme ambiente, ex: us-central1
export ZONE="us-central1-a"         # ajuste conforme ambiente, ex: us-central1-a

export SA_TASK1_NAME="my-sa-123"
export SA_BQ_NAME="bigquery-qwiklab"
export VM_NAME="bigquery-instance"

export SA_TASK1_EMAIL="${SA_TASK1_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
export SA_BQ_EMAIL="${SA_BQ_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
```

### Passo 2: Configurar defaults do gcloud

```bash
gcloud config set project "$DEVSHELL_PROJECT_ID"
gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"
```

### Passo 3: Habilitar APIs necessarias

```bash
gcloud services enable \
  iam.googleapis.com \
  compute.googleapis.com \
  bigquery.googleapis.com
```

### Explicacao dos parametros principais

- `gcloud config set compute/region`: define regiao padrao para recursos regionais.
- `gcloud config set compute/zone`: define zona padrao para comandos de VM.
- `gcloud services enable`: garante que APIs usadas no lab estao ativas.

### Resultado esperado

- Projeto, regiao e zona definidos no contexto atual do `gcloud`.
- APIs IAM, Compute Engine e BigQuery habilitadas.

---

## TAREFA 1: Criar e gerenciar service account

### Conceito

Service account e a identidade de workload (app/VM) no Google Cloud. Ao vincular papeis IAM a ela, voce controla quais APIs e recursos essa workload pode acessar.

### Passo 1: Criar a service account `my-sa-123`

```bash
gcloud iam service-accounts create "$SA_TASK1_NAME" \
  --display-name="my service account"
```

### Passo 2: Conceder papel `roles/editor` no projeto

```bash
gcloud projects add-iam-policy-binding "$DEVSHELL_PROJECT_ID" \
  --member="serviceAccount:${SA_TASK1_EMAIL}" \
  --role="roles/editor"
```

### Explicacao dos parametros principais

- `iam service-accounts create`: cria uma identidade gerenciada pelo usuario.
- `--display-name`: nome amigavel exibido no Console.
- `projects add-iam-policy-binding`: adiciona um binding IAM no nivel de projeto.
- `--member=serviceAccount:...`: principal que recebera o papel.
- `--role=roles/editor`: papel predefinido com permissoes amplas no projeto.

### Resultado esperado

- A service account `my-sa-123` existe no projeto.
- Ela aparece como membro com papel `roles/editor` na politica IAM do projeto.

### Validacao rapida da tarefa

```bash
gcloud iam service-accounts list \
  --filter="email:${SA_TASK1_EMAIL}" \
  --format="table(email,displayName,disabled)"

gcloud projects get-iam-policy "$DEVSHELL_PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:${SA_TASK1_EMAIL}" \
  --format="table(bindings.role,bindings.members)"
```

---

## TAREFA 2: Consultar BigQuery com service account em VM

### Conceito

Nesta etapa, voce cria uma service account dedicada para BigQuery, associa papeis minimos para consulta e executa um script Python em uma VM que usa essa identidade.

### Passo 1: Criar service account `bigquery-qwiklab`

```bash
gcloud iam service-accounts create "$SA_BQ_NAME" \
  --display-name="bigquery-qwiklab"
```

### Passo 2: Conceder papeis de BigQuery

```bash
gcloud projects add-iam-policy-binding "$DEVSHELL_PROJECT_ID" \
  --member="serviceAccount:${SA_BQ_EMAIL}" \
  --role="roles/bigquery.dataViewer"

gcloud projects add-iam-policy-binding "$DEVSHELL_PROJECT_ID" \
  --member="serviceAccount:${SA_BQ_EMAIL}" \
  --role="roles/bigquery.user"
```

### Explicacao dos parametros principais

- `roles/bigquery.dataViewer`: permite ler dados/datasets.
- `roles/bigquery.user`: permite executar jobs de consulta.
- Dois bindings separados deixam claro o principio de menor privilegio por papel.

### Passo 3: Criar VM `bigquery-instance` com essa service account

```bash
gcloud compute instances create "$VM_NAME" \
  --zone="$ZONE" \
  --machine-type="e2-medium" \
  --image-family="debian-12" \
  --image-project="debian-cloud" \
  --service-account="$SA_BQ_EMAIL" \
  --scopes="https://www.googleapis.com/auth/bigquery"
```

### Explicacao dos parametros principais

- `--machine-type=e2-medium`: tipo de maquina solicitado no lab.
- `--image-family=debian-12`: usa Debian GNU/Linux 12.
- `--service-account`: identidade anexada a VM.
- `--scopes=.../auth/bigquery`: libera escopo de acesso BigQuery para metadata credentials da VM.

### Passo 4: Preparar ambiente Python dentro da VM

```bash
gcloud compute ssh "$VM_NAME" --zone="$ZONE" --command '
  sudo apt-get update &&
  sudo apt-get install -y git python3 python3-pip python3.11-venv &&
  python3 -m venv myvenv &&
  . myvenv/bin/activate &&
  pip3 install --upgrade pip &&
  pip3 install google-cloud-bigquery pyarrow pandas db-dtypes
'
```

### Passo 5: Criar e executar o script `query.py` na VM

```bash
gcloud compute ssh "$VM_NAME" --zone="$ZONE" --command "cat > query.py <<'PYEOF'
from google.auth import compute_engine
from google.cloud import bigquery

credentials = compute_engine.Credentials(
    service_account_email='${SA_BQ_EMAIL}')

query = '''
SELECT
  year,
  COUNT(1) AS num_babies
FROM publicdata.samples.natality
WHERE year > 2000
GROUP BY year
ORDER BY year
'''

client = bigquery.Client(
    project='${PROJECT_ID}',
    credentials=credentials)

print(client.query(query).to_dataframe())
PYEOF"

# myvenv/bin/python3 query.py
```

### Resultado esperado

- A VM executa consulta no dataset publico `publicdata.samples.natality`.
- O output mostra linhas com `year` e `num_babies` para anos maiores que 2000.

---

## Validacao e Testes

### 1) Validar service accounts criadas

```bash
gcloud iam service-accounts list \
  --filter="email~'(${SA_TASK1_NAME}|${SA_BQ_NAME})@${PROJECT_ID}.iam.gserviceaccount.com'" \
  --format="table(email,displayName)"
```

### 2) Validar papeis da `bigquery-qwiklab`

```bash
gcloud projects get-iam-policy "$DEVSHELL_PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:${SA_BQ_EMAIL}" \
  --format="table(bindings.role,bindings.members)"
```

### 3) Validar service account anexada a VM

```bash
gcloud compute instances describe "$VM_NAME" --zone="$ZONE" \
  --format="value(serviceAccounts.email)"
```

### 4) Validar consulta BigQuery novamente

```bash
gcloud compute ssh "$VM_NAME" --zone="$ZONE" --command "python3 query.py"
```

---

## Troubleshooting

### Erro de permissao no BigQuery (403)

- Confirme se `roles/bigquery.dataViewer` e `roles/bigquery.user` estao vinculados a `bigquery-qwiklab`.
- Aguarde propagacao IAM por alguns minutos e rode novamente.

### VM criada sem service account correta

- Valide com `gcloud compute instances describe`.
- Se necessario, recrie a VM com `--service-account="${SA_BQ_EMAIL}"`.

### Script nao encontra bibliotecas Python

- Reative o venv: `. myvenv/bin/activate`.
- Reinstale pacotes: `pip3 install google-cloud-bigquery pyarrow pandas db-dtypes`.

---

## Limpeza (opcional)

```bash
gcloud compute instances delete "$VM_NAME" --zone="$ZONE" --quiet

gcloud iam service-accounts delete "$SA_TASK1_EMAIL" --quiet
gcloud iam service-accounts delete "$SA_BQ_EMAIL" --quiet
```

---

## Conceitos-chave

- Service account representa identidade de workload, nao de usuario humano.
- IAM Roles definem permissoes; preferir papeis predefinidos e escopo minimo.
- Em Compute Engine, acesso a APIs depende de:
  - papel IAM da service account;
  - escopos configurados na VM.
- Para BigQuery em VM, combinar `roles/bigquery.user` + `roles/bigquery.dataViewer` e escopo BigQuery.

---

## Fluxo Final

1. Definir projeto/regiao/zona e habilitar APIs.
2. Criar `my-sa-123` e conceder `roles/editor`.
3. Criar `bigquery-qwiklab` e conceder papeis de BigQuery.
4. Criar `bigquery-instance` com Debian 12, `e2-medium`, e service account `bigquery-qwiklab`.
5. Instalar dependencias Python na VM.
6. Executar `query.py` para consultar dataset publico do BigQuery.
7. Validar IAM, VM e resultado da consulta.
