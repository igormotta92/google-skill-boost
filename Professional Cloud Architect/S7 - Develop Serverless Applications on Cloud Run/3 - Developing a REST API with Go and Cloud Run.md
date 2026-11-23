# Developing a REST API with Go and Cloud Run

## Visao Geral

Ha doze anos, Lily fundou a rede de clinicas veterinarias Pet Theory. Com o crescimento da rede, ela passa mais tempo ao telefone com seguradoras do que tratando os animais. Se as seguradoras pudessem ver online os totais dos tratamentos...

Nos labs anteriores, Ruby (consultora) e Patrick (DevOps) migraram a base de clientes para o Firestore e liberaram agendamentos online. Como o time de Ops e uma unica pessoa, a solucao precisa ser serverless e de pouca manutencao.

Neste lab, voce ajuda Ruby e Patrick a dar as seguradoras acesso aos dados dos clientes **sem expor PII**, criando um gateway REST seguro no **Cloud Run** que mostra apenas o custo total dos tratamentos.

Objetivos:

- Desenvolver uma REST API em Go.
- Importar dados de teste de clientes no Firestore.
- Conectar a REST API ao Firestore.
- Fazer deploy da REST API no Cloud Run.

## Introducao

Este guia converte o lab (GSP761) para execucao via CLI com `gcloud`, `go` e `curl`, mantendo os nomes de recursos do enunciado. Os arquivos que o lab pede para criar no editor sao gerados com `cat`.

Plano em versoes:
- `rest-api:0.1`: endpoint `/v1/` de status;
- `rest-api:0.2`: endpoint `/v1/customer/{id}` lendo o Firestore e devolvendo os totais `proposed`, `approved` e `rejected`.

Recursos usados no lab: diretorio `pet-theory/lab08`, repositorio `my-repo` no Artifact Registry, servico `rest-api`, banco Firestore `(default)` e bucket `<PROJECT_ID>-customer`.

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
export REGION="<REGION_DO_LAB>"   # ajuste conforme o lab

# Imagem base do Dockerfile (o enunciado mostra "Distroless base image")
export DISTROLESS_IMAGE="<DISTROLESS_IMAGE_DO_LAB>"   # ex: gcr.io/distroless/base-debian12

export APP_DIR="$HOME/pet-theory/lab08"
export AR_REPO="my-repo"
export SERVICE_NAME="rest-api"
export IMAGE_BASE="${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/${SERVICE_NAME}"
export CUSTOMER_BUCKET="gs://${PROJECT_ID}-customer"
export EXPORT_FOLDER="2019-10-06T20:10:37_43617"

gcloud config set project "$PROJECT_ID"
gcloud config set run/region "$REGION"
```

### Explicacao dos parametros
- `IMAGE_BASE`: caminho no Artifact Registry (`REGION-docker.pkg.dev/PROJECT/REPO/IMAGEM`); a tag (`0.1`/`0.2`) e adicionada no build.
- `EXPORT_FOLDER`: pasta do export do Firestore com os dados de teste.

---

## TAREFA 1: Habilitar as APIs do Google

### Conceito
O lab ja habilita Cloud Build (`cloudbuild.googleapis.com`) e Cloud Run Admin (`run.googleapis.com`). Artifact Registry e Firestore tambem serao usados.

### Passo 1: Conferir/habilitar

```bash
gcloud services enable \
  cloudbuild.googleapis.com \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  firestore.googleapis.com

gcloud services list --enabled --filter="name:(cloudbuild.googleapis.com run.googleapis.com)"
```

---

## TAREFA 2: Desenvolver a REST API (v0.1)

### Conceito
Primeira versao: um servidor HTTP em Go com o endpoint `/v1/` que responde o status. O binario `server` e compilado no Cloud Shell e copiado para uma imagem **distroless** (minima, sem shell).

### Passo 1: Baixar o codigo inicial

```bash
cd "$HOME"
gcloud storage cp -r "gs://${PROJECT_ID}-assets/pet-theory" . && cd "$APP_DIR"
cat go.mod
```

### Passo 2: Criar o `main.go` (v0.1)

```bash
cat > main.go <<'EOF'
package main

