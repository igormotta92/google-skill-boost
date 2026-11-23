## Desafio Final

Este runbook cria uma Cloud Function Gen2 que:
1. Escuta upload de arquivos no bucket.
2. Gera thumbnail 64x64 para imagens (`png`, `jpg`, `jpeg`).
3. Publica o nome do thumbnail em um topico Pub/Sub.

Fluxo sugerido de execucao:
1. Definir variaveis e ativar APIs.
2. Criar bucket e topico (Tasks 1 e 2).
3. Criar o codigo da funcao, fazer deploy e testar (Task 3).
4. Remover o engenheiro anterior (Task 4).
5. Executar validacoes finais.

> **IAM restrito no lab:** o usuario `student` nao pode criar service accounts nem conceder papeis (apenas mexer em `roles/viewer`). As permissoes que a funcao precisa ja vem pre-concedidas pelo lab, por isso o deploy usa a Compute default SA, sem SA customizada.

## 0) Variaveis e contexto

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
gcloud config set project "$PROJECT_ID"
export PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"

export REGION="us-central1"
export TOPIC_NAME="topic-memories-303"
export FUNCTION_NAME="memories-thumbnail-creator"

export ENTRY_POINT=${FUNCTION_NAME}
export BUCKET_INPUT="${PROJECT_ID}-bucket"
export OLD_ENGINEER="student-01-b7ea4fe32e69@qwiklabs.net"
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
```

## 2) Criar bucket e topico (Tasks 1 e 2)

Bucket regional em `us-central1` e topico Pub/Sub usado pela funcao.

```bash
gcloud storage buckets create "gs://${BUCKET_INPUT}" --location="${REGION}"
gcloud pubsub topics create "${TOPIC_NAME}"
```

## 3) Criar projeto da funcao (Task 3)

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

## 4) Deploy da Cloud Function Gen2

Deploy com trigger por bucket usando a Compute default SA (`${PROJECT_NUMBER}-compute@developer.gserviceaccount.com`) para runtime e trigger. Rode de dentro do diretorio `${FUNCTION_NAME}`.

```bash
gcloud functions deploy "${FUNCTION_NAME}" \
  --gen2 \
  --runtime=nodejs22 \
  --region="${REGION}" \
  --source=. \
  --entry-point="${ENTRY_POINT}" \
  --trigger-bucket="${BUCKET_INPUT}"
```

Se falhar com erro de permissao do Eventarc / service agent ("it may take a few minutes before all necessary permissions are propagated"), aguarde 2-5 minutos e rode o mesmo comando novamente. Persistindo, crie a funcao pelo Console (Cloud Run > Write a function), que oferece botao para conceder os papeis faltantes.

Observacao: `--allow-unauthenticated` nao se aplica para trigger por bucket (evento).

## 5) Teste fim a fim

Upload de arquivo de teste. Para o "Check my progress" basta o upload; nao precisa esperar o thumbnail.

```bash
curl -L -o map.jpg https://storage.googleapis.com/cloud-training/gsp315/map.jpg
gcloud storage cp map.jpg "gs://${BUCKET_INPUT}"
```

Verificar log:

```bash
gcloud functions logs read ${FUNCTION_NAME} \
  --gen2 \
  --region=${REGION} \
  --project=${PROJECT_ID} \
  --limit=50

# Se a funcao foi criada pelo Console (Cloud Run > Write a function),
# ela existe apenas como servico Cloud Run e nao aparece em `gcloud functions`:
gcloud run services logs read ${FUNCTION_NAME} \
  --region=${REGION} \
  --project=${PROJECT_ID} \
  --limit=50
```

Validar se o thumbnail foi criado:

```bash
gcloud storage ls "gs://${BUCKET_INPUT}"
```

## 6) Remover o engenheiro anterior (Task 4)

Remove o papel `Viewer` do cloud engineer anterior. Como a policy do projeto possui bindings com condicao, o `gcloud` pede uma condicao interativamente; `--condition=None` evita o prompt.

```bash
gcloud projects remove-iam-policy-binding "${PROJECT_ID}" \
  --member="user:${OLD_ENGINEER}" \
  --role="roles/viewer" \
  --condition=None
```

Validar que o usuario nao aparece mais na policy:

```bash
gcloud projects get-iam-policy "${PROJECT_ID}" \
  --flatten="bindings[].members" \
  --filter="bindings.members:${OLD_ENGINEER}" \
  --format="table(bindings.role)"
```

## 7) Comandos uteis de validacao

Checklist final para confirmar estado da funcao e do trigger.

```bash
gcloud functions describe "${FUNCTION_NAME}" --gen2 --region="${REGION}" \
  --format="value(state, serviceConfig.serviceAccountEmail, eventTrigger.eventType)"
gcloud functions list --regions="${REGION}"
gcloud run services list --region="${REGION}"   # funcoes criadas pelo Console aparecem aqui
gcloud eventarc triggers list --location="${REGION}"
```
