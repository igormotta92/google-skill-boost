# Build a Serverless App with Cloud Run that Creates PDF Files

## Visao Geral

Ha doze anos, Lily fundou a rede de clinicas veterinarias Pet Theory. Hoje as faturas sao enviadas em DOCX, e muitos clientes nao conseguem abri-las. Para melhorar a satisfacao dos clientes, Lily pede ao Patrick (TI) uma alternativa.

O time de Ops da Pet Theory e uma unica pessoa, entao a solucao precisa ter bom custo e pouca manutencao. Patrick escolhe o **Cloud Run**: serverless, sem gerenciamento de infraestrutura, escala ate zero (sem custo quando ocioso) e aceita binarios customizados dentro de containers.

Neste lab, voce constroi um conversor de PDF no Cloud Run que converte automaticamente arquivos enviados ao Cloud Storage e grava os PDFs em outro bucket.

Objetivos:

- Converter uma aplicacao Node.js em container.
- Gerar containers com Cloud Build.
- Criar um servico Cloud Run que converte arquivos para PDF.
- Usar processamento de eventos com Cloud Storage (notificacao -> Pub/Sub -> Cloud Run).

## Introducao

Este guia converte o lab (GSP644) para execucao via CLI com `gcloud`, `npm` e `curl`, mantendo os nomes de recursos do enunciado. As edicoes de arquivo feitas no Cloud Shell Editor aparecem com o equivalente via `cat`/`npm pkg`.

Recursos usados no lab:
- codigo inicial em `gs://<PROJECT_ID>-assets/pet-theory` (pasta `lab03`);
- imagem `gcr.io/<PROJECT_ID>/pdf-converter` e servico Cloud Run `pdf-converter` (privado, `max-instances=1`);
- buckets `<PROJECT_ID>-upload` e `<PROJECT_ID>-processed`;
- topico `new-doc`, assinatura push `pdf-conv-sub` e SA pre-criada `pubsub-cloud-run-invoker`.

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
export REGION="<REGION_DO_LAB>"   # "Region" / "Lab Region" do enunciado
export PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"

export SERVICE_NAME="pdf-converter"
export IMAGE_URI="gcr.io/${PROJECT_ID}/${SERVICE_NAME}"
export UPLOAD_BUCKET="gs://${PROJECT_ID}-upload"
export PROCESSED_BUCKET="gs://${PROJECT_ID}-processed"
export TOPIC_NAME="new-doc"
export SUBSCRIPTION_NAME="pdf-conv-sub"
export INVOKER_SA="pubsub-cloud-run-invoker@${PROJECT_ID}.iam.gserviceaccount.com"
export APP_DIR="$HOME/pet-theory/lab03"

gcloud config set project "$PROJECT_ID"
gcloud config set run/region "$REGION"
echo "Projeto: $PROJECT_ID ($PROJECT_NUMBER) | Regiao: $REGION"
```

### Explicacao dos parametros
- `GOOGLE_CLOUD_PROJECT`: variavel usada nos comandos do lab (ja existe no Cloud Shell; exportada por garantia).
- `PROJECT_NUMBER`: capturado direto, sem copiar do `gcloud projects list`.
- `run/region`: regiao padrao para os comandos `gcloud run`.

---

## TAREFA 1: Entender a tarefa

### Conceito
A Pet Theory quer converter faturas para PDF automaticamente. A proposta: rodar o **LibreOffice** em um container no Cloud Run, sem manter servidores, disparado por uploads no Cloud Storage.

Sem comandos nesta tarefa.

---

## TAREFA 2: Habilitar a Cloud Run API

### Conceito
A API do Cloud Run (Cloud Run Admin API) precisa estar ativa. Cloud Build e Pub/Sub tambem serao usados.

### Passo 1: Habilitar as APIs

Console (passo do lab): **APIs & Services > Library** > `Cloud Run Admin API` > **Enable**.

Equivalente CLI:

```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  pubsub.googleapis.com