import (
  "fmt"
  "log"
  "net/http"
  "os"
)

func main() {
  port := os.Getenv("PORT")
  if port == "" {
      port = "8080"
  }
  http.HandleFunc("/v1/", func(w http.ResponseWriter, r *http.Request) {
      fmt.Fprintf(w, "{status: 'running'}")
  })
  log.Println("Pets REST API listening on port", port)
  if err := http.ListenAndServe(":"+port, nil); err != nil {
      log.Fatalf("Error launching Pets REST API server: %v", err)
  }
}
EOF
```

### Passo 3: Criar o `Dockerfile` e compilar

```bash
cat > Dockerfile <<EOF
FROM ${DISTROLESS_IMAGE}
WORKDIR /usr/src/app
COPY server .
CMD [ "/usr/src/app/server" ]
EOF

go build -o server
ls -la
```

### Passo 4: Criar o repositorio e gerar a imagem `0.1`

```bash
gcloud artifacts repositories create "$AR_REPO" \
  --repository-format=docker \
  --location="$REGION" \
  --description="Docker repository for REST API"

gcloud builds submit --tag "${IMAGE_BASE}:0.1"

gcloud artifacts docker images list "${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}" \
  --include-tags --format="table(package,tags)"
```

> Click **Check my progress**: *Build an image with Cloud Build*.

### Passo 5: Deploy no Cloud Run

```bash
gcloud run deploy "$SERVICE_NAME" \
  --image "${IMAGE_BASE}:0.1" \
  --region "$REGION" \
  --allow-unauthenticated \
  --max-instances=2

export SERVICE_URL="$(gcloud run services describe "$SERVICE_NAME" --region "$REGION" --format='value(status.url)')"
echo "$SERVICE_URL"
```

> Click **Check my progress**: *REST API service deployed*.

### Passo 6: Testar

```bash
curl -s "${SERVICE_URL}/v1/"
```

### Explicacao dos parametros
- `<<'EOF'` no `main.go`: grava o codigo sem expansao do shell.
- `go build -o server`: gera o binario copiado para a imagem.
- `FROM distroless`: imagem minima; o container so executa o binario.
- `--allow-unauthenticated`: API publica (autenticacao fica para um passo futuro do plano).
- `--max-instances=2`: limite de escala.

### Resultado esperado
- `{status: 'running'}` em `/v1/`.

### Resumo visual

```text
[lab08/main.go] --go build--> [server] + [Dockerfile distroless]
        |
        | gcloud builds submit
        v
[Artifact Registry my-repo/rest-api:0.1] (REGION)
        |
        | gcloud run deploy (publico, max 2)
        v
[Cloud Run rest-api] -- GET /v1/ --> {status: 'running'}
```

---

## TAREFA 3: Importar os dados de teste de clientes

### Conceito
O Firestore (modo nativo) recebe um export pronto com 10 clientes e tratamentos propostos. O import le o export a partir de um bucket do Cloud Storage.

### Passo 1: Criar o banco Firestore

Console (passo do lab): **Databases > Firestore > Create a Firestore database** > **Standard Edition** > Security rules **Open** > Location type **Region** > `REGION` > **Create Database**.

Equivalente CLI:

```bash
gcloud firestore databases create \
  --location="$REGION" \
  --type=firestore-native

gcloud firestore databases list --format="table(name,locationId,type)"
```

> Click **Check my progress**: *Firestore database created*.

### Passo 2: Copiar o export para um bucket

```bash
gcloud storage buckets create "$CUSTOMER_BUCKET" \
  --default-storage-class=standard \
  --location="$REGION"

