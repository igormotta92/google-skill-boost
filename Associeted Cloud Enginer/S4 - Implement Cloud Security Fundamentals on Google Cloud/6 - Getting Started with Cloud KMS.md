# Getting Started with Cloud KMS

## Visao Geral

Neste lab, voce vai explorar recursos avancados das APIs de seguranca e privacidade do Google Cloud. Voce comeca configurando um bucket seguro no Cloud Storage e depois segue para o gerenciamento de chaves de criptografia e protecao de dados com o Cloud Key Management Service. Alem disso, voce analisa os logs de auditoria do Cloud Storage para monitorar acessos e modificacoes. Ao longo do lab, voce processa dados financeiros resumidos, criptografando-os para garantir confidencialidade antes do upload para o Cloud Storage.

## Introducao

Este guia converte o lab para execucao via CLI, cobrindo criacao de bucket no Cloud Storage, uso do Cloud KMS para criptografar dados, configuracao de IAM no KeyRing e validacao com Cloud Audit Logs.

Objetivo tecnico:
- criar e validar um bucket de destino;
- criar `KeyRing` e `CryptoKey` no KMS;
- criptografar arquivos via API do KMS;
- fazer upload dos artefatos criptografados no Cloud Storage;
- verificar permissoes e auditoria das acoes.

---

## Pre-requisitos e Variaveis

### Conceito

Padronizar variaveis reduz erro de digitacao e facilita reaproveitar comandos.

### Passo 1: Definir variaveis

```bash
# Projeto ativo
export PROJECT_ID="$(gcloud config get-value project)"

# Bucket do lab (ajuste o prefixo conforme ambiente)
export BUCKET_NAME="$PROJECT_ID-kms-lab"

# Dataset de origem disponibilizado pelo lab
export SOURCE_BUCKET="${PROJECT_ID}-kms-lab-data"

# KMS
export LOCATION="global"
export KEYRING_NAME="labkey"
export CRYPTOKEY_NAME="qwiklab"

# Diretorio de trabalho
export WORKDIR="finance-dept"
```

### Passo 2: Validar autenticacao e projeto

```bash
gcloud auth list
gcloud config list project
```

**Explicacao dos parametros:**
- `gcloud auth list`: mostra a conta autenticada.
- `gcloud config list project`: confirma em qual projeto os recursos serao criados.

**Resultado esperado:**
- conta ativa disponivel e `PROJECT_ID` correto antes de provisionar recursos.

---

## TAREFA 1: Criar bucket no Cloud Storage

### Conceito

O bucket sera o destino dos arquivos criptografados produzidos no lab.

### Passo 1: Criar bucket

```bash
gsutil mb "gs://${BUCKET_NAME}"
```

**Explicacao dos parametros:**
- `gsutil mb`: cria bucket.
- `gs://${BUCKET_NAME}`: nome globalmente unico do bucket.

### Passo 2: Validar criacao

```bash
gsutil ls -b "gs://${BUCKET_NAME}"
```

**Resultado esperado:**
- bucket listado com metadados basicos.

---

## TAREFA 2: Revisar os dados de origem

### Conceito

Antes de criptografar, valide que o arquivo de entrada esta em texto puro.

### Passo 1: Copiar um arquivo de exemplo

```bash
gsutil cp "gs://${SOURCE_BUCKET}/finance-dept/inbox/1.txt" .
```

### Passo 2: Inspecionar o conteudo

```bash
tail -n 5 1.txt
```

**Resultado esperado:**
- visualizar texto semelhante a `This is a sample financial document for encryption`.

---

## TAREFA 3: Habilitar e validar Cloud KMS API

### Conceito

A API `cloudkms.googleapis.com` precisa estar ativa para criar e usar chaves.

### Passo 1: Habilitar servico

```bash
gcloud services enable cloudkms.googleapis.com
```

### Passo 2: Confirmar status

