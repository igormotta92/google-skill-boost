# Build and Deploy a Docker Image to a Kubernetes Cluster: Challenge Lab

## Visao Geral

Em um challenge lab voce recebe um cenario e um conjunto de tarefas, sem passo a passo. Voce usa o que aprendeu no curso para resolver cada tarefa, e um sistema de avaliacao automatica indica se cada uma foi concluida corretamente. Para fazer 100%, todas as tarefas precisam ser concluidas dentro do tempo.

Cenario: o time de desenvolvimento quer adotar microservicos em containers. Voce precisa testar se a aplicacao de exemplo `echo-web` (Go, com `Dockerfile` e contexto prontos) pode ser implantada no GKE: baixar o codigo, gerar a imagem com tag para o Container Registry, enviar a imagem, criar o cluster e fazer o deploy.

## Introducao

Este guia converte o challenge lab (GSP304) para execucao via CLI com `gcloud`, `docker`, `kubectl` e `curl`. Como e um challenge lab, a sequencia de comandos foi inferida a partir dos criterios de cada tarefa.

Nomes exigidos:
- imagem `echo-app` com tag `v1` em `gcr.io`;
- cluster `echo-cluster` na `ZONE` do lab, com **2 nodes `e2-standard-2`**;
- deployment `echo-web`;
- aplicacao escuta na porta `8000`, mas o Service deve responder na porta `80`.

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export ZONE="<ZONE_DO_LAB>"   # ajuste conforme o lab

export CLUSTER_NAME="echo-cluster"
export DEPLOYMENT_NAME="echo-web"
export IMAGE_NAME="echo-app"
export IMAGE_TAG="v1"
export IMAGE_URI="gcr.io/${PROJECT_ID}/${IMAGE_NAME}:${IMAGE_TAG}"
export APP_PORT="8000"
export SERVICE_PORT="80"

export SOURCE_ARCHIVE="gs://${PROJECT_ID}/echo-web.tar.gz"
export WORK_DIR="$HOME/echo-web"

gcloud config set project "$PROJECT_ID"
gcloud config set compute/zone "$ZONE"
```

### Explicacao dos parametros
- `IMAGE_URI`: hostname `gcr.io` exigido pelo lab + projeto + imagem + tag.
- `SOURCE_ARCHIVE`: arquivo do codigo no bucket do projeto (`gs://[PROJECT_ID]`).

---

## TAREFA 1: Criar o cluster Kubernetes

### Conceito
Ambiente de teste com capacidade limitada: cluster zonal com apenas 2 nodes `e2-standard-2`.

### Passo 1: Criar o cluster

```bash
gcloud container clusters create "$CLUSTER_NAME" \
  --zone="$ZONE" \
  --num-nodes=2 \
  --machine-type=e2-standard-2
```

### Passo 2: Obter credenciais

```bash
gcloud container clusters get-credentials "$CLUSTER_NAME" --zone="$ZONE"
kubectl get nodes
```

### Explicacao dos parametros
- `--num-nodes=2 --machine-type=e2-standard-2`: limite de capacidade do lab.
- `--zone`: o check procura o cluster na `ZONE` do lab.

### Resultado esperado
- 2 nodes `Ready`.

> Click **Check my progress**: *Create the Kubernetes cluster*.

---

## TAREFA 2: Gerar a imagem Docker com tag

### Conceito
O `echo-web.tar.gz` traz o `Dockerfile` e o contexto da aplicacao. A imagem e gerada ja com o nome completo de destino (`gcr.io/PROJECT_ID/echo-app:v1`).

### Passo 1: Baixar e extrair o codigo

```bash
mkdir -p "$WORK_DIR" && cd "$WORK_DIR"
gcloud storage cp "$SOURCE_ARCHIVE" .
tar -xzf echo-web.tar.gz

# Localizar o diretorio do Dockerfile (contexto do build)
export BUILD_CONTEXT="$(dirname "$(find "$WORK_DIR" -name Dockerfile | head -n1)")"
echo "$BUILD_CONTEXT"
grep -i -E '^(FROM|EXPOSE)' "$BUILD_CONTEXT/Dockerfile"
```

### Passo 2: Build da imagem

```bash
docker build -t "$IMAGE_URI" "$BUILD_CONTEXT"
docker images | grep "$IMAGE_NAME"
```

### Explicacao dos parametros
- `find ... -name Dockerfile`: descobre o contexto sem depender da estrutura interna do `.tar.gz`.
- `docker build -t IMAGE_URI CONTEXTO`: gera a imagem ja com o nome do registry e a tag `v1`.

### Resultado esperado
- Imagem `gcr.io/<PROJECT_ID>/echo-app:v1` listada no `docker images`.

---

## TAREFA 3: Enviar a imagem para o Container Registry

### Conceito
O Docker precisa de credenciais para o `gcr.io`. O `configure-docker` registra o `gcloud` como credential helper.

### Passo 1: Autenticar e fazer push

```bash
gcloud auth configure-docker --quiet
docker push "$IMAGE_URI"
```

### Passo 2: Validar no registry

```bash
gcloud container images list-tags "gcr.io/${PROJECT_ID}/${IMAGE_NAME}"
```

### Explicacao dos parametros
- `gcloud auth configure-docker`: configura autenticacao para `gcr.io`.
- `docker push`: envia a imagem com a tag `v1`.

### Resultado esperado
- Tag `v1` listada para `gcr.io/<PROJECT_ID>/echo-app`.