gcloud storage cp -r "gs://spls/gsp645/${EXPORT_FOLDER}" "$CUSTOMER_BUCKET"
```

### Passo 3: Importar no Firestore

```bash
gcloud beta firestore import "${CUSTOMER_BUCKET}/${EXPORT_FOLDER}/"
```

### Explicacao dos parametros
- `firestore databases create --type=firestore-native`: banco `(default)` no modo nativo.
- `buckets create --location=REGION`: bucket na mesma regiao do banco.
- `firestore import gs://.../`: restaura colecoes e documentos do export.

### Resultado esperado
- Colecao `customers` com os clientes (Console: **Firestore > customers**).

### Resumo visual

```text
[gs://spls/gsp645/2019-10-06T20:10:37_43617]
        |
        | gcloud storage cp -r
        v
[gs://PROJECT_ID-customer/2019-10-06T20:10:37_43617/]
        |
        | gcloud beta firestore import
        v
[Firestore (default)] (REGION) -- colecao customers/{email}/treatments
```

---

## TAREFA 4: Conectar a REST API ao Firestore

### Conceito
A v0.2 usa o cliente Go do Firestore, o roteador `gorilla/mux` (rota `/v1/customer/{id}`) e o middleware CORS do `gorilla/handlers`. Ela soma o custo dos tratamentos por status, sem expor dados pessoais. Cliente inexistente deve retornar erro.

### Passo 1: Substituir o `main.go` (v0.2)

O codigo usa crases e `%s`, por isso o heredoc e `<<'EOF'`. O ID do projeto entra depois via `sed` no placeholder `__PROJECT_ID__`:

```bash
cd "$APP_DIR"

cat > main.go <<'EOF'
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"

	"cloud.google.com/go/firestore"
	"github.com/gorilla/handlers"
	"github.com/gorilla/mux"
	"google.golang.org/api/iterator"
)

var client *firestore.Client

func main() {
	var err error
	ctx := context.Background()
	client, err = firestore.NewClient(ctx, "__PROJECT_ID__")
	if err != nil {
		log.Fatalf("Error initializing Cloud Firestore client: %v", err)
	}

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	r := mux.NewRouter()
	r.HandleFunc("/v1/", rootHandler)
	r.HandleFunc("/v1/customer/{id}", customerHandler)

	log.Println("Pets REST API listening on port", port)
	cors := handlers.CORS(
		handlers.AllowedHeaders([]string{"X-Requested-With", "Authorization", "Origin"}),
		handlers.AllowedOrigins([]string{"https://storage.googleapis.com"}),
		handlers.AllowedMethods([]string{"GET", "HEAD", "POST", "OPTIONS", "PATCH", "CONNECT"}),
	)

	if err := http.ListenAndServe(":"+port, cors(r)); err != nil {
		log.Fatalf("Error launching Pets REST API server: %v", err)
	}
}

func rootHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "{status: 'running'}")
}

func customerHandler(w http.ResponseWriter, r *http.Request) {
	id := mux.Vars(r)["id"]
	ctx := context.Background()
	customer, err := getCustomer(ctx, id)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		fmt.Fprintf(w, `{"status": "fail", "data": '%s'}`, err)
		return
	}
	if customer == nil {
		w.WriteHeader(http.StatusNotFound)
		msg := fmt.Sprintf("`Customer \"%s\" not found`", id)
		fmt.Fprintf(w, fmt.Sprintf(`{"status": "fail", "data": {"title": %s}}`, msg))
		return
	}
	amount, err := getAmounts(ctx, customer)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		fmt.Fprintf(w, `{"status": "fail", "data": "Unable to fetch amounts: %s"}`, err)
		return
	}
	data, err := json.Marshal(amount)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		fmt.Fprintf(w, `{"status": "fail", "data": "Unable to fetch amounts: %s"}`, err)
		return
	}
	fmt.Fprintf(w, fmt.Sprintf(`{"status": "success", "data": %s}`, data))
}

type Customer struct {
	Email string `firestore:"email"`
	ID    string `firestore:"id"`
	Name  string `firestore:"name"`
	Phone string `firestore:"phone"`
}