```bash
gcloud services list --enabled --filter="name:cloudkms.googleapis.com"
```

**Resultado esperado:**
- servico `cloudkms.googleapis.com` habilitado no projeto.

---

## TAREFA 4: Criar KeyRing e CryptoKey

### Conceito

`KeyRing` organiza chaves; `CryptoKey` e o recurso usado para criptografar/decriptografar dados.

### Passo 1: Criar KeyRing

```bash
gcloud kms keyrings create "${KEYRING_NAME}" \
  --location="${LOCATION}"
```

### Passo 2: Criar CryptoKey

```bash
gcloud kms keys create "${CRYPTOKEY_NAME}" \
  --location="${LOCATION}" \
  --keyring="${KEYRING_NAME}" \
  --purpose="encryption"
```

**Explicacao dos parametros principais:**
- `--location=global`: escopo global do KMS neste lab.
- `--keyring`: KeyRing pai da chave.
- `--purpose=encryption`: chave simetrica para encrypt/decrypt.

### Passo 3: Validar recursos

```bash
gcloud kms keyrings list --location="${LOCATION}"
gcloud kms keys list --location="${LOCATION}" --keyring="${KEYRING_NAME}"
```

**Resultado esperado:**
- `labkey` e `qwiklab` disponiveis.

---

## TAREFA 5: Criptografar arquivo e enviar para o bucket

### Conceito

O endpoint de `encrypt` do KMS recebe texto em base64 e retorna `ciphertext`.

### Passo 1: Converter plaintext para base64

```bash
PLAINTEXT="$(base64 -w0 1.txt)"
```

Nota:
- em ambientes sem `-w0`, ajuste conforme ambiente usando alternativa equivalente para remover quebra de linha.
- no fluxo CLI do `gcloud kms encrypt`, esta conversao nao e necessaria, pois o comando aceita `--plaintext-file` diretamente.

### Passo 2: Chamar endpoint encrypt (teste rapido)

```bash

# rest
curl -s \
  "https://cloudkms.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/keyRings/${KEYRING_NAME}/cryptoKeys/${CRYPTOKEY_NAME}:encrypt" \
  -d "{\"plaintext\":\"${PLAINTEXT}\"}" \
  -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
  -H "Content-Type: application/json" | jq .

# cli
gcloud kms encrypt \
  --location="${LOCATION}" \
  --keyring="${KEYRING_NAME}" \
  --key="${CRYPTOKEY_NAME}" \
  --plaintext-file="1.txt" \
  --ciphertext-file="1.encrypted"

# Nao ha equivalente exato no gcloud para retornar o JSON do endpoint :encrypt em stdout.
# A CLI grava o resultado no arquivo definido em --ciphertext-file.
```

### Passo 3: Salvar ciphertext em arquivo

```bash
# rest
curl -s \
  "https://cloudkms.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/keyRings/${KEYRING_NAME}/cryptoKeys/${CRYPTOKEY_NAME}:encrypt" \
  -d "{\"plaintext\":\"${PLAINTEXT}\"}" \
  -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
  -H "Content-Type: application/json" \
  | jq -r .ciphertext > 1.encrypted

# cli
gcloud kms encrypt \
  --location="${LOCATION}" \
  --keyring="${KEYRING_NAME}" \
  --key="${CRYPTOKEY_NAME}" \
  --plaintext-file="1.txt" \
  --ciphertext-file="1.encrypted"
```

### Passo 4: Decriptar para validacao

```bash
# rest
curl -s \
  "https://cloudkms.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/keyRings/${KEYRING_NAME}/cryptoKeys/${CRYPTOKEY_NAME}:decrypt" \
  -d "{\"ciphertext\":\"$(cat 1.encrypted)\"}" \
  -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
  -H "Content-Type: application/json" \
  | jq -r .plaintext | base64 -d

# cli
gcloud kms decrypt \
  --location="${LOCATION}" \
  --keyring="${KEYRING_NAME}" \
  --key="${CRYPTOKEY_NAME}" \
  --ciphertext-file="1.encrypted" \
  --plaintext-file="1.decrypted"

cat 1.decrypted
```

