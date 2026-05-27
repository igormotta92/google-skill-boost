## Desafio Final

Este runbook cria uma Cloud Function Gen2 que:
1. Escuta upload de arquivos no bucket.
2. Gera thumbnail 64x64 para imagens (`png`, `jpg`, `jpeg`).
3. Publica o nome do thumbnail em um topico Pub/Sub.

Fluxo sugerido de execucao:
1. Definir variaveis e preparar recursos base.
2. Criar o codigo da funcao e dependencias.
3. Aplicar IAM minimo.
4. Fazer deploy, validar e testar ponta a ponta.
5. Executar validacoes finais e limpeza opcional.

## 0) Variaveis e contexto

```bash
export PROJECT_ID="qwiklabs-gcp-01-00b8f77fd3a5"
gcloud config set project "$PROJECT_ID"
export PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"

export REGION="europe-west3"
export ZONE="europe-west3-b"
export TOPIC_NAME="topic-memories-492"
export FUNCTION_NAME="memories-thumbnail-generator"

export ENTRY_POINT=${FUNCTION_NAME}
export SA_FUNCTION="cloudfunctionsa@${PROJECT_ID}.iam.gserviceaccount.com"
export BUCKET_INPUT="${PROJECT_ID}-bucket"
export BUCKET_STAGE="${PROJECT_ID}-stage"
```

## 1) Ativar APIs necessarias

```bash
gcloud services enable \
  cloudfunctions.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  run.googleapis.com \
  eventarc.googleapis.com \
  pubsub.googleapis.com \
  storage.googleapis.com

  # Garante APIs
# gcloud services enable storage.googleapis.com pubsub.googleapis.com eventarc.googleapis.com
```

## 2) Criar buckets e topico

```bash
gcloud storage buckets create "gs://${BUCKET_STAGE}" --location="${REGION}"
gcloud storage buckets create "gs://${BUCKET_INPUT}" --location="${REGION}"
gcloud pubsub topics create "${TOPIC_NAME}"
```

## 3) Criar projeto da funcao

Crie o diretorio da funcao e entre nele para manter os arquivos isolados.

```bash
mkdir -p ${FUNCTION_NAME} && cd $_
```

### index.js

Arquivo principal da Cloud Function (handler de evento + geracao de thumbnail + publicacao no Pub/Sub).

```bash
cat > index.js << 'EOF'
const functions = require('@google-cloud/functions-framework');
const { Storage } = require('@google-cloud/storage');
const { PubSub } = require('@google-cloud/pubsub');
const sharp = require('sharp');

functions.cloudEvent('__ENTRY_POINT__', async cloudEvent => {
  const event = cloudEvent.data;

  console.log(`Event: ${JSON.stringify(event)}`);
  console.log(`Hello ${event.bucket}`);

  const fileName = event.name;
  const bucketName = event.bucket;
  const size = "64x64";
  const bucket = new Storage().bucket(bucketName);
  const topicName = "__TOPIC_NAME__";
  const pubsub = new PubSub();

  if (fileName.search("64x64_thumbnail") === -1) {
    // doesn't have a thumbnail, get the filename extension
    const filename_split = fileName.split('.');
    const filename_ext = filename_split[filename_split.length - 1].toLowerCase();
    const filename_without_ext = fileName.substring(0, fileName.length - filename_ext.length - 1); // fix sub string to remove the dot

    if (filename_ext === 'png' || filename_ext === 'jpg' || filename_ext === 'jpeg') {
      // only support png and jpg at this point
      console.log(`Processing Original: gs://${bucketName}/${fileName}`);
      const gcsObject = bucket.file(fileName);
      const newFilename = `${filename_without_ext}_64x64_thumbnail.${filename_ext}`;
      const gcsNewObject = bucket.file(newFilename);

      try {
        const [buffer] = await gcsObject.download();
        const resizedBuffer = await sharp(buffer)
          .resize(64, 64, {
            fit: 'inside',
            withoutEnlargement: true,
          })
          .toFormat(filename_ext)
          .toBuffer();

        await gcsNewObject.save(resizedBuffer, {
          metadata: {
            contentType: `image/${filename_ext}`,
          },
        });

        console.log(`Success: ${fileName} → ${newFilename}`);

        await pubsub
          .topic(topicName)
          .publishMessage({ data: Buffer.from(newFilename) });

        console.log(`Message published to ${topicName}`);
      } catch (err) {
        console.error(`Error: ${err}`);
      }
    } else {
      console.log(`gs://${bucketName}/${fileName} is not an image I can handle`);
    }
  } else {
    console.log(`gs://${bucketName}/${fileName} already has a thumbnail`);
  }
});
EOF

# Cloud Shell (Linux)
sed -i "s|__TOPIC_NAME__|$TOPIC_NAME|g" index.js
sed -i "s|__ENTRY_POINT__|$ENTRY_POINT|g" index.js