gcloud services list --enabled --filter="name:(run.googleapis.com pubsub.googleapis.com)"
```

### Resultado esperado
- `run.googleapis.com` habilitada.

---

## TAREFA 3: Fazer deploy de um servico Cloud Run simples

### Conceito
O prototipo Node.js aceita `POST`: se for notificacao do Pub/Sub, loga os dados do arquivo; senao, responde `OK`. A imagem e gerada pelo Cloud Build e o servico e publicado **sem acesso anonimo**.

### Passo 1: Baixar o codigo inicial

```bash
cd "$HOME"
gcloud storage cp -r "gs://${PROJECT_ID}-assets/pet-theory" .
cd "$APP_DIR"
ls
```

### Passo 2: Adicionar o script `start` no `package.json`

Console (passo do lab): editar `package.json` e incluir `"start": "node index.js",` em `scripts`.

Equivalente CLI:

```bash
npm pkg set scripts.start="node index.js"
npm pkg get scripts
```

### Passo 3: Instalar as dependencias

```bash
npm install express body-parser child_process @google-cloud/storage
```

### Passo 4: Revisar o codigo e o Dockerfile

```bash
cat index.js
cat Dockerfile
```

- `FROM node...`: imagem base Node.
- `CMD [ "npm", "start" ]`: comando de inicio (usa o script `start`).

### Passo 5: Build da imagem com Cloud Build

```bash
gcloud builds submit --tag "$IMAGE_URI"

gcloud artifacts docker images list "gcr.io/${PROJECT_ID}" --include-tags \
  --format="table(package,tags)" | grep "$SERVICE_NAME"
```

> Click **Check my progress**: *Build simple a REST API*.

### Passo 6: Deploy no Cloud Run (privado)

```bash
gcloud run deploy "$SERVICE_NAME" \
  --image "$IMAGE_URI" \
  --region "$REGION" \
  --no-allow-unauthenticated \
  --max-instances=1

export SERVICE_URL="$(gcloud run services describe "$SERVICE_NAME" --region "$REGION" --format='value(status.url)')"
echo "$SERVICE_URL"
```

> Click **Check my progress**: *Create a Revision for Cloud Run*.

### Passo 7: Testar acesso anonimo x autenticado

```bash
# Anonimo: deve ser negado
curl -s -X POST "$SERVICE_URL" | head -n 5

# Autenticado: deve responder OK
curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" "$SERVICE_URL"
```

### Explicacao dos parametros
- `npm pkg set scripts.start=...`: edita o `package.json` sem abrir o editor.
- `gcloud builds submit --tag`: build remoto e push da imagem para o `gcr.io` (Artifact Registry).
- `--no-allow-unauthenticated`: exige token de identidade com `roles/run.invoker`.
- `--max-instances=1`: limita o numero de instancias.
- `gcloud auth print-identity-token`: token OIDC do usuario para chamar o servico privado.

### Resultado esperado
- Anonimo: `Your client does not have permission to get the URL`.
- Autenticado: `OK`.

### Resumo visual

```text
[lab03: index.js + Dockerfile]
        |
        | gcloud builds submit
        v
[gcr.io/PROJECT_ID/pdf-converter]
        |
        | gcloud run deploy (--no-allow-unauthenticated, max 1)
        v
[Cloud Run pdf-converter] (REGION) -- SERVICE_URL
        ^                    ^
        | curl anonimo: 403  | curl + identity token: OK
```

---

## TAREFA 4: Disparar o Cloud Run no upload de arquivos

### Conceito
Cadeia de eventos: upload no bucket `-upload` -> notificacao `OBJECT_FINALIZE` no topico `new-doc` -> assinatura **push** autenticada -> `POST` no Cloud Run. A SA `pubsub-cloud-run-invoker` (pre-criada) assina o token do push e precisa de `run.invoker` no servico.

### Passo 1: Criar os buckets

```bash
gcloud storage buckets create "$UPLOAD_BUCKET"
gcloud storage buckets create "$PROCESSED_BUCKET"
gcloud storage ls | grep -E -- "-(upload|processed)/"
```

> Click **Check my progress**: *Create two cloud storage buckets*.

### Passo 2: Notificacao do bucket para o Pub/Sub

```bash
gcloud storage buckets notifications create \
  -t "$TOPIC_NAME" -f json -e OBJECT_FINALIZE "$UPLOAD_BUCKET"

