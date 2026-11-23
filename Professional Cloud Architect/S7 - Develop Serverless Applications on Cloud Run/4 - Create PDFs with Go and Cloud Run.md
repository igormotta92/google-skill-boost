# Create PDFs with Go and Cloud Run

## Visao Geral

Neste lab, voce constroi um conversor de PDF no **Cloud Run** (serverless) que converte automaticamente arquivos enviados ao Cloud Storage em PDFs gravados em outro bucket.

Cenario: voce ajuda a clinica veterinaria Pet Theory a converter as faturas em PDF para facilitar a vida dos clientes.

APIs ja habilitadas pelo lab: Cloud Build, Cloud Storage, Cloud Run Admin, Artifact Registry e Pub/Sub.

Objetivos:

- Obter o codigo-fonte do lab.
- Converter uma aplicacao Go em container.
- Gerar containers com Cloud Build.
- Criar um servico Cloud Run que converte arquivos para PDF.
- Criar uma service account e conceder permissoes.
- Configurar notificacoes do Cloud Storage como gatilho.
- Usar processamento de eventos com Cloud Storage.

## Introducao

Este guia converte o lab (GSP762) para execucao via CLI com `gcloud`, `git`, `go` e `curl`, mantendo os nomes de recursos do enunciado. Os arquivos editados no Cloud Shell Editor sao gerados com `cat`.

Recursos usados no lab:
- codigo em `github.com/Deleplace/pet-theory` (pasta `lab03`: `server.go`, `notification.go`, `gcs.go`);
- repositorio `pdf-converter` no Artifact Registry e servico Cloud Run `pdf-converter` (privado, `2Gi`, ate 3 instancias);
- buckets pre-criados `<PROJECT_ID>-upload` e `<PROJECT_ID>-processed`;
- topico `new-doc`, assinatura push `pdf-conv-sub` e SA `pubsub-cloud-run-invoker` (pre-criada).

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
export REGION="<REGION_DO_LAB>"   # ajuste conforme o lab

export APP_DIR="$HOME/pet-theory/lab03"
export SERVICE_NAME="pdf-converter"
export AR_REPO="pdf-converter"
export IMAGE_URI="${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/${SERVICE_NAME}"
export UPLOAD_BUCKET="gs://${PROJECT_ID}-upload"
export PROCESSED_BUCKET_NAME="${PROJECT_ID}-processed"
export TOPIC_NAME="new-doc"
export SUBSCRIPTION_NAME="pdf-conv-sub"
export INVOKER_SA_NAME="pubsub-cloud-run-invoker"
export INVOKER_SA="${INVOKER_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud config set project "$PROJECT_ID"
gcloud config set run/region "$REGION"
```

### Explicacao dos parametros
- `IMAGE_URI`: imagem no Artifact Registry (`REGION-docker.pkg.dev/PROJECT/REPO/IMAGEM`).
- `PROCESSED_BUCKET_NAME`: nome (sem `gs://`) usado na variavel `PDF_BUCKET` do servico.

---

## TAREFA 1: Obter o codigo-fonte

### Conceito
O repositorio `pet-theory` traz o servico Go com as funcoes auxiliares de Pub/Sub (`notification.go`) e Cloud Storage (`gcs.go`).

### Passo 1: Conferir a conta e clonar o repositorio

```bash
gcloud auth list --filter=status:ACTIVE --format="value(account)"

cd "$HOME"
git clone https://github.com/Deleplace/pet-theory.git
cd "$APP_DIR"
ls
```

### Resultado esperado
- Pasta `lab03` com `server.go`, `notification.go`, `gcs.go`, `go.mod` e `Dockerfile`.

---

## TAREFA 2: Criar o microservico de faturas

### Conceito
O `server.go` recebe o `POST` do Pub/Sub, baixa o arquivo do bucket para um diretorio temporario, converte com `libreoffice --headless --convert-to pdf`, envia o PDF para o bucket `PDF_BUCKET` e apaga o original. Um `GET` responde que o servico esta pronto.

### Passo 1: Substituir o `server.go`