# macOS (BSD sed)
# sed -i '' "s|__TOPIC_NAME__|$TOPIC_NAME|g" index.js
# sed -i '' "s|__ENTRY_POINT__|$ENTRY_POINT|g" index.js
```

### package.json

Manifesto de dependencias e runtime da aplicacao Node.js.

```bash
cat > package.json << EOF
{
 "name": "thumbnails",
 "version": "1.0.0",
 "description": "Create Thumbnail of uploaded image",
 "scripts": {
   "start": "node index.js"
 },
 "dependencies": {
   "@google-cloud/functions-framework": "^3.0.0",
   "@google-cloud/pubsub": "^2.0.0",
   "@google-cloud/storage": "^6.11.0",
   "sharp": "^0.32.1"
 },
 "devDependencies": {},
 "engines": {
   "node": ">=4.3.2"
 }
}
EOF
```

```bash
npm install
```

## 4) Criar service account da funcao

Conta de servico usada no runtime da Cloud Function.

```bash
gcloud iam service-accounts create cloudfunctionsa \
  --display-name="Cloud Functions Runtime Service Account"
```

## 5) Permissoes IAM (minimo recomendado)

Aplicar na ordem abaixo para reduzir falhas durante criacao de trigger e execucao.

### 5.1 Runtime SA escreve no bucket de entrada

```bash
gcloud storage buckets add-iam-policy-binding "gs://${BUCKET_INPUT}" \
  --member="serviceAccount:${SA_FUNCTION}" \
  --role="roles/storage.objectAdmin"
```

### 5.2 Runtime SA publica no topico

```bash
gcloud pubsub topics add-iam-policy-binding "${TOPIC_NAME}" \
  --project="${PROJECT_ID}" \
  --member="serviceAccount:${SA_FUNCTION}" \
  --role="roles/pubsub.publisher"
```

### 5.3 Runtime SA recebe eventos do Eventarc

```bash
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:${SA_FUNCTION}" \
  --role="roles/eventarc.eventReceiver"
```

### 5.4 Cloud Storage Service Agent publica eventos

Necessario para eventos de Cloud Storage chegarem via Pub/Sub/Eventarc.

```bash
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gs-project-accounts.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"
```

Caso de problema no bind da conta de serviço, será necessário forçar o GCP a criar novamente o SA de serviço so Storage
```bash
gcloud storage service-agent --project=${PROJECT_NUMBER}
```

### 5.5 (Opcional) Eventarc receiver para a Compute default SA

Em alguns ambientes (org policies mais restritivas), o trigger so cria se esta permissao existir:

```bash
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/eventarc.eventReceiver"
```

### 5.6 Trigger SA pode invocar o servico (rode apos o primeiro deploy)

Use esta etapa apenas quando necessaria no seu ambiente/politica.

```bash
gcloud run services add-iam-policy-binding "${FUNCTION_NAME}" \
  --region="${REGION}" \
  --member="serviceAccount:${SA_FUNCTION}" \
  --role="roles/run.invoker"
```

## 6) Deploy da Cloud Function Gen2

Deploy da funcao Gen2 com trigger por bucket e conta de servico dedicada.

```bash
gcloud functions deploy "${FUNCTION_NAME}" \
  --gen2 \
  --runtime=nodejs22 \
  --region="${REGION}" \
  --source=. \
  --entry-point="${ENTRY_POINT}" \
  --trigger-bucket="${BUCKET_INPUT}" \
  --trigger-service-account="${SA_FUNCTION}" \
  --stage-bucket="${BUCKET_STAGE}" \
  --service-account="${SA_FUNCTION}" \
  --set-env-vars="TOPIC_NAME=${TOPIC_NAME},ENTRY_POINT=${ENTRY_POINT}"
```

Observacao: `--allow-unauthenticated` nao se aplica para trigger por bucket (evento), entao foi removido.

## 7) Teste fim a fim

Upload de arquivo de teste e validacao de logs/artefatos gerados.

```bash
curl -L -o map.jpg https://storage.googleapis.com/cloud-training/gsp315/map.jpg
gcloud storage cp map.jpg "gs://${BUCKET_INPUT}"
```

Verificar log

```bash
# old
gcloud run services logs read ${FUNCTION_NAME} \
  --region=${REGION} \
  --project=${PROJECT_ID} \
  --limit=100

# gen2
gcloud functions logs read ${FUNCTION_NAME} \
  --gen2 \
  --region=${REGION} \
  --project=${PROJECT_ID} \
  --limit=50
```

Validar se o thumbnail foi criado:

```bash
gcloud storage ls "gs://${BUCKET_INPUT}"
```

Opcional: criar subscription temporaria para validar mensagem Pub/Sub:

```bash
gcloud pubsub subscriptions create topic-memories-sub --topic="${TOPIC_NAME}"
gcloud pubsub subscriptions pull topic-memories-sub --limit=1 --auto-ack
```

## 8) Removendo permissao (opcional)

Use apenas se precisar remover acesso previamente concedido.

```bash
gcloud projects remove-iam-policy-binding "${PROJECT_ID}" \
  --member="user:student-00-ef0210023997@qwiklabs.net" \
  --role="roles/viewer"
```

## 9) Comandos uteis de validacao

Checklist final para confirmar estado da funcao, IAM, trigger e topico.

```bash
gcloud functions describe "${FUNCTION_NAME}" --gen2 --region="${REGION}"
gcloud projects get-iam-policy "${PROJECT_ID}" --flatten="bindings[].members" --filter="bindings.members:${SA_FUNCTION}" --format="table(bindings.role)"
gcloud eventarc triggers list --location="${REGION}"
gcloud pubsub topics get-iam-policy "${TOPIC_NAME}"
```