gcloud storage buckets notifications list "$UPLOAD_BUCKET"
```

> Click **Check my progress**: *Create a Pub/Sub topic for handling notifications from storage bucket*.

### Passo 3: Autorizar a SA a invocar o servico

```bash
gcloud iam service-accounts describe "$INVOKER_SA" --format="value(email)"

gcloud run services add-iam-policy-binding "$SERVICE_NAME" \
  --member="serviceAccount:${INVOKER_SA}" \
  --role=roles/run.invoker \
  --region "$REGION"
```

### Passo 4: Garantir o service agent do Pub/Sub

```bash
gcloud services enable pubsub.googleapis.com --project="$PROJECT_ID"

gcloud beta services identity create \
  --service=pubsub.googleapis.com \
  --project="$PROJECT_ID"

# Ja pre-configurado no lab: service agent com Token Creator
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.role:roles/iam.serviceAccountTokenCreator" \
  --format="table(bindings.members)"
```

### Passo 5: Criar a assinatura push

```bash
gcloud pubsub subscriptions create "$SUBSCRIPTION_NAME" \
  --topic "$TOPIC_NAME" \
  --push-endpoint="$SERVICE_URL" \
  --push-auth-service-account="$INVOKER_SA"
```

### Explicacao dos parametros
- `notifications create -t new-doc -f json -e OBJECT_FINALIZE`: cria (se preciso) o topico `new-doc` e publica um JSON a cada upload concluido.
- `add-iam-policy-binding --role=roles/run.invoker`: permite a SA chamar o servico privado.
- `services identity create`: cria o service agent `service-PROJECT_NUMBER@gcp-sa-pubsub...`.
- `serviceAccountTokenCreator`: permite ao Pub/Sub gerar tokens OIDC em nome da SA invoker.
- `--push-endpoint` + `--push-auth-service-account`: push autenticado para o Cloud Run.

### Resultado esperado
- Buckets criados, notificacao ativa no `-upload` e assinatura `pdf-conv-sub` apontando para o `SERVICE_URL`.

> Click **Check my progress**: *Create a Pub/Sub subscription*.

### Resumo visual

```text
[gs://PROJECT_ID-upload] -- OBJECT_FINALIZE --> [Topic new-doc]
                                                      |
                                                      | push pdf-conv-sub
                                                      | OIDC: invoker SA
                                                      v
                                     [Cloud Run pdf-converter] (run.invoker)

[gs://PROJECT_ID-processed]  (destino dos PDFs na Tarefa 6)
```

---

## TAREFA 5: Verificar que os uploads disparam o Cloud Run

### Conceito
Nesta versao o servico apenas **loga** o metadado recebido (`file: {...}`). O teste confirma a cadeia de eventos.

### Passo 1: Enviar arquivos de teste

```bash
gcloud storage cp "gs://spls/gsp644/*" "$UPLOAD_BUCKET"
```

### Passo 2: Conferir os logs

Console (passo do lab): **Logging** > **All resources: Cloud Run Revision** > **Run query** > entradas que comecam com `file:`.

Equivalente CLI:

```bash
gcloud logging read \
  "resource.type=\"cloud_run_revision\" AND resource.labels.service_name=\"${SERVICE_NAME}\" AND textPayload:\"file:\"" \
  --limit=5 --freshness=10m --format="value(textPayload)"
```

### Passo 3: Limpar o bucket de upload

```bash
gcloud storage rm "${UPLOAD_BUCKET}/*"
```

### Resultado esperado
- Logs `file:` com o campo `name` de cada arquivo enviado.

---

## TAREFA 6: Atualizar o container com LibreOffice

### Conceito
A versao final baixa o arquivo para `/tmp`, converte com `libreoffice --headless --convert-to pdf`, envia o PDF para o bucket definido em `PDF_BUCKET` e apaga o original. O container precisa ter o LibreOffice e mais memoria (`2Gi`).

### Passo 1: Adicionar o LibreOffice ao Dockerfile

Mantendo a imagem base atual do `FROM` (o enunciado mostra `node_docker_image` como placeholder):

```bash
cd "$APP_DIR"
export BASE_IMAGE="$(grep -m1 '^FROM' Dockerfile | awk '{print $2}')"
echo "$BASE_IMAGE"

cat > Dockerfile <<EOF
FROM ${BASE_IMAGE}
RUN apt-get update -y \\
    && apt-get install -y libreoffice \\
    && apt-get clean
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
EOF

cat Dockerfile
```

### Passo 2: Substituir o `index.js` pelo codigo completo do lab

O lab recomenda substituir todo o arquivo para evitar erros de formatacao. Como o codigo tem `${...}` e crases, o heredoc usa `<<'EOF'` (sem expansao):

```bash
cat > index.js <<'EOF'
const { promisify } = require("util");
const { Storage } = require("@google-cloud/storage");
const exec = promisify(require("child_process").exec);
const storage = new Storage();
const express = require("express");
const bodyParser = require("body-parser");
const app = express();

app.use(bodyParser.json());

const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log("Listening on port", port);
});