```bash
cd "$APP_DIR"

cat > server.go <<'EOF'
package main

import (
     "fmt"
     "log"
     "net/http"
     "os"
     "os/exec"
     "regexp"
     "strings"
)

func main() {
     http.HandleFunc("/", process)

     port := os.Getenv("PORT")
     if port == "" {
          port = "8080"
          log.Printf("Defaulting to port %s", port)
     }

     log.Printf("Listening on port %s", port)
     err := http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
     log.Fatal(err)
}

func process(w http.ResponseWriter, r *http.Request) {
     log.Println("Serving request")

     if r.Method == "GET" {
          fmt.Fprintln(w, "Ready to process POST requests from Cloud Storage trigger")
          return
     }

     //
     // Read request body containing Cloud Storage object metadata
     //
     gcsInputFile, err1 := readBody(r)
     if err1 != nil {
          log.Printf("Error reading POST data: %v", err1)
          w.WriteHeader(http.StatusBadRequest)
          fmt.Fprintf(w, "Problem with POST data: %v \n", err1)
          return
     }

     //
     // Working directory (concurrency-safe)
     localDir, err := os.MkdirTemp("", "")
     if err != nil {
          log.Printf("Error creating local temp dir: %v", err)
          w.WriteHeader(http.StatusInternalServerError)
          fmt.Fprintf(w, "Could not create a temp directory on server. \n")
          return
     }
     defer os.RemoveAll(localDir)

     //
     // Download input file from Cloud Storage
     //
     localInputFile, err2 := download(gcsInputFile, localDir)
     if err2 != nil {
          log.Printf("Error downloading Cloud Storage file [%s] from bucket [%s]: %v",
               gcsInputFile.Name, gcsInputFile.Bucket, err2)
          w.WriteHeader(http.StatusInternalServerError)
          fmt.Fprintf(w, "Error downloading Cloud Storage file [%s] from bucket [%s]",
               gcsInputFile.Name, gcsInputFile.Bucket)
          return
     }

     //
     // Use LibreOffice to convert local input file to local PDF file.
     //
     localPDFFilePath, err3 := convertToPDF(localInputFile.Name(), localDir)
     if err3 != nil {
          log.Printf("Error converting to PDF: %v", err3)
          w.WriteHeader(http.StatusInternalServerError)
          fmt.Fprintf(w, "Error converting to PDF.")
          return
     }

     //
     // Upload the freshly generated PDF to Cloud Storage
     //
     targetBucket := os.Getenv("PDF_BUCKET")
     err4 := upload(localPDFFilePath, targetBucket)
     if err4 != nil {
          log.Printf("Error uploading PDF file to bucket [%s]: %v", targetBucket, err4)
          w.WriteHeader(http.StatusInternalServerError)
          fmt.Fprintf(w, "Error downloading Cloud Storage file [%s] from bucket [%s]",
               gcsInputFile.Name, gcsInputFile.Bucket)
          return
     }

     //
     // Delete the original input file from Cloud Storage.
     //
     err5 := deleteGCSFile(gcsInputFile.Bucket, gcsInputFile.Name)
     if err5 != nil {
          log.Printf("Error deleting file [%s] from bucket [%s]: %v", gcsInputFile.Name,
               gcsInputFile.Bucket, err5)
          // This is not a blocking error.
          // The PDF was successfully generated and uploaded.
     }

     log.Println("Successfully produced PDF")
     fmt.Fprintln(w, "Successfully produced PDF")
}

func convertToPDF(localFilePath string, localDir string) (resultFilePath string, err error) {
     log.Printf("Converting [%s] to PDF", localFilePath)
     cmd := exec.Command("libreoffice", "--headless", "--convert-to", "pdf",
          "--outdir", localDir,
          localFilePath)
     cmd.Stdout, cmd.Stderr = os.Stdout, os.Stderr
     log.Println(cmd)
     err = cmd.Run()
     if err != nil {
          return "", err
     }

     pdfFilePath := regexp.MustCompile(`\.\w+$`).ReplaceAllString(localFilePath, ".pdf")
     if !strings.HasSuffix(pdfFilePath, ".pdf") {
          pdfFilePath += ".pdf"
     }
     log.Printf("Converted %s to %s", localFilePath, pdfFilePath)
     return pdfFilePath, nil
}
EOF
```