### Passo 5: Enviar arquivo criptografado ao bucket

```bash
gsutil cp 1.encrypted "gs://${BUCKET_NAME}/"
```

**Resultado esperado:**
- `1.encrypted` presente no bucket e decriptacao retornando o conteudo original.

---

## TAREFA 6: Configurar IAM no KeyRing

### Conceito

Neste lab, voce aplica dois papeis no KeyRing para o usuario atual:
- `roles/cloudkms.admin` (gerenciar recursos KMS)
- `roles/cloudkms.cryptoKeyEncrypterDecrypter` (usar chave para encrypt/decrypt)

### Passo 1: Identificar usuario autenticado

```bash
export USER_EMAIL="$(gcloud auth list --filter=status:ACTIVE --format='value(account)' | head -n1)"
echo "${USER_EMAIL}"
```

### Passo 2: Conceder permissao administrativa

```bash
gcloud kms keyrings add-iam-policy-binding "${KEYRING_NAME}" \
  --location="${LOCATION}" \
  --member="user:${USER_EMAIL}" \
  --role="roles/cloudkms.admin"
```

### Passo 3: Conceder permissao de uso criptografico

```bash
gcloud kms keyrings add-iam-policy-binding "${KEYRING_NAME}" \
  --location="${LOCATION}" \
  --member="user:${USER_EMAIL}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

### Passo 4: Validar policy

```bash
gcloud kms keyrings get-iam-policy "${KEYRING_NAME}" \
  --location="${LOCATION}" \
  --format="yaml(bindings)"
```

**Resultado esperado:**
- usuario presente nos dois bindings esperados.

---

## TAREFA 7: Criptografar multiplos arquivos e enviar ao Cloud Storage

### Conceito

Automatizar o processo de criptografia para todos os arquivos do diretorio `finance-dept`.

### Passo 1: Baixar diretorio de origem

```bash
gsutil -m cp -r "gs://${SOURCE_BUCKET}/finance-dept" .
```

### Passo 2: Executar loop de criptografia

```bash
# rest
MYDIR="${WORKDIR}"

find "${MYDIR}" -type f -not -name "*.encrypted" | while IFS= read -r file; do
  PLAINTEXT="$(base64 -w0 "${file}")"
  curl -s \
    "https://cloudkms.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/keyRings/${KEYRING_NAME}/cryptoKeys/${CRYPTOKEY_NAME}:encrypt" \
    -d "{\"plaintext\":\"${PLAINTEXT}\"}" \
    -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
    -H "Content-Type: application/json" \
    | jq -r .ciphertext > "${file}.encrypted"
done

# cli
MYDIR="${WORKDIR}"

find "${MYDIR}" -type f -not -name "*.encrypted" | while IFS= read -r file; do
  gcloud kms encrypt \
    --location="${LOCATION}" \
    --keyring="${KEYRING_NAME}" \
    --key="${CRYPTOKEY_NAME}" \
    --plaintext-file="${file}" \
    --ciphertext-file="${file}.encrypted"
done