> Click **Check my progress**: *An application image with a v1 tag has been pushed to the gcr.io repository*.

---

## TAREFA 4: Fazer deploy da aplicacao no cluster

### Conceito
O deployment `echo-web` roda a imagem `v1`. Um Service `LoadBalancer` expoe a porta `80` para fora e encaminha para a porta `8000` do container.

### Passo 1: Criar o deployment

```bash
kubectl create deployment "$DEPLOYMENT_NAME" --image="$IMAGE_URI"
kubectl rollout status deployment/"$DEPLOYMENT_NAME"
```

> Click **Check my progress**: *Check that an application has been deployed to the cluster*.

### Passo 2: Expor na porta 80

```bash
kubectl expose deployment "$DEPLOYMENT_NAME" \
  --type=LoadBalancer \
  --port="$SERVICE_PORT" \
  --target-port="$APP_PORT"
```

### Passo 3: Testar

```bash
until EXTERNAL_IP="$(kubectl get svc "$DEPLOYMENT_NAME" -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')" && [ -n "$EXTERNAL_IP" ]; do
  echo "Aguardando EXTERNAL-IP..."; sleep 5
done
export EXTERNAL_IP

curl -s "http://${EXTERNAL_IP}" | head -n 20
```

### Explicacao dos parametros
- `kubectl create deployment --image`: cria o deployment com a imagem do registry.
- `expose --type=LoadBalancer --port=80 --target-port=8000`: porta publica 80 -> porta 8000 da app.

### Resultado esperado
- Pagina do Echo-app com dados do ambiente em que a aplicacao roda.

> Click **Check my progress**: *Test that a service exists that responds to requests like Echo-app*.

---

## Validacao

```bash
gcloud container clusters describe "$CLUSTER_NAME" --zone="$ZONE" \
  --format="value(status,currentNodeCount,nodeConfig.machineType)"

gcloud container images list-tags "gcr.io/${PROJECT_ID}/${IMAGE_NAME}"

kubectl get deployment "$DEPLOYMENT_NAME" \
  -o=jsonpath='{.spec.template.spec.containers[0].image}'; echo
kubectl get svc "$DEPLOYMENT_NAME"

curl -s -o /dev/null -w "%{http_code}\n" "http://${EXTERNAL_IP}"
```

Checklist esperado:
- `echo-cluster` na `ZONE`, 2 nodes `e2-standard-2`;
- `gcr.io/<PROJECT_ID>/echo-app:v1` no registry;
- deployment `echo-web` com essa imagem;
- Service `echo-web` LoadBalancer `80 -> 8000` respondendo `200`.

---

## Resumo Visual da Arquitetura

```text
[gs://PROJECT_ID/echo-web.tar.gz]
        |
        | gcloud storage cp + tar
        v
[Cloud Shell] docker build -> gcr.io/PROJECT_ID/echo-app:v1
        |
        | docker push
        v
[Container Registry gcr.io]
        |
        | kubectl create deployment
        v
[GKE echo-cluster] (ZONE, 2 x e2-standard-2)
  [Deployment echo-web] Pods :8000
        ^
        | Service echo-web LoadBalancer :80 -> 8000
        |
[Internet] -- curl http://EXTERNAL_IP
```

---

## Troubleshooting

- `504 Gateway Timeout`:
  - a app pode ainda estar iniciando; ou ha divergencia entre a porta do `Dockerfile` (`8000`) e o `--target-port`.
- Sem pontuacao nos ultimos checks:
  - o cluster foi criado fora da `ZONE` do lab; recrie na zona correta.
- `docker push` com `denied` / `unauthorized`:
  - rode `gcloud auth configure-docker --quiet` e confira o `PROJECT_ID` no `IMAGE_URI`.
- `docker push` com erro de repositorio `gcr.io` inexistente (projetos com Container Registry migrado para Artifact Registry):
  ```bash
  gcloud artifacts repositories create gcr.io --repository-format=docker --location=us
  ```
- `find` nao encontrou o `Dockerfile`:
  - liste o conteudo com `tar -tzf echo-web.tar.gz` e ajuste `BUILD_CONTEXT`.

---

## Limpeza (Opcional)

```bash
kubectl delete svc "$DEPLOYMENT_NAME"
kubectl delete deployment "$DEPLOYMENT_NAME"
gcloud container images delete "$IMAGE_URI" --quiet --force-delete-tags
gcloud container clusters delete "$CLUSTER_NAME" --zone="$ZONE" --quiet
```

---

## Conceitos-Chave

- Imagem com tag: nome completo `REGISTRY/PROJECT/IMAGEM:TAG` define para onde o `push` vai.
- `gcloud auth configure-docker`: integra o Docker com as credenciais do `gcloud`.
- Deployment: mantem os Pods da aplicacao rodando com a imagem definida.
- Service LoadBalancer: IP externo; `port` (publica) pode ser diferente de `targetPort` (container).

---

## Fluxo Final

1. Definir variaveis (`ZONE` do lab, `IMAGE_URI`).
2. Criar `echo-cluster` (2 x `e2-standard-2`) e obter credenciais.
3. Baixar e extrair `echo-web.tar.gz` e localizar o `Dockerfile`.
4. `docker build` com `gcr.io/PROJECT_ID/echo-app:v1`.
5. `configure-docker` + `docker push`.
6. `kubectl create deployment echo-web` e `expose` `80 -> 8000`.
7. Testar `curl http://EXTERNAL_IP`.