func getCustomer(ctx context.Context, id string) (*Customer, error) {
	query := client.Collection("customers").Where("id", "==", id)
	iter := query.Documents(ctx)

	var c Customer
	for {
		doc, err := iter.Next()
		if err == iterator.Done {
			break
		}
		if err != nil {
			return nil, err
		}
		err = doc.DataTo(&c)
		if err != nil {
			return nil, err
		}
	}
	return &c, nil
}

func getAmounts(ctx context.Context, c *Customer) (map[string]int64, error) {
	if c == nil {
		return map[string]int64{}, fmt.Errorf("Customer should be non-nil: %v", c)
	}
	result := map[string]int64{
		"proposed": 0,
		"approved": 0,
		"rejected": 0,
	}
	query := client.Collection(fmt.Sprintf("customers/%s/treatments", c.Email))
	if query == nil {
		return map[string]int64{}, fmt.Errorf("Query is nil: %v", c)
	}
	iter := query.Documents(ctx)
	for {
		doc, err := iter.Next()
		if err == iterator.Done {
			break
		}
		if err != nil {
			return nil, err
		}
		treatment := doc.Data()
		result[treatment["status"].(string)] += treatment["cost"].(int64)
	}
	return result, nil
}
EOF

sed -i "s/__PROJECT_ID__/${PROJECT_ID}/" main.go
grep -n "firestore.NewClient" main.go
```

### Explicacao dos parametros
- `firestore.NewClient(ctx, PROJECT_ID)`: cliente do Firestore do projeto (o enunciado pede o ID do projeto nesse ponto).
- `mux.NewRouter()` + `/v1/customer/{id}`: rota com parametro.
- `handlers.CORS(...)`: libera chamadas do front-end hospedado em `storage.googleapis.com`.
- `getCustomer`: busca o cliente pelo campo `id`.
- `getAmounts`: soma `cost` por `status` na subcolecao `customers/{email}/treatments`.

### Resultado esperado
- `main.go` com o projeto do lab no `NewClient`, pronto para compilar.

---

## TAREFA 6: Quiz

Respostas:

- **Qual funcao responde a URLs com o padrao `/v1/customer/`?** `customerHandler`.
- **Qual instrucao retorna sucesso ao cliente?** a que usa `fmt.Sprintf` com `{"status": "success", "data": %s}`.
- **Quais funcoes leem do Firestore?** `getCustomer` e `getAmounts`.

---

## TAREFA 7: Fazer deploy de uma nova revisao (v0.2)

### Conceito
Novo binario, nova imagem com tag `0.2` e nova revisao do mesmo servico. A URL do Cloud Run nao muda.

### Passo 1: Compilar e gerar a imagem `0.2`

```bash
cd "$APP_DIR"
go build -o server

gcloud builds submit --tag "${IMAGE_BASE}:0.2"
```

> Click **Check my progress**: *Build image revision 0.2*.

### Passo 2: Deploy da nova revisao

```bash
gcloud run deploy "$SERVICE_NAME" \
  --image "${IMAGE_BASE}:0.2" \
  --region "$REGION" \
  --allow-unauthenticated \
  --max-instances=2
```

### Passo 3: Testar os endpoints

```bash
curl -s "${SERVICE_URL}/v1/"; echo

for id in 22530 34216 70156 12345; do
  echo "== cliente $id"
  curl -s -w "  [HTTP %{http_code}]\n" "${SERVICE_URL}/v1/customer/${id}"
done
```

### Explicacao dos parametros
- Tag `0.2`: mantem a `0.1` no repositorio para rollback.
- `curl -w "%{http_code}"`: mostra o status HTTP de cada chamada.

### Resultado esperado
- `/v1/`: `{status: 'running'}`.
- `22530`: `{"status": "success", "data": {"approved":585,"proposed":1602,"rejected":489}}`.
- `34216`: totais do cliente.
- `70156`: todos os valores `0`.
- `12345`: erro (cliente inexistente).

### Resumo visual

```text
Antes:  [rest-api rev 1] image :0.1 -- /v1/ (status)

        main.go v0.2 (Firestore + mux + CORS) -> go build -> :0.2

