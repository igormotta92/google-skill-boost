# Build a Resilient, Asynchronous System with Cloud Run and Pub/Sub

## Visao Geral

Ha doze anos, Lily fundou a rede de clinicas veterinarias Pet Theory. Com o crescimento da rede, cresceu tambem a necessidade de automacao. Hoje, quando um resultado de exame volta do laboratorio, o Patrick (TI) escreve um email para o cliente e depois manda o resultado por SMS pelo celular. O processo e lento e sujeito a erros.

Patrick e Ruby (consultora) decidem construir uma solucao **serverless**, escalavel e com pouca manutencao, em que cada atividade e um servico isolado que se comunica via **Pub/Sub**.

Objetivos:

- Criar um topico e assinaturas do Pub/Sub.
- Criar servicos Cloud Run que publicam e recebem mensagens.
- Criar assinaturas Pub/Sub que disparam servicos Cloud Run.
- Testar a resiliencia do sistema.

## Introducao

Este guia converte o lab (GSP650) para execucao via CLI com `gcloud`, `npm` e `curl`, mantendo os nomes de recursos do enunciado. Os arquivos que o lab pede para criar/editar no editor sao gerados com `cat` e `npm pkg`.

Arquitetura alvo:
- `lab-report-service` (publico): recebe o `POST` do laboratorio e publica no topico `new-lab-report`;
- `email-service` e `sms-service` (privados): recebem as mensagens via assinaturas push `email-service-sub` e `sms-service-sub`;
- SA `pubsub-cloud-run-invoker` (pre-criada) autentica os pushes.

Observacao do lab: entre no Console como **Username 1**.

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Definir regiao, zona e variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"

gcloud config set compute/zone "<ZONE_DO_LAB>"     # ajuste conforme o lab
export ZONE=$(gcloud config get compute/zone)
gcloud config set compute/region "<REGION_DO_LAB>" # ajuste conforme o lab
export REGION=$(gcloud config get compute/region)
gcloud config set run/region "$REGION"

export PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"

# Imagem base Node dos Dockerfiles (o enunciado mostra node_docker_image)
export NODE_IMAGE="<NODE_IMAGE_DO_LAB>"   # ajuste conforme o lab (ex: node:20)

# Recursos do lab
export TOPIC_NAME="new-lab-report"
export LAB_SERVICE="lab-report-service"
export EMAIL_SERVICE="email-service"
export SMS_SERVICE="sms-service"
export EMAIL_SUB="email-service-sub"
export SMS_SUB="sms-service-sub"
export INVOKER_SA_NAME="pubsub-cloud-run-invoker"
export INVOKER_SA="${INVOKER_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
export LAB_DIR="$HOME/pet-theory/lab05"
```

### Passo 2: Funcoes auxiliares (Dockerfile e deploy.sh iguais nos 3 servicos)

```bash
write_dockerfile() {
cat > Dockerfile <<EOF
FROM ${NODE_IMAGE}
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
EOF
}

# $1 = nome do servico | $2 = --allow-unauthenticated ou --no-allow-unauthenticated
write_deploy_sh() {
cat > deploy.sh <<EOF
gcloud builds submit \\
  --tag gcr.io/${PROJECT_ID}/$1

gcloud run deploy $1 \\
  --image gcr.io/${PROJECT_ID}/$1 \\
  --region "${REGION}" \\
  $2 \\
  --max-instances=1
EOF
chmod u+x deploy.sh
}
```

### Explicacao dos parametros
- `NODE_IMAGE`: imagem base do `FROM` (o enunciado traz o placeholder `node_docker_image`).
- `write_dockerfile` / `write_deploy_sh`: geram os mesmos arquivos do lab, com os valores das variaveis.
- `\\` no heredoc sem aspas: grava `\` literal (continuacao de linha) no `deploy.sh`.

---

## TAREFA 1: Arquitetura e topico Pub/Sub

### Conceito
O laboratorio externo envia `POST` com os resultados. O `lab-report-service` publica cada resultado no topico `new-lab-report`; qualquer servico inscrito no topico (email, SMS, futuros canais) consome a mensagem de forma independente.

### Passo 1: Criar o topico e habilitar as APIs

```bash
gcloud pubsub topics create "$TOPIC_NAME"

gcloud services enable run.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com