### Passo 2: Compilar

```bash
go build -o server
ls -la server
```

### Explicacao dos parametros
- `<<'EOF'`: grava o Go sem expansao (o codigo tem crases e `%s`).
- `readBody`, `download`, `upload`, `deleteGCSFile`: funcoes ja existentes em `notification.go` e `gcs.go`.
- `os.MkdirTemp`: diretorio temporario por requisicao (seguro com concorrencia).

### Resultado esperado
- Binario `server` gerado sem erros.

---

## TAREFA 3: Criar o servico de conversao de PDF

### Conceito
O container junta o binario `server` e o LibreOffice. A imagem e gerada pelo Cloud Build e publicada como servico **privado** com `2Gi` de memoria e o bucket de destino em `PDF_BUCKET`.

### Passo 1: Atualizar o `Dockerfile`

```bash
cat > Dockerfile <<'EOF'
FROM amd64/debian
RUN apt-get update -y \
  && apt-get install -y libreoffice \
  && apt-get clean
WORKDIR /usr/src/app
COPY server .
CMD [ "./server" ]
EOF
```

### Passo 2: Criar o repositorio e gerar a imagem

```bash
gcloud artifacts repositories create "$AR_REPO" \
  --repository-format=docker \
  --location="$REGION"

gcloud builds submit --tag "$IMAGE_URI"
```

> Click **Check my progress**: *Build an image with Cloud Build*.

### Passo 3: Deploy no Cloud Run

```bash
gcloud run deploy "$SERVICE_NAME" \
  --image "$IMAGE_URI" \
  --region "$REGION" \
  --memory=2Gi \
  --no-allow-unauthenticated \
  --set-env-vars PDF_BUCKET="$PROCESSED_BUCKET_NAME" \
  --max-instances=3
```

### Explicacao dos parametros
- `FROM amd64/debian` + `apt-get install libreoffice`: base Debian com o conversor.
- `COPY server .` / `CMD [ "./server" ]`: copia e executa o binario Go.
- `--memory=2Gi`: memoria recomendada para o LibreOffice.
- `--no-allow-unauthenticated`: so chamadas autenticadas.
- `--set-env-vars PDF_BUCKET=...`: bucket de saida dos PDFs.
- `--max-instances=3`: limite de escala.

### Resultado esperado
- Servico `pdf-converter` implantado e privado.

> Click **Check my progress**: *Deploy the PDF converter service*.

---

## TAREFA 4: Criar a service account e o gatilho de notificacao

### Conceito
O bucket `-upload` publica um evento `OBJECT_FINALIZE` no topico `new-doc` a cada upload. A SA `pubsub-cloud-run-invoker` sera usada pelo Pub/Sub para chamar o servico privado, por isso precisa de `roles/run.invoker`. O service agent do Pub/Sub ja tem `serviceAccountTokenCreator` (pre-configurado).

### Passo 1: Conferir os buckets pre-criados

```bash
gcloud storage ls | grep -E -- "-(upload|processed)/"
```

### Passo 2: Notificacao do bucket para o topico `new-doc`

```bash
gcloud storage buckets notifications create \
  --topic="$TOPIC_NAME" \
  --payload-format=json \
  --event-types=OBJECT_FINALIZE \
  "$UPLOAD_BUCKET"
```

### Passo 3: Service account (pre-criada) e permissao

```bash
gcloud iam service-accounts describe "$INVOKER_SA" >/dev/null 2>&1 || \
  gcloud iam service-accounts create "$INVOKER_SA_NAME" --display-name "PubSub Cloud Run Invoker"

gcloud run services add-iam-policy-binding "$SERVICE_NAME" \
  --member="serviceAccount:${INVOKER_SA}" \
  --role=roles/run.invoker \
  --region "$REGION"
```