Depois: [rest-api rev 2] image :0.2 (mesma URL)
          /v1/                -> {status: 'running'}
          /v1/customer/{id}   -> [Firestore customers/{email}/treatments]
                                 -> {"proposed","approved","rejected"}
```

---

## Validacao

```bash
gcloud artifacts docker images list "${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}" \
  --include-tags --format="table(package,tags)"

gcloud run services describe "$SERVICE_NAME" --region "$REGION" \
  --format="value(status.url,spec.template.spec.containers[0].image)"

gcloud firestore databases list --format="table(name,locationId,type)"

curl -s "${SERVICE_URL}/v1/customer/22530"; echo
```

Checklist esperado:
- imagens `rest-api:0.1` e `rest-api:0.2` em `my-repo`;
- servico `rest-api` publico usando a `0.2`;
- Firestore `(default)` na `REGION` com a colecao `customers`;
- `/v1/customer/22530` retornando os totais.

---

## Resumo Visual da Arquitetura

Compilado: estado final ao termino do lab. Os detalhes de cada etapa estao no `### Resumo visual` das Tarefas 2, 3 e 7.

```text
[Seguradora / curl]
        |
        | GET /v1/customer/{id}   (publico, CORS storage.googleapis.com)
        v
[Cloud Run rest-api] (REGION, max 2) <-- [my-repo/rest-api:0.2]
        |
        | firestore client (PROJECT_ID)
        v
[Firestore (default)] customers -> treatments (cost, status)
        ^
        | import
[gs://PROJECT_ID-customer] (export de teste)
```

---

## Troubleshooting

- `go build` falha com pacote nao encontrado:
  - rode `go mod tidy` (ou `go get cloud.google.com/go/firestore github.com/gorilla/mux github.com/gorilla/handlers`) e compile de novo.
- Container nao inicia (exec format / not found):
  - o binario e o `FROM` precisam ser compativeis; confira `DISTROLESS_IMAGE` com o valor do lab.
- `artifacts repositories create` diz que ja existe:
  - siga para o build; o repositorio ja foi criado.
- `firestore databases create` falha porque ja existe:
  - o banco ja foi criado (Console ou execucao anterior); siga para o import.
- `/v1/customer/22530` retorna zeros ou erro de permissao:
  - confira se o import terminou (`gcloud firestore operations list`) e se o `NewClient` usa o projeto correto (`grep NewClient main.go`).
- `SERVICE_URL` vazio apos reconectar:
  - rode novamente o `export SERVICE_URL=...` da Tarefa 2.

---

## Limpeza (Opcional)

```bash
gcloud run services delete "$SERVICE_NAME" --region "$REGION" --quiet
gcloud artifacts repositories delete "$AR_REPO" --location="$REGION" --quiet
gcloud storage rm -r "$CUSTOMER_BUCKET"
```

---

## Conceitos-Chave

- REST API serverless: Cloud Run escala por requisicao e ate zero.
- Imagem distroless: so o binario Go, menor superficie de ataque.
- Artifact Registry: imagens versionadas por tag (`0.1`, `0.2`) para deploy e rollback.
- Firestore import/export: carga de dados a partir de um export no Cloud Storage.
- Exposicao sem PII: a API devolve so agregados (totais por status), nao dados pessoais.
- Revisoes do Cloud Run: novo deploy cria revisao nova mantendo a mesma URL.

---

## Fluxo Final

1. Definir variaveis (`REGION` e imagem distroless do lab) e conferir as APIs.
2. Baixar `pet-theory/lab08`, criar `main.go` v0.1, `Dockerfile` e compilar.
3. Criar `my-repo`, build `rest-api:0.1` e deploy publico; testar `/v1/`.
4. Criar o Firestore na `REGION`, copiar o export para o bucket e importar.
5. Substituir o `main.go` pela v0.2 (Firestore + mux + CORS) com o projeto do lab.
6. Responder o quiz.
7. Compilar, build `rest-api:0.2`, deploy e testar `/v1/customer/{id}`.