gcloud pubsub topics list --filter="name~${TOPIC_NAME}"
```

### Resultado esperado
- Topico `new-lab-report` criado e Cloud Run API habilitada.

> Click **Check my progress**: *Create a Pub/Sub topic*.

---

## TAREFA 2: Construir o Lab Report Service

### Conceito
Servico publico que faz so duas coisas: recebe o `POST` com o resultado e publica uma mensagem no Pub/Sub, respondendo `204`.

### Passo 1: Baixar o codigo inicial e instalar dependencias

```bash
cd "$HOME"
gcloud storage cp -r "gs://${PROJECT_ID}-assets/pet-theory" .
cd "$LAB_DIR/lab-service"

npm install express body-parser @google-cloud/pubsub
npm pkg set scripts.start="node index.js"
npm pkg get scripts
```

### Passo 2: Criar o `index.js`

```bash
cat > index.js <<'EOF'
const {PubSub} = require('@google-cloud/pubsub');
const pubsub = new PubSub();
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;

app.listen(port, () => {
  console.log('Listening on port', port);
});

app.post('/', async (req, res) => {
  try {
    const labReport = req.body;
    await publishPubSubMessage(labReport);
    res.status(204).send();
  }
  catch (ex) {
    console.log(ex);
    res.status(500).send(ex);
  }
})

async function publishPubSubMessage(labReport) {
  const buffer = Buffer.from(JSON.stringify(labReport));
  await pubsub.topic('new-lab-report').publish(buffer);
}
EOF
```

### Passo 3: Criar `Dockerfile` e `deploy.sh` e fazer o deploy (publico)

```bash
write_dockerfile
write_deploy_sh "$LAB_SERVICE" "--allow-unauthenticated"
cat deploy.sh

./deploy.sh
```

Se falhar na primeira vez (timing), rode `./deploy.sh` novamente.

> Click **Check my progress**: *Deploy the Lab Report Service: Build*.

> Click **Check my progress**: *Deploy the Lab Report Service: Create Revision*.

### Passo 4: Testar com tres relatorios

```bash
export LAB_REPORT_SERVICE_URL=$(gcloud run services describe "$LAB_SERVICE" --region "$REGION" --format="value(status.address.url)")
echo "$LAB_REPORT_SERVICE_URL"

cat > post-reports.sh <<'EOF'
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 12}" \
  $LAB_REPORT_SERVICE_URL &
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 34}" \
  $LAB_REPORT_SERVICE_URL &
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 56}" \
  $LAB_REPORT_SERVICE_URL &
EOF
chmod u+x post-reports.sh

./post-reports.sh
```

Conferir os logs (Console: **Cloud Run > lab-report-service > Logs**):

```bash
gcloud logging read \
  "resource.type=\"cloud_run_revision\" AND resource.labels.service_name=\"${LAB_SERVICE}\" AND httpRequest.status>0" \
  --limit=5 --freshness=10m --format="table(timestamp,httpRequest.status)"
```

### Explicacao dos parametros
- `--allow-unauthenticated`: o laboratorio externo chama sem token.
- `status.address.url`: URL do servico usada pelo `post-reports.sh`.
- `export LAB_REPORT_SERVICE_URL`: o script roda em outro processo e precisa da variavel exportada.
- `<<'EOF'` no `post-reports.sh`: grava `$LAB_REPORT_SERVICE_URL` literal, resolvido quando o script roda.

### Resultado esperado
- 3 requisicoes com status `204`.

### Resumo visual

```text
[Laboratorio externo / post-reports.sh]
        |
        | POST {"id": 12|34|56}  (publico)
        v
[Cloud Run lab-report-service] --publish--> [Topic new-lab-report]
        |
        +-- responde 204
```

---

## TAREFA 3: Email Service

### Conceito
Servico **privado** acionado pelo Pub/Sub. Decodifica a mensagem, chama `sendEmail()` e responde `204` (sucesso) ou `500` (falha, o Pub/Sub tenta de novo).

### Passo 1: Codigo do servico

```bash
cd "$LAB_DIR/email-service"
npm install express body-parser
npm pkg set scripts.start="node index.js"

cat > index.js <<'EOF'
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());

const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});

app.post('/', async (req, res) => {
  const labReport = decodeBase64Json(req.body.message.data);
  try {
    console.log(`Email Service: Report ${labReport.id} trying...`);
    sendEmail();
    console.log(`Email Service: Report ${labReport.id} success :-)`);
    res.status(204).send();
  }
  catch (ex) {
    console.log(`Email Service: Report ${labReport.id} failure: ${ex}`);
    res.status(500).send();
  }
})

function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}

function sendEmail() {
  console.log('Sending email');
}
EOF
```

### Passo 2: Deploy (privado)

```bash
write_dockerfile
write_deploy_sh "$EMAIL_SERVICE" "--no-allow-unauthenticated"
./deploy.sh
```

> Click **Check my progress**: *Deploy the Email Service: Build*.

> Click **Check my progress**: *Deploy the Email Service: Create Revision*.

### Passo 3: Permitir que a SA invoque o Email Service

```bash
# Pre-criada pelo lab; so cria se nao existir
gcloud iam service-accounts describe "$INVOKER_SA" >/dev/null 2>&1 || \
  gcloud iam service-accounts create "$INVOKER_SA_NAME" --display-name "PubSub Cloud Run Invoker"

gcloud run services add-iam-policy-binding "$EMAIL_SERVICE" \
  --member="serviceAccount:${INVOKER_SA}" \
  --role=roles/run.invoker \
  --region "$REGION"
```

> Click **Check my progress**: *Configure service account permissions*.

### Passo 4: Conferir o service agent do Pub/Sub (pre-configurado)

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --format="table(bindings.role)" \
  --filter="bindings.members:service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com"
```

### Passo 5: Criar a assinatura push do Email Service

```bash
export EMAIL_SERVICE_URL=$(gcloud run services describe "$EMAIL_SERVICE" --region "$REGION" --format="value(status.address.url)")
echo "$EMAIL_SERVICE_URL"

gcloud pubsub subscriptions create "$EMAIL_SUB" \
  --topic "$TOPIC_NAME" \
  --push-endpoint="$EMAIL_SERVICE_URL" \
  --push-auth-service-account="$INVOKER_SA"
```

> Click **Check my progress**: *Create the Pub/Sub subscription*.

### Passo 6: Testar Lab Report + Email

```bash
"$LAB_DIR/lab-service/post-reports.sh"
sleep 20

gcloud logging read \
  "resource.type=\"cloud_run_revision\" AND resource.labels.service_name=\"${EMAIL_SERVICE}\" AND textPayload:\"Email Service\"" \
  --limit=10 --freshness=10m --format="value(textPayload)"
```

### Explicacao dos parametros
- `--no-allow-unauthenticated`: so quem tem `run.invoker` chama o servico.
- `describe || create`: evita erro se a SA ja existir (o lab a pre-cria).
- `--push-auth-service-account`: o Pub/Sub envia token OIDC dessa SA no push.
- Service agent do Pub/Sub com `serviceAccountTokenCreator`: necessario para gerar esses tokens.

### Resultado esperado
- Logs `Email Service: Report 12/34/56 trying...` e `success :-)`.

### Resumo visual

```text
[Topic new-lab-report]
        |
        | push email-service-sub (OIDC pubsub-cloud-run-invoker)
        v
[Cloud Run email-service] (privado, run.invoker)
        |
        +-- 204 = processado | 500 = Pub/Sub reenvia
```

---

## TAREFA 4: SMS Service

### Conceito
Mesmo padrao do Email Service, com uma segunda assinatura no mesmo topico (fan-out): cada mensagem e entregue para os dois servicos.

### Passo 1: Codigo do servico

```bash
cd "$LAB_DIR/sms-service"
npm install express body-parser
npm pkg set scripts.start="node index.js"

cat > index.js <<'EOF'
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());

const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});

app.post('/', async (req, res) => {
  const labReport = decodeBase64Json(req.body.message.data);
  try {
    console.log(`SMS Service: Report ${labReport.id} trying...`);
    sendSms();

    console.log(`SMS Service: Report ${labReport.id} success :-)`);
    res.status(204).send();
  }
  catch (ex) {
    console.log(`SMS Service: Report ${labReport.id} failure: ${ex}`);
    res.status(500).send();
  }
})

function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}

function sendSms() {
  console.log('Sending SMS');
}
EOF
```

### Passo 2: Deploy (privado)

```bash
write_dockerfile
write_deploy_sh "$SMS_SERVICE" "--no-allow-unauthenticated"
./deploy.sh
```

> Click **Check my progress**: *Deploy the SMS Service*.

### Passo 3: Permissao e assinatura push

```bash
gcloud run services add-iam-policy-binding "$SMS_SERVICE" \
  --member="serviceAccount:${INVOKER_SA}" \
  --role=roles/run.invoker \
  --region "$REGION"

export SMS_SERVICE_URL=$(gcloud run services describe "$SMS_SERVICE" --region "$REGION" --format="value(status.address.url)")
echo "$SMS_SERVICE_URL"

gcloud pubsub subscriptions create "$SMS_SUB" \
  --topic "$TOPIC_NAME" \
  --push-endpoint="$SMS_SERVICE_URL" \
  --push-auth-service-account="$INVOKER_SA"
```

### Passo 4: Testar os tres servicos

```bash
"$LAB_DIR/lab-service/post-reports.sh"
sleep 20

gcloud logging read \
  "resource.type=\"cloud_run_revision\" AND resource.labels.service_name=\"${SMS_SERVICE}\" AND textPayload:\"SMS Service\"" \
  --limit=10 --freshness=10m --format="value(textPayload)"
```

### Resultado esperado
- Logs `SMS Service: Report ... success :-)` alem dos logs do Email Service.

### Resumo visual

```text
                        [Topic new-lab-report]
                                 |
                 +---------------+---------------+
                 |                               |
       push email-service-sub          push sms-service-sub
                 |                               |
                 v                               v
        [email-service]                   [sms-service]
```

---

## TAREFA 5: Testar a resiliencia do sistema

### Conceito
Com comunicacao assincrona, uma falha no Email Service nao afeta o SMS Service nem o Lab Report Service. O Pub/Sub reenvia as mensagens enquanto o servico responde `500` e para quando ele volta a responder `204`, sem perda de dados.

### Passo 1: Publicar uma versao com erro do Email Service

Console (passo do lab): editar `sendEmail()` e adicionar `throw 'Email server is down';` como primeira linha.

Equivalente CLI:

```bash
cd "$LAB_DIR/email-service"
sed -i "s/^function sendEmail() {/function sendEmail() {\n  throw 'Email server is down';/" index.js
grep -n -A3 "function sendEmail" index.js

./deploy.sh
```

### Passo 2: Enviar relatorios e observar as falhas

```bash
"$LAB_DIR/lab-service/post-reports.sh"
sleep 30

# Email falhando e sendo reenviado (500)
gcloud logging read \
  "resource.type=\"cloud_run_revision\" AND resource.labels.service_name=\"${EMAIL_SERVICE}\" AND textPayload:\"failure\"" \
  --limit=10 --freshness=10m --format="value(timestamp,textPayload)"

# SMS seguindo normalmente
gcloud logging read \
  "resource.type=\"cloud_run_revision\" AND resource.labels.service_name=\"${SMS_SERVICE}\" AND textPayload:\"success\"" \
  --limit=5 --freshness=10m --format="value(timestamp,textPayload)"
```

### Passo 3: Corrigir e publicar a versao boa

```bash
sed -i "/throw 'Email server is down';/d" index.js
grep -n -A3 "function sendEmail" index.js

./deploy.sh
sleep 30

gcloud logging read \
  "resource.type=\"cloud_run_revision\" AND resource.labels.service_name=\"${EMAIL_SERVICE}\" AND textPayload:\"success\"" \
  --limit=10 --freshness=10m --format="value(timestamp,textPayload)"
```

### Explicacao dos parametros
- `sed "s/^function sendEmail() {/...\n  throw .../"`: insere o `throw` logo apos a abertura da funcao (GNU sed, Cloud Shell).
- `sed "/throw .../d"`: remove a linha do erro.
- `500` -> retry do Pub/Sub; `204` -> mensagem confirmada (ack).

### Resultado esperado
- Com a versao ruim: `Email Service: Report ... failure: Email server is down` repetindo; SMS com `success`.
- Apos a correcao: relatorios `12`, `34` e `56` com `success` no Email Service, sem perda de mensagens.

### Resumo visual

```text
Versao ruim:
[Topic new-lab-report] --push--> [email-service] throw -> 500
        |                              ^
        |                              | Pub/Sub reenvia (retry)
        +--push--> [sms-service] -> 204 (nao afetado)

Versao corrigida (deploy):
[Topic new-lab-report] --retry--> [email-service] -> 204 (ack)
                                   12, 34, 56 entregues, sem perda
```