app.post("/", async (req, res) => {
  try {
    const file = decodeBase64Json(req.body.message.data);
    await downloadFile(file.bucket, file.name);
    const pdfFileName = await convertFile(file.name);
    await uploadFile(process.env.PDF_BUCKET, pdfFileName);
    await deleteFile(file.bucket, file.name);
  } catch (ex) {
    console.log(`Error: ${ex}`);
  }
  res.set("Content-Type", "text/plain");
  res.send("\n\nOK\n\n");
});

function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, "base64").toString());
}

// Helper function to check file existence (using fs.promises for async)
async function fileExists(filePath) {
  try {
    await fs.promises.access(filePath); // Throws an error if the file doesn't exist
    return true;
  } catch (err) {
    return false;
  }
}
async function downloadFile(bucketName, fileName) {
  // 1. Check if the file exists
  const fileExistsLocally = await fileExists(`/tmp/${fileName}`);

  // 2. Delete if present
  if (fileExistsLocally) {
    console.log(`File exists locally. Deleting: ${fileName}`);
    await fs.promises.unlink(`/tmp/${fileName}`); // Use fs.promises for async file operations
    console.log(`File deleted.`);
  } else {
    console.log(`File does not exist locally: ${fileName}`);
  }

  // 3. Download from the storage bucket
  const options = { destination: `/tmp/${fileName}` };
  await storage.bucket(bucketName).file(fileName).download(options);
  console.log(`File downloaded: ${fileName}`);
}

async function convertFile(fileName) {
  const cmd =
    "libreoffice --headless --convert-to pdf --outdir /tmp " +
    `"/tmp/${fileName}"`;
  console.log(cmd);
  const { stdout, stderr } = await exec(cmd);
  if (stderr) {
    console.log(`Conversion Failed: ${stderr}`);
    throw stderr;
  }
  console.log(`Conversion Success: ${stdout}`);
  pdfFileName = fileName.replace(/\.\w+$/, ".pdf");
  return pdfFileName;
}

async function deleteFile(bucketName, fileName) {
  await storage.bucket(bucketName).file(fileName).delete();
}

async function uploadFile(bucketName, fileName) {
  await storage.bucket(bucketName).upload(`/tmp/${fileName}`);
}
EOF

node --check index.js && echo "index.js ok"
```

Observacao: o codigo do lab usa `fs` sem `require("fs")`. Na pratica, o `fileExists` cai no `catch` e retorna `false`, e o fluxo segue normalmente. Mantido igual ao lab.

### Passo 3: Novo build

```bash
gcloud builds submit --tag "$IMAGE_URI"
```

> Click **Check my progress**: *Create another build for REST API*.

### Passo 4: Deploy da nova revisao com memoria e `PDF_BUCKET`

```bash
gcloud run deploy "$SERVICE_NAME" \
  --image "$IMAGE_URI" \
  --region "$REGION" \
  --memory=2Gi \
  --no-allow-unauthenticated \
  --max-instances=1 \
  --set-env-vars PDF_BUCKET="${PROJECT_ID}-processed"