### Explicacao dos parametros
- `notifications create --topic --payload-format=json --event-types=OBJECT_FINALIZE`: cria o topico (se necessario) e publica um JSON a cada upload concluido.
- `describe || create`: so cria a SA se ela nao existir.
- `roles/run.invoker`: permite a SA invocar o `pdf-converter`.

### Resultado esperado
- `Created notification config ...` e binding `run.invoker` para `pubsub-cloud-run-invoker`.

> Click **Check my progress**: *Create a service account*.

---

## TAREFA 5: Testar o servico Cloud Run

### Conceito
Como o servico e privado, uma chamada anonima deve ser negada e uma chamada com token de identidade deve funcionar.

### Passo 1: Capturar a URL e testar

```bash
export SERVICE_URL=$(gcloud run services describe "$SERVICE_NAME" \
  --region "$REGION" \
  --format "value(status.url)")
echo "$SERVICE_URL"

# Anonimo: 403 Forbidden
curl -s -X GET "$SERVICE_URL" | grep -o "403 Forbidden"

# Autenticado: pronto para processar
curl -s -X GET -H "Authorization: Bearer $(gcloud auth print-identity-token)" "$SERVICE_URL"
```

### Resultado esperado
- Anonimo: `403 Forbidden`.
- Autenticado: `Ready to process POST requests from Cloud Storage trigger`.

---

## TAREFA 6: Configurar o gatilho do Cloud Storage

### Conceito
A assinatura **push** `pdf-conv-sub` entrega cada mensagem do topico `new-doc` ao Cloud Run, autenticada com token OIDC da SA invoker.

### Passo 1: Criar a assinatura

```bash
gcloud pubsub subscriptions create "$SUBSCRIPTION_NAME" \
  --topic "$TOPIC_NAME" \
  --push-endpoint="$SERVICE_URL" \
  --push-auth-service-account="$INVOKER_SA"
```

### Resultado esperado
- `Created subscription [projects/<PROJECT_ID>/subscriptions/pdf-conv-sub]`.

> Click **Check my progress**: *Confirm the Pub/Sub subscription*.

---

## TAREFA 7: Testar a notificacao do Cloud Storage

### Conceito
Teste ponta a ponta: arquivos enviados ao `-upload` sao convertidos, os PDFs aparecem no `-processed` e os originais sao apagados.

### Passo 1: Enviar os arquivos de exemplo

```bash
gcloud storage cp -r "gs://spls/gsp762/*" "$UPLOAD_BUCKET"
```

### Passo 2: Acompanhar a conversao

```bash
# Originais sumindo do -upload (rode algumas vezes)
gcloud storage ls "$UPLOAD_BUCKET"

# PDFs no -processed
sleep 60
gcloud storage ls "gs://${PROCESSED_BUCKET_NAME}"

# Logs com o filtro "Converting" (Console: Cloud Run > pdf-converter > LOGS)
gcloud logging read \
  "resource.type=\"cloud_run_revision\" AND resource.labels.service_name=\"${SERVICE_NAME}\" AND textPayload:\"Converting\"" \
  --limit=10 --freshness=15m --format="value(textPayload)"
```

### Resultado esperado
- Bucket `-upload` vazio apos alguns minutos.
- Bucket `-processed` com um PDF para cada arquivo (`.jpg`, `.doc`, `.docx`, `.xls`, `.xlsx`).

---

## Validacao

```bash
gcloud run services describe "$SERVICE_NAME" --region "$REGION" \
  --format="yaml(status.url,spec.template.spec.containers[0].resources.limits,spec.template.spec.containers[0].env)"

gcloud run services get-iam-policy "$SERVICE_NAME" --region "$REGION"

gcloud storage buckets notifications list "$UPLOAD_BUCKET"
gcloud pubsub subscriptions describe "$SUBSCRIPTION_NAME" \
  --format="yaml(topic,pushConfig.pushEndpoint,pushConfig.oidcToken.serviceAccountEmail)"

gcloud storage ls "gs://${PROCESSED_BUCKET_NAME}"
```

Checklist esperado:
- imagem `pdf-converter` no repositorio `pdf-converter`;
- servico privado com `memory: 2Gi`, `PDF_BUCKET` e ate 3 instancias;
- `pubsub-cloud-run-invoker` com `run.invoker`;
- notificacao `OBJECT_FINALIZE` -> `new-doc` e assinatura `pdf-conv-sub`;
- PDFs no bucket `-processed`.

---

## Resumo Visual da Arquitetura

```text
[gs://spls/gsp762/*] --cp--> [gs://PROJECT_ID-upload]
                                   |          ^
                    OBJECT_FINALIZE|          | delete original
                                   v          |
                            [Topic new-doc]   |
                                   |          |
              push pdf-conv-sub    |          |
              (OIDC invoker SA)    v          |
              [Cloud Run pdf-converter] ------+
               (REGION, 2Gi, max 3, privado)
               server Go + LibreOffice
                                   |
                                   | upload PDF (PDF_BUCKET)
                                   v
                       [gs://PROJECT_ID-processed]

[Artifact Registry pdf-converter/pdf-converter] --> imagem do servico
```

---

## Troubleshooting

- `go build` falha:
  - confira se esta em `~/pet-theory/lab03` (precisa de `notification.go` e `gcs.go`); rode `go mod tidy` se faltar dependencia.
- `artifacts repositories create` diz que ja existe:
  - siga para o `gcloud builds submit`.
- Chamada autenticada retorna `403`:
  - confira o token (`gcloud auth print-identity-token`) e a conta ativa.
- Arquivos ficam no `-upload` e nao viram PDF:
  - confira a notificacao do bucket, o `pushEndpoint` da assinatura e o binding `run.invoker` da SA; veja os logs do servico.
- Erros de conversao nos logs:
  - confira `--memory=2Gi` e o `Dockerfile` com o LibreOffice.
- PDFs nao chegam ao `-processed`:
  - `PDF_BUCKET` deve ser o nome do bucket **sem** `gs://`.
- `SERVICE_URL` vazio apos reconectar:
  - rode novamente o `export SERVICE_URL=...` da Tarefa 5.

---

## Limpeza (Opcional)

Nao remova os buckets nem a SA `pubsub-cloud-run-invoker`: sao pre-criados pelo lab.

```bash
gcloud pubsub subscriptions delete "$SUBSCRIPTION_NAME" --quiet
gcloud storage buckets notifications delete "$UPLOAD_BUCKET"
gcloud pubsub topics delete "$TOPIC_NAME" --quiet
gcloud run services delete "$SERVICE_NAME" --region "$REGION" --quiet
gcloud artifacts repositories delete "$AR_REPO" --location="$REGION" --quiet
```

---

## Conceitos-Chave

- Go no Cloud Run: binario compilado + dependencias do sistema (LibreOffice) na mesma imagem.
- Cloud Build + Artifact Registry: build remoto e armazenamento da imagem por regiao.
- Servico privado: `--no-allow-unauthenticated` + `run.invoker` para quem chama.
- Notificacoes do Cloud Storage: eventos `OBJECT_FINALIZE` publicados no Pub/Sub sem codigo extra.
- Push autenticado: Pub/Sub chama o Cloud Run com token OIDC da SA invoker.
- Diretorio temporario por requisicao: processamento seguro com concorrencia.

---

## Fluxo Final

1. Definir variaveis (`REGION` do lab).
2. Clonar `Deleplace/pet-theory` e entrar em `lab03`.
3. Substituir o `server.go` e compilar com `go build`.
4. Atualizar o `Dockerfile` (Debian + LibreOffice), criar o repositorio `pdf-converter` e gerar a imagem.
5. Deploy privado com `2Gi`, `PDF_BUCKET` e ate 3 instancias.
6. Criar a notificacao do `-upload` para `new-doc` e dar `run.invoker` a SA.
7. Testar acesso anonimo x autenticado.
8. Criar a assinatura push `pdf-conv-sub`.
9. Enviar os arquivos de exemplo e conferir os PDFs no `-processed`.