# Nao ha equivalente exato no gcloud para capturar apenas o campo ciphertext em stdout,
# pois a CLI grava o resultado diretamente no arquivo --ciphertext-file.
```

### Passo 3: Enviar arquivos criptografados

```bash
gsutil -m cp finance-dept/inbox/*.encrypted "gs://${BUCKET_NAME}/finance-dept/inbox/"
```

### Passo 4: Validar upload

```bash
gsutil ls "gs://${BUCKET_NAME}/finance-dept/inbox/"
```

**Resultado esperado:**
- multiplos arquivos `.encrypted` no caminho `finance-dept/inbox` do bucket.

---

## TAREFA 8: Consultar Cloud Audit Logs (KMS)

### Conceito

Cloud Audit Logs registra quem fez o que, quando e em qual recurso.

### Passo 1: Buscar eventos de KMS no projeto

```bash
gcloud logging read \
  'resource.type="cloudkms_key_ring"' \
  --limit=20 \
  --format='table(timestamp,protoPayload.methodName,protoPayload.authenticationInfo.principalEmail,resource.labels.key_ring_id)'
```

### Passo 2: Filtrar eventos no KeyRing do lab

```bash
gcloud logging read \
  "resource.type=\"cloudkms_key_ring\" AND resource.labels.key_ring_id=\"${KEYRING_NAME}\"" \
  --limit=20 \
  --format='table(timestamp,protoPayload.methodName,protoPayload.resourceName)'
```

**Resultado esperado:**
- eventos de criacao/alteracao/uso do `labkey` visiveis no Log Explorer via CLI.

---

## Validacao Final

Execute os comandos abaixo para confirmar que todos os objetivos foram atendidos:

```bash
# Bucket criado
gsutil ls -b "gs://${BUCKET_NAME}"

# KeyRing/CryptoKey criados
gcloud kms keyrings list --location="${LOCATION}" --filter="name:${KEYRING_NAME}"
gcloud kms keys list --location="${LOCATION}" --keyring="${KEYRING_NAME}" --filter="name:${CRYPTOKEY_NAME}"

# Arquivos criptografados no bucket
gsutil ls "gs://${BUCKET_NAME}/"
gsutil ls "gs://${BUCKET_NAME}/finance-dept/inbox/"

# IAM aplicado no KeyRing
gcloud kms keyrings get-iam-policy "${KEYRING_NAME}" --location="${LOCATION}"

# Logs de auditoria KMS
gcloud logging read "resource.type=\"cloudkms_key_ring\"" --limit=5
```

---

## Troubleshooting (CLI)

- Erro `403` ao chamar `encrypt`/`decrypt`:
  confirme papeis IAM com `get-iam-policy` e conta ativa com `gcloud auth list`.
- Erro `PERMISSION_DENIED` no token ADC:
  rode `gcloud auth application-default login` (ajuste conforme politica do ambiente).
- Bucket name em uso:
  altere `BUCKET_NAME` para um valor globalmente unico.
- `base64 -w0` indisponivel fora do Cloud Shell:
  ajuste comando de base64 para remover quebras de linha conforme shell local.

---

## Limpeza Opcional

Nota importante:
- no Cloud KMS, `KeyRing` e `CryptoKey` nao sao removidos diretamente; use rotacao, desabilitacao e destruicao de versoes quando aplicavel.

Comandos de limpeza parcial:

```bash
# Remover objetos do bucket
gsutil -m rm -r "gs://${BUCKET_NAME}/**"

# Remover bucket (se vazio)
gsutil rb "gs://${BUCKET_NAME}"
```

---

## Conceitos-Chave

- Cloud KMS separa gerenciamento de chave (admin) de uso criptografico (encrypt/decrypt).
- Criptografia via API retorna `ciphertext` nao deterministico para o mesmo plaintext.
- IAM em nivel de KeyRing pode ser herdado pelos CryptoKeys filhos.
- Cloud Audit Logs permite rastrear operacoes administrativas e de acesso aos recursos KMS.

---

## Fluxo Final

1. Definir variaveis e validar autenticacao/projeto.
2. Criar bucket de destino.
3. Habilitar Cloud KMS API.
4. Criar `labkey` e `qwiklab`.
5. Criptografar arquivo de teste, validar decriptacao e enviar ao bucket.
6. Conceder IAM (`cloudkms.admin` e `cloudkms.cryptoKeyEncrypterDecrypter`).
7. Criptografar lote de arquivos e enviar ao bucket.
8. Consultar logs de auditoria para rastreabilidade.