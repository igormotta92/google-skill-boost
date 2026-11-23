# Scale Out and Update a Containerized Application on a Kubernetes Cluster: Challenge Lab

## Visao Geral

Em um challenge lab voce recebe um cenario e um conjunto de tarefas, sem passo a passo. Voce usa o que aprendeu no curso para resolver cada tarefa, e um sistema de avaliacao automatica indica se cada uma foi concluida corretamente. Para fazer 100%, todas as tarefas precisam ser concluidas dentro do tempo.

Cenario: voce assume um ambiente de teste com a aplicacao `echo-app` rodando no cluster `echo-cluster` (deployment `echo-web`). Recebeu a versao `v2` do codigo e precisa atualizar a aplicacao de `v1` para `v2`, escalar para 2 instancias e confirmar que tudo esta respondendo.

## Introducao

Este guia converte o challenge lab (GSP305) para execucao via CLI com `gcloud`, `docker`, `kubectl` e `curl`. Como e um challenge lab, a sequencia de comandos foi inferida a partir dos criterios de cada tarefa.

Recursos e nomes do lab:
- cluster `echo-cluster` na `ZONE` do lab (pre-criado);
- deployment `echo-web`, imagem inicial `gcr.io/qwiklabs-resources/echo-app:v1`;
- arquivo `echo-web-v2.tar.gz` no bucket indicado no lab;
- nova imagem `gcr.io/<PROJECT_ID>/echo-app:v2`;
- Service na porta `80` -> container `8000`; 2 replicas no final.

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export ZONE="<ZONE_DO_LAB>"               # ajuste conforme o lab
export BUCKET_NAME="<BUCKET_NAME_DO_LAB>" # ajuste conforme o lab (sem gs://)

export CLUSTER_NAME="echo-cluster"
export DEPLOYMENT_NAME="echo-web"
export IMAGE_NAME="echo-app"
export IMAGE_V1="gcr.io/qwiklabs-resources/${IMAGE_NAME}:v1"
export IMAGE_V2="gcr.io/${PROJECT_ID}/${IMAGE_NAME}:v2"
export APP_PORT="8000"
export SERVICE_PORT="80"
export REPLICAS="2"
export WORK_DIR="$HOME/echo-web-v2"

gcloud config set project "$PROJECT_ID"
gcloud config set compute/zone "$ZONE"
```

### Passo 2: Descobrir o bucket do arquivo (se nao souber o nome)

```bash
for b in $(gcloud storage ls); do
  gcloud storage ls "${b}echo-web-v2.tar.gz" 2>/dev/null
done
```

Use o bucket retornado em `BUCKET_NAME`.

### Passo 3: Conferir o cluster pre-criado

```bash
gcloud container clusters list --format="table(name,location,status)"
```

**Resultado esperado:** `echo-cluster` em `RUNNING` (check verde no Console).

### Passo 4: Deploy inicial da v1 (comandos do lab)

```bash
gcloud container clusters get-credentials "$CLUSTER_NAME" --zone="$ZONE"

kubectl create deployment "$DEPLOYMENT_NAME" --image="$IMAGE_V1"
kubectl expose deployment "$DEPLOYMENT_NAME" --type=LoadBalancer --port "$SERVICE_PORT" --target-port "$APP_PORT"
```

### Explicacao dos parametros
- `IMAGE_V1`: imagem inicial fornecida pelo lab (projeto `qwiklabs-resources`).
- `IMAGE_V2`: nova imagem no `gcr.io` do **seu** projeto.
- `expose --port 80 --target-port 8000`: porta publica 80 para a porta 8000 da app.

---

## TAREFA 1: Gerar a imagem atualizada com a nova tag

### Conceito
O `echo-web-v2.tar.gz` traz o `Dockerfile` e o codigo da v2, que adiciona o numero de versao na resposta. A imagem e gerada com a tag `v2`.

### Passo 1: Baixar e extrair

```bash
mkdir -p "$WORK_DIR" && cd "$WORK_DIR"
gcloud storage cp "gs://${BUCKET_NAME}/echo-web-v2.tar.gz" .
tar -xzf echo-web-v2.tar.gz

export BUILD_CONTEXT="$(dirname "$(find "$WORK_DIR" -name Dockerfile | head -n1)")"
echo "$BUILD_CONTEXT"
```

### Passo 2: Build com a tag v2

```bash
docker build -t "$IMAGE_V2" "$BUILD_CONTEXT"
docker images | grep "$IMAGE_NAME"
```

### Explicacao dos parametros
- `find ... -name Dockerfile`: descobre o contexto sem depender da estrutura do `.tar.gz`.
- `docker build -t gcr.io/PROJECT_ID/echo-app:v2`: imagem pronta para o push.

### Resultado esperado
- Imagem `gcr.io/<PROJECT_ID>/echo-app:v2` local.

---

## TAREFA 2: Enviar a imagem para o Container Registry

### Conceito
A organizacao usa o hostname `gcr.io`. O push precisa da autenticacao do Docker via `gcloud`.

### Passo 1: Autenticar e fazer push

```bash
gcloud auth configure-docker --quiet
docker push "$IMAGE_V2"

gcloud container images list-tags "gcr.io/${PROJECT_ID}/${IMAGE_NAME}"
```

### Resultado esperado
- Tag `v2` listada em `gcr.io/<PROJECT_ID>/echo-app`.

> Click **Check my progress**: *Check that there is a tagged image in gcr.io for echo-app:v2.*

### Resumo visual

```text
[gs://BUCKET_NAME/echo-web-v2.tar.gz]
        |
        | gcloud storage cp + tar
        v
[Cloud Shell] docker build -> gcr.io/PROJECT_ID/echo-app:v2
        |
        | docker push
        v
[Container Registry gcr.io/PROJECT_ID/echo-app]  tags: v2
```

---

## TAREFA 3: Fazer deploy da versao atualizada no cluster

### Conceito
Trocar a imagem do deployment `echo-web` dispara um rolling update de `v1` para `v2`. O Service existente continua expondo a porta `80`.

### Passo 1: Atualizar a imagem

```bash
export CONTAINER_NAME="$(kubectl get deployment "$DEPLOYMENT_NAME" \
  -o=jsonpath='{.spec.template.spec.containers[0].name}')"
echo "$CONTAINER_NAME"

kubectl set image deployment/"$DEPLOYMENT_NAME" "${CONTAINER_NAME}=${IMAGE_V2}"
kubectl rollout status deployment/"$DEPLOYMENT_NAME"
```

### Passo 2: Conferir a exposicao externa

```bash
kubectl get svc "$DEPLOYMENT_NAME"
```

Se o Service nao existir (o deploy inicial nao foi feito), crie:

```bash
kubectl expose deployment "$DEPLOYMENT_NAME" --type=LoadBalancer --port "$SERVICE_PORT" --target-port "$APP_PORT"
```

### Explicacao dos parametros
- `CONTAINER_NAME`: nome do container lido do proprio deployment (no `create deployment`, e o nome da imagem: `echo-app`).
- `kubectl set image`: troca a imagem e dispara o rolling update.
- `rollout status`: aguarda os Pods da v2 ficarem prontos.

### Resultado esperado
- Deployment `echo-web` com imagem `gcr.io/<PROJECT_ID>/echo-app:v2`.

> Click **Check my progress**: *Deploy the updated application version (v2) to the Kubernetes cluster.*

### Resumo visual

```text
Antes:  [Deployment echo-web] gcr.io/qwiklabs-resources/echo-app:v1
          |
          | kubectl set image (rolling update)
          v
Depois: [Deployment echo-web] gcr.io/PROJECT_ID/echo-app:v2
          ^
          | Service echo-web LoadBalancer :80 -> 8000 (inalterado)
```

---

## TAREFA 4: Escalar a aplicacao

### Conceito
`kubectl scale` ajusta `spec.replicas`; o Service distribui o trafego entre os Pods.

### Passo 1: Escalar para 2 replicas

```bash
kubectl scale deployment "$DEPLOYMENT_NAME" --replicas="$REPLICAS"
kubectl rollout status deployment/"$DEPLOYMENT_NAME"
kubectl get pods -o wide
```

### Resultado esperado
- `READY 2/2` no deployment, 2 Pods em `Running`.

> Click **Check my progress**: *Scale out the kubernetes application so that it is running 2 replicas.*

### Resumo visual

```text
[Service echo-web LB :80]
        |
   +----+----+
   |         |
   v         v
[Pod v2]  [Pod v2]   (replicas: 2)
```

---

## TAREFA 5: Confirmar que a aplicacao esta rodando

### Conceito
Teste final pelo IP externo do Service: a resposta da v2 inclui o numero de versao.

### Passo 1: Capturar o IP e testar

```bash
until EXTERNAL_IP="$(kubectl get svc "$DEPLOYMENT_NAME" -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')" && [ -n "$EXTERNAL_IP" ]; do
  echo "Aguardando EXTERNAL-IP..."; sleep 5
done
export EXTERNAL_IP

curl -s -o /dev/null -w "%{http_code}\n" "http://${EXTERNAL_IP}"
curl -s "http://${EXTERNAL_IP}" | grep -i version
```

### Resultado esperado
- HTTP `200` e resposta com a versao `v2`.

> Click **Check my progress**: *Verify your deployed application service is responding correctly.*

---

## Validacao

```bash
gcloud container images list-tags "gcr.io/${PROJECT_ID}/${IMAGE_NAME}"

kubectl get deployment "$DEPLOYMENT_NAME" \
  -o=jsonpath='{.spec.replicas} {.status.readyReplicas} {.spec.template.spec.containers[0].image}'; echo

kubectl get svc "$DEPLOYMENT_NAME" -o=jsonpath='{.spec.ports[0].port} -> {.spec.ports[0].targetPort}'; echo

curl -s "http://${EXTERNAL_IP}"
```

Checklist esperado:
- `echo-app:v2` em `gcr.io/<PROJECT_ID>`;
- `echo-web` com 2 replicas prontas e imagem `v2`;
- Service `80 -> 8000` respondendo com a versao.

---

## Resumo Visual da Arquitetura

Compilado: estado final ao termino do lab. Os detalhes de build, update e scale estao no `### Resumo visual` das Tarefas 2, 3 e 4.

```text
[gcr.io/PROJECT_ID/echo-app:v2]
        |
        v
[GKE echo-cluster] (ZONE)
  [Deployment echo-web] replicas: 2, imagem v2, porta 8000
        ^
        | Service echo-web LoadBalancer :80 -> 8000
        |
[Internet] -- curl http://EXTERNAL_IP  (resposta com versao)
```

---

## Troubleshooting

- `504 Gateway Timeout`:
  - a app pode ainda estar iniciando; ou ha divergencia entre a porta do `Dockerfile` (`8000`) e o `--target-port` do Service.
- `BUCKET_NAME` desconhecido:
  - use o loop do Passo 2 das variaveis ou **Cloud Storage > Buckets** no Console.
- `docker push` com `denied` / `unauthorized`:
  - rode `gcloud auth configure-docker --quiet` e confira o `PROJECT_ID` no `IMAGE_V2`.
- `docker push` com erro de repositorio `gcr.io` inexistente:
  ```bash
  gcloud artifacts repositories create gcr.io --repository-format=docker --location=us
  ```
- `set image` sem efeito / `container not found`:
  - confira `CONTAINER_NAME` com `kubectl get deploy echo-web -o yaml | grep -A2 containers`.
- `AlreadyExists` no deploy inicial:
  - o deployment/Service ja existe; siga para a Tarefa 1.

---

## Limpeza (Opcional)

Nao remova o cluster `echo-cluster`: ele e pre-criado pelo lab.

```bash
kubectl delete svc "$DEPLOYMENT_NAME"
kubectl delete deployment "$DEPLOYMENT_NAME"
gcloud container images delete "$IMAGE_V2" --quiet --force-delete-tags
```

---

## Conceitos-Chave

- Versionamento por tag: `v1` e `v2` coexistem no registry; o deployment aponta para uma delas.
- Rolling update: `kubectl set image` troca os Pods gradualmente, sem derrubar o Service.
- Scale out: `kubectl scale --replicas` aumenta a capacidade horizontalmente.
- Service LoadBalancer: IP externo estavel durante update e scale.

---

## Fluxo Final

1. Definir variaveis (`ZONE`, `BUCKET_NAME` do lab) e conferir o `echo-cluster`.
2. Deploy inicial da v1 e Service `80 -> 8000` (comandos do lab).
3. Baixar `echo-web-v2.tar.gz`, `docker build` com tag `v2`.
4. `configure-docker` + `docker push` para `gcr.io/PROJECT_ID/echo-app:v2`.
5. `kubectl set image` no `echo-web` para a v2.
6. `kubectl scale --replicas=2`.
7. Testar `curl http://EXTERNAL_IP` e conferir a versao.