---

## Validacao

```bash
gcloud run services list --region "$REGION" \
  --format="table(metadata.name,status.url)"

gcloud run services get-iam-policy "$LAB_SERVICE" --region "$REGION" --format="value(bindings.members)"
gcloud run services get-iam-policy "$EMAIL_SERVICE" --region "$REGION" --format="value(bindings.members)"
gcloud run services get-iam-policy "$SMS_SERVICE" --region "$REGION" --format="value(bindings.members)"

gcloud pubsub subscriptions list \
  --format="table(name.basename(),topic.basename(),pushConfig.pushEndpoint)"
```

Checklist esperado:
- topico `new-lab-report`;
- `lab-report-service` publico (`allUsers`), `email-service` e `sms-service` privados com `pubsub-cloud-run-invoker` como invoker;
- assinaturas `email-service-sub` e `sms-service-sub` apontando para as URLs dos servicos;
- Email Service corrigido, sem `throw`.

---

## Resumo Visual da Arquitetura

Compilado: estado final ao termino do lab. Os detalhes de cada etapa e do teste de falha estao no `### Resumo visual` das Tarefas 2 a 5.

```text
[Laboratorio externo] --POST--> [lab-report-service] (publico)
                                         |
                                         | publish
                                         v
                                [Topic new-lab-report]
                                         |
                   +---------------------+---------------------+
                   | email-service-sub                         | sms-service-sub
                   | (push OIDC)                               | (push OIDC)
                   v                                           v
          [email-service] (privado)                  [sms-service] (privado)

Push autenticado pela SA pubsub-cloud-run-invoker (run.invoker).
```

---

## Troubleshooting

- `./deploy.sh` falha na primeira execucao:
  - problema de timing citado pelo lab; rode novamente.
- Build falhando no `FROM`:
  - confira `NODE_IMAGE` com a imagem indicada no lab e regenere com `write_dockerfile`.
- Deploy falha por `npm start` inexistente:
  - confira `npm pkg get scripts` (deve ter `start: node index.js`).
- `post-reports.sh` sem efeito / URL vazia:
  - rode `export LAB_REPORT_SERVICE_URL=...` antes; o script depende da variavel exportada.
- Email/SMS sem logs:
  - confira o binding `run.invoker` da SA no servico e o `pushEndpoint` da assinatura; aguarde alguns segundos.
- `service-accounts create` diz que ja existe:
  - esperado; a SA e pre-criada pelo lab.
- Funcoes `write_dockerfile` / `write_deploy_sh` nao encontradas:
  - foram perdidas ao reconectar; rode de novo o Passo 2 dos pre-requisitos.

---

## Limpeza (Opcional)

Nao remova a SA `pubsub-cloud-run-invoker`: ela e pre-criada pelo lab.

```bash
gcloud pubsub subscriptions delete "$EMAIL_SUB" "$SMS_SUB" --quiet
gcloud pubsub topics delete "$TOPIC_NAME" --quiet
gcloud run services delete "$LAB_SERVICE" "$EMAIL_SERVICE" "$SMS_SERVICE" --region "$REGION" --quiet
```

---

## Conceitos-Chave

- Comunicacao assincrona: servicos desacoplados via Pub/Sub, sem chamadas diretas.
- Fan-out: um topico, varias assinaturas; novos canais entram sem alterar o publicador.
- Push autenticado: Pub/Sub chama o Cloud Run com token OIDC de uma SA com `run.invoker`.
- Retry automatico: `500` faz o Pub/Sub reenviar; `204` confirma a mensagem.
- Auto-recuperacao: ao corrigir o servico, as mensagens pendentes sao entregues sem perda.

---

## Fluxo Final

1. Definir regiao/zona, variaveis e funcoes auxiliares.
2. Criar o topico `new-lab-report` e habilitar o Cloud Run.
3. Criar e publicar o `lab-report-service` (publico) e testar com `post-reports.sh`.
4. Criar e publicar o `email-service` (privado), dar `run.invoker` a SA e criar `email-service-sub`.
5. Criar e publicar o `sms-service` (privado), dar `run.invoker` e criar `sms-service-sub`.
6. Publicar uma versao com erro do Email Service, observar os retries e o SMS funcionando.
7. Corrigir o Email Service e confirmar a entrega dos relatorios pendentes.