```

O deploy leva alguns minutos (imagem maior por causa do LibreOffice).

### Explicacao dos parametros
- `RUN apt-get ... libreoffice`: instala o conversor na imagem.
- `\\` no heredoc sem aspas: grava `\` literal no Dockerfile (continuacao de linha).
- `<<'EOF'` no `index.js`: preserva `${...}` e crases do JavaScript.
- `--memory=2Gi`: memoria suficiente para o LibreOffice.
- `--set-env-vars PDF_BUCKET=...`: bucket de destino dos PDFs (nome sem `gs://`).

### Resultado esperado
- Nova revisao do `pdf-converter` servindo 100% do trafego, com `PDF_BUCKET` definido.

> Click **Check my progress**: *Create a new Revision*.

### Resumo visual

```text
Antes:  [pdf-converter v1] POST do Pub/Sub -> console.log("file: ...")

        Dockerfile + libreoffice | index.js completo | --memory=2Gi
        --set-env-vars PDF_BUCKET=PROJECT_ID-processed

Depois: [pdf-converter v2]
          download -> /tmp -> libreoffice --convert-to pdf
          -> upload PDF no -processed -> delete original no -upload
```

---

## TAREFA 7: Testar o servico de conversao

### Conceito
Teste ponta a ponta: arquivos enviados ao `-upload` sao convertidos e aparecem como PDF no `-processed`, enquanto os originais sao apagados.

### Passo 1: Conferir o servico

```bash
curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" "$SERVICE_URL"
```

### Passo 2: Criar o script de upload (comando do lab)

```bash
cat <<'EOF' > copy_files.sh
#!/bin/bash

SOURCE_BUCKET="gs://spls/gsp644"
DESTINATION_BUCKET="gs://${GOOGLE_CLOUD_PROJECT}-upload"  # Replace with your actual bucket name
DELAY=5

# Get a list of files in the source bucket
files=$(gcloud storage ls "$SOURCE_BUCKET")

# Loop through the files
for file in $files; do
  # Construct the full path of the source file
  source_file_path="$file"

  # Copy the file to the destination bucket
  gcloud storage cp "$source_file_path" "$DESTINATION_BUCKET"

  # Check if the copy was successful
  if [ $? -eq 0 ]; then  # $? is the exit status of the previous command
    echo "Copied: $source_file_path to $DESTINATION_BUCKET"
  else
    echo "Failed to copy: $source_file_path"
  fi

  # Sleep for 5 seconds
  sleep $DELAY
done

echo "All files copied!"
EOF
```

### Passo 3: Enviar os arquivos e acompanhar

```bash
bash copy_files.sh

# Originais sumindo do -upload
gcloud storage ls "$UPLOAD_BUCKET"

# PDFs chegando no -processed
sleep 60
gcloud storage ls "$PROCESSED_BUCKET"
```

### Explicacao dos parametros
- `<<'EOF'`: o script e gravado sem expansao; `${GOOGLE_CLOUD_PROJECT}` e resolvido quando o script roda.
- `DELAY=5`: espaca os uploads para o servico (1 instancia) processar um por vez.

### Resultado esperado
- Bucket `-upload` esvaziando e bucket `-processed` com um PDF para cada arquivo (DOCX, XLSX, JPG, PNG, GIF...). Se faltar algum, rode o script novamente.

---

## Validacao

```bash
gcloud run services describe "$SERVICE_NAME" --region "$REGION" \
  --format="yaml(status.url,spec.template.spec.containers[0].resources.limits,spec.template.spec.containers[0].env)"

gcloud run services get-iam-policy "$SERVICE_NAME" --region "$REGION"

gcloud storage buckets notifications list "$UPLOAD_BUCKET"
gcloud pubsub subscriptions describe "$SUBSCRIPTION_NAME" \
  --format="yaml(topic,pushConfig.pushEndpoint,pushConfig.oidcToken.serviceAccountEmail)"

gcloud storage ls "$PROCESSED_BUCKET"
```

Checklist esperado:
- `pdf-converter` com `memory: 2Gi`, `PDF_BUCKET` definido e acesso nao anonimo;
- `pubsub-cloud-run-invoker` com `roles/run.invoker` no servico;
- notificacao `OBJECT_FINALIZE` -> `new-doc` e assinatura push `pdf-conv-sub` para o `SERVICE_URL`;
- PDFs no bucket `-processed`.

---

## Resumo Visual da Arquitetura

Compilado: estado final ao termino do lab. Os detalhes de cada etapa estao no `### Resumo visual` das Tarefas 3, 4 e 6.

```text
[Usuario / copy_files.sh]
        |
        v
[gs://PROJECT_ID-upload] -- OBJECT_FINALIZE --> [Topic new-doc]
        ^                                             |
        | delete original                             | push pdf-conv-sub (OIDC)
        |                                             v
        +---------------- [Cloud Run pdf-converter] (REGION, 2Gi, max 1)
                           LibreOffice: /tmp -> PDF
                                  |
                                  | upload PDF (PDF_BUCKET)
                                  v
                       [gs://PROJECT_ID-processed]
```

---

## Troubleshooting

- `Your client does not have permission` no `curl` autenticado:
  - confira o token (`gcloud auth print-identity-token`) e se a conta tem `run.invoker` (o dono do projeto ja tem).
- Nenhum log `file:` na Tarefa 5:
  - confira a notificacao do bucket, a assinatura `pdf-conv-sub` (endpoint = `SERVICE_URL`) e o binding `run.invoker` da SA; clique em **Load newer logs**.
- Assinatura recebendo `403` no push:
  - falta `roles/run.invoker` para `pubsub-cloud-run-invoker` ou o service agent do Pub/Sub nao tem `serviceAccountTokenCreator`.
- PDFs nao aparecem no `-processed`:
  - veja os logs (`Conversion Failed` / `Error:`); confira `PDF_BUCKET` sem `gs://` e `--memory=2Gi`.
- Build falhando no `apt-get`:
  - confira se o `FROM` do Dockerfile manteve a imagem base original (`BASE_IMAGE`).
- `SERVICE_URL` vazio apos reconectar:
  - rode novamente o `export SERVICE_URL=...` da Tarefa 3.

---

## Limpeza (Opcional)

Nao remova a SA `pubsub-cloud-run-invoker`: ela e pre-criada pelo lab.

```bash
gcloud pubsub subscriptions delete "$SUBSCRIPTION_NAME" --quiet
gcloud storage buckets notifications delete "$UPLOAD_BUCKET"
gcloud pubsub topics delete "$TOPIC_NAME" --quiet
gcloud run services delete "$SERVICE_NAME" --region "$REGION" --quiet
gcloud storage rm -r "$UPLOAD_BUCKET" "$PROCESSED_BUCKET"
```

---

## Conceitos-Chave

- Cloud Run: containers serverless, escala ate zero, binarios customizados (LibreOffice) na imagem.
- Cloud Build: build remoto da imagem a partir do `Dockerfile`.
- Servico privado: `--no-allow-unauthenticated` + `roles/run.invoker` para quem chama.
- Notificacao de bucket -> Pub/Sub: eventos `OBJECT_FINALIZE` publicados como JSON.
- Assinatura push autenticada: Pub/Sub chama o Cloud Run com token OIDC da SA invoker.
- `/tmp` no Cloud Run: sistema de arquivos em memoria, usado para download e conversao.

---

## Fluxo Final

1. Definir variaveis (`REGION` do lab) e habilitar as APIs.
2. Baixar o `pet-theory/lab03`, adicionar o script `start` e instalar dependencias.
3. Build `gcr.io/PROJECT_ID/pdf-converter` e deploy privado; testar anonimo x autenticado.
4. Criar os buckets `-upload` e `-processed` e a notificacao para `new-doc`.
5. Dar `run.invoker` a `pubsub-cloud-run-invoker` e criar a assinatura push `pdf-conv-sub`.
6. Enviar arquivos de teste e conferir os logs `file:`.
7. Adicionar LibreOffice ao Dockerfile, substituir o `index.js`, novo build e deploy com `2Gi` e `PDF_BUCKET`.
8. Rodar `copy_files.sh` e conferir os PDFs no `-processed`.
