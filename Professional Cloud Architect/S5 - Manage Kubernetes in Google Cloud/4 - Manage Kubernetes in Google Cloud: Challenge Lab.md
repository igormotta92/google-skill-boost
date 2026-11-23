# Manage Kubernetes in Google Cloud: Challenge Lab

## Visao Geral

Em um challenge lab voce recebe um cenario e um conjunto de tarefas, sem passo a passo. Voce usa o que aprendeu no curso para resolver cada tarefa, e um sistema de avaliacao automatica indica se cada uma foi concluida corretamente. Para fazer 100%, todas as tarefas precisam ser concluidas dentro do tempo.

Cenario: voce entrou ha poucos meses na Cymbal Shops, ja trabalhou com Docker, Artifact Registry e deployments no GKE, e agora precisa demonstrar ao time que sabe gerenciar rollouts e atualizacoes do novo e-commerce. Em um sandbox, voce vai:

- criar um cluster GKE com uma configuracao definida;
- habilitar o Managed Prometheus no cluster;
- fazer deploy de um manifesto Kubernetes e depurar os erros;
- criar uma metrica baseada em logs e uma alerting policy para os erros;
- corrigir o manifesto, containerizar o codigo e enviar a imagem ao Artifact Registry com Docker;
- expor um Service para a aplicacao e verificar a atualizacao.

## Introducao

Este guia converte o challenge lab (GSP510) para execucao via CLI com `gcloud`, `kubectl`, `docker` e `curl`. Como e um challenge lab, a sequencia de comandos foi inferida a partir dos criterios de cada tarefa.

Regras e pontos de atencao:
- Os valores `CLUSTER_NAME`, `ZONE`, `REGION`, `NAMESPACE_NAME`, `REPO_NAME`, `SERVICE_NAME` e o intervalo do PodMonitoring mudam a cada lab e aparecem no painel **Lab setup and access** / no enunciado: preencha as variaveis com eles.
- O repositorio `REPO_NAME` no Artifact Registry ja existe (pre-criado), com uma versao containerizada do `hello-app`.
- Nomes fixos exigidos: metrica `pod-image-errors`, policy `Pod Error Alert`, deployment `helloweb`, app/PodMonitoring `prometheus-test`.

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID

# Valores do painel/enunciado do lab (ajuste conforme o lab)
export CLUSTER_NAME="<CLUSTER_NAME_DO_LAB>"
export REGION="<REGION_DO_LAB>"
export ZONE="<ZONE_DO_LAB>"
export NAMESPACE_NAME="<NAMESPACE_DO_LAB>"
export REPO_NAME="<REPO_NAME_DO_LAB>"
export SERVICE_NAME="<SERVICE_NAME_DO_LAB>"
export MONITORING_INTERVAL="<INTERVAL_DO_LAB>"   # ex: 30s ("interval period" do enunciado)

# Nomes fixos do lab
export DEPLOYMENT_NAME="helloweb"
export PROM_APP_NAME="prometheus-test"
export METRIC_NAME="pod-image-errors"
export ALERT_NAME="Pod Error Alert"

# Imagens
export PROM_APP_IMAGE="nilebox/prometheus-example-app:latest"
export HELLO_IMAGE_V1="us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0"
export HELLO_IMAGE_V2="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/hello-app:v2"

gcloud config set project "$PROJECT_ID"
gcloud config set compute/zone "$ZONE"
gcloud config set compute/region "$REGION"
```

### Passo 2: Conferir o repositorio pre-criado

```bash
gcloud artifacts repositories list --location="$REGION" \
  --format="table(name,format,location)"
```

### Explicacao dos parametros
- `HELLO_IMAGE_V2`: segue a convencao do Artifact Registry `LOCATION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE:TAG`.
- `MONITORING_INTERVAL`: valor de `endpoints.interval` do PodMonitoring informado no enunciado.

### Resultado esperado
- Repositorio `REPO_NAME` (formato `DOCKER`) listado em `REGION`. Se estiver em outra localizacao, ajuste `REGION`/`HELLO_IMAGE_V2`.

---

## TAREFA 1: Criar o cluster GKE

### Conceito
Cluster zonal com canal de release **Regular**, versao padrao do canal e **cluster autoscaler** habilitado (3 nodes iniciais, minimo 2, maximo 6).

### Passo 1: Criar o cluster

```bash
gcloud container clusters create "$CLUSTER_NAME" \
  --zone="$ZONE" \
  --release-channel=regular \
  --num-nodes=3 \
  --enable-autoscaling \
  --min-nodes=2 \
  --max-nodes=6
```

### Passo 2: Obter credenciais

```bash
gcloud container clusters get-credentials "$CLUSTER_NAME" --zone="$ZONE"
kubectl get nodes
```

### Explicacao dos parametros
- `--release-channel=regular`: canal Regular; sem `--cluster-version`, usa a versao default do canal.
- `--num-nodes=3`: nodes iniciais no node pool padrao.
- `--enable-autoscaling --min-nodes=2 --max-nodes=6`: cluster autoscaler com limites.

### Resultado esperado
- Cluster `CLUSTER_NAME` em `RUNNING`, 3 nodes `Ready`.

> Click **Check my progress**: *Create a GKE cluster*.

---

## TAREFA 2: Habilitar o Managed Prometheus no cluster

### Conceito
A coleta gerenciada do Managed Service for Prometheus instala coletores no namespace `gmp-system`. A app de exemplo expoe metricas na porta `metrics`, e o **PodMonitoring** define o scrape dos Pods com `app: prometheus-test` no namespace do lab.

### Passo 1: Habilitar a coleta gerenciada

```bash
gcloud container clusters update "$CLUSTER_NAME" \
  --zone="$ZONE" \
  --enable-managed-prometheus
```

### Passo 2: Criar o namespace

```bash
kubectl create namespace "$NAMESPACE_NAME"
```

### Passo 3: Baixar e completar o `prometheus-app.yaml`

```bash
cd "$HOME"
gcloud storage cp gs://spls/gsp510/prometheus-app.yaml .

# Ver os <todo> (linhas 35-38)
grep -n '<todo>' prometheus-app.yaml
```

Preencher na ordem do arquivo (`image`, `name` do container, `name` da porta):

```bash
sed -i "s|image: <todo>|image: ${PROM_APP_IMAGE}|" prometheus-app.yaml
sed -i "0,/name: <todo>/s|name: <todo>|name: ${PROM_APP_NAME}|" prometheus-app.yaml
sed -i "0,/name: <todo>/s|name: <todo>|name: metrics|" prometheus-app.yaml

# Conferir: nenhum <todo> restante e valores nas linhas certas
grep -n '<todo>' prometheus-app.yaml
sed -n '30,40p' prometheus-app.yaml
```

Se a ordem dos campos no arquivo for diferente, edite manualmente com `nano prometheus-app.yaml` usando os valores: `image: nilebox/prometheus-example-app:latest`, `name: prometheus-test` (container) e `name: metrics` (porta).

### Passo 4: Fazer deploy da app no namespace

```bash
kubectl -n "$NAMESPACE_NAME" apply -f prometheus-app.yaml
kubectl -n "$NAMESPACE_NAME" get pods
```

### Passo 5: Baixar e completar o `pod-monitoring.yaml`

```bash
gcloud storage cp gs://spls/gsp510/pod-monitoring.yaml .

# Ver os <todo> (linhas 18-24)
grep -n '<todo>' pod-monitoring.yaml
```

A ordem dos `sed` importa: `app.kubernetes.io/name` antes de `name`, pois `name: <todo>` tambem casaria dentro dele.

```bash
sed -i "s|app.kubernetes.io/name: <todo>|app.kubernetes.io/name: ${PROM_APP_NAME}|" pod-monitoring.yaml
sed -i "s|name: <todo>|name: ${PROM_APP_NAME}|" pod-monitoring.yaml
sed -i "s|app: <todo>|app: ${PROM_APP_NAME}|" pod-monitoring.yaml
sed -i "s|interval: <todo>|interval: ${MONITORING_INTERVAL}|" pod-monitoring.yaml

grep -n '<todo>' pod-monitoring.yaml
cat pod-monitoring.yaml
```

### Passo 6: Aplicar o PodMonitoring no namespace

```bash
kubectl -n "$NAMESPACE_NAME" apply -f pod-monitoring.yaml
kubectl get podmonitoring -A
```

### Explicacao dos parametros
- `clusters update --enable-managed-prometheus`: liga a coleta gerenciada em cluster existente.
- `sed "0,/re/s|..|..|"`: substitui apenas a **primeira** ocorrencia (GNU sed, Cloud Shell).
- `metadata.name` / `labels.app.kubernetes.io/name`: identificam o PodMonitoring.
- `selector.matchLabels.app: prometheus-test`: seleciona os Pods da app de exemplo.
- `endpoints.interval`: frequencia de scrape exigida pelo lab.

### Resultado esperado
- Pods `prometheus-test-*` em `Running` no namespace do lab.
- PodMonitoring `prometheus-test` listado no namespace do lab.

> Click **Check my progress**: *Enable Managed Prometheus on the GKE cluster*.

### Resumo visual

```text
[GKE CLUSTER_NAME] (ZONE, regular, autoscaler 2-6)
  |
  +-- namespace NAMESPACE_NAME
  |     [Deployment prometheus-test] (nilebox/prometheus-example-app)
  |              ^  porta: metrics
  |              | scrape a cada MONITORING_INTERVAL
  |     [PodMonitoring prometheus-test] (matchLabels app: prometheus-test)
  |
  +-- namespace gmp-system
        [GMP collectors] --> [Managed Service for Prometheus]
```

---

## TAREFA 3: Fazer deploy da aplicacao no cluster

### Conceito
O manifesto `helloweb-deployment.yaml` vem com a imagem como `<todo>`. O deploy e feito assim mesmo para gerar o erro `InvalidImageName`, que sera usado na metrica e no alerta da Tarefa 4.

### Passo 1: Baixar os arquivos do hello-app

```bash
cd "$HOME"
gcloud storage cp -r gs://spls/gsp510/hello-app/ .
ls hello-app hello-app/manifests
```

### Passo 2: Criar o deployment no namespace

```bash
kubectl -n "$NAMESPACE_NAME" create -f hello-app/manifests/helloweb-deployment.yaml
```

### Passo 3: Inspecionar o erro

```bash
kubectl -n "$NAMESPACE_NAME" get deployment "$DEPLOYMENT_NAME"
kubectl -n "$NAMESPACE_NAME" get pods

kubectl -n "$NAMESPACE_NAME" get events --sort-by=.lastTimestamp | grep -i image
```

Console (passo do lab): **Kubernetes Engine > Workloads > helloweb** > barra vermelha **Pod errors: InvalidImageName** > **View details** / **Events**.

### Resultado esperado
- Pods com status `InvalidImageName`.
- Evento: `Failed to apply default image tag "<todo>": couldn't parse image reference "<todo>": invalid reference format`.

> Click **Check my progress**: *Deploy an application onto the GKE cluster*.

---

## TAREFA 4: Criar metrica baseada em logs e alerting policy

### Conceito
Os erros de imagem aparecem como logs de **Kubernetes Pod** com severidade **WARNING**. A metrica `pod-image-errors` (Counter) conta essas entradas e a policy `Pod Error Alert` dispara quando a soma em 10 minutos passa de 0.

### Passo 1: Montar a query (Logs Explorer)

Dica do lab: um unico Resource Type e uma unica Severity.

```text
resource.type="k8s_pod"
severity=WARNING
```

Equivalente CLI para confirmar os erros:

```bash
export POD_ERROR_FILTER='resource.type="k8s_pod"
severity=WARNING'

gcloud logging read "$POD_ERROR_FILTER" --limit=5 --freshness=30m \
  --format="table(timestamp,jsonPayload.reason,jsonPayload.message)"
```

**Resultado esperado:** entradas `InvalidImageName` / `Failed to apply default image tag "<todo>"...`.

### Passo 2: Criar a metrica `pod-image-errors`

Console (passo do lab): **Logs Explorer > Actions > Create Metric** > Metric type **Counter** > Log Metric Name `pod-image-errors`.

Equivalente CLI (metricas criadas pelo `gcloud` sao Counter por padrao):

```bash
gcloud logging metrics create "$METRIC_NAME" \
  --description="Erros/warnings de imagem em Pods" \
  --log-filter="$POD_ERROR_FILTER"

gcloud logging metrics describe "$METRIC_NAME"
```

### Passo 3: Criar a alerting policy `Pod Error Alert`

Console (passo do lab): **Monitoring > Alerting > Create Policy** > metrica `logging.googleapis.com/user/pod-image-errors` (desmarque **Active** se nao aparecer) > Rolling window `10 min` > function `Count` > Time series aggregation `Sum` > Threshold, **Any time series violates**, **Above threshold**, valor `0` > desabilite **Use notification channel** > nome `Pod Error Alert`.

Equivalente CLI (arquivo gerado com `cat` e variaveis):

```bash
cat > pod-error-alert-policy.json <<EOF
{
  "displayName": "${ALERT_NAME}",
  "combiner": "OR",
  "conditions": [
    {
      "displayName": "Kubernetes Pod - logging/user/${METRIC_NAME}",
      "conditionThreshold": {
        "filter": "resource.type = \"k8s_pod\" AND metric.type = \"logging.googleapis.com/user/${METRIC_NAME}\"",
        "aggregations": [
          {
            "alignmentPeriod": "600s",
            "perSeriesAligner": "ALIGN_COUNT",
            "crossSeriesReducer": "REDUCE_SUM"
          }
        ],
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0,
        "duration": "0s",
        "trigger": { "count": 1 }
      }
    }
  ],
  "enabled": true
}
EOF

gcloud alpha monitoring policies create --policy-from-file=pod-error-alert-policy.json

gcloud alpha monitoring policies list --filter="displayName=\"${ALERT_NAME}\"" \
  --format="table(displayName,enabled)"
```

### Explicacao dos parametros
- `resource.type="k8s_pod"` + `severity=WARNING`: eventos de Pod como `InvalidImageName`.
- `alignmentPeriod: 600s`: rolling window de 10 minutos.
- `ALIGN_COUNT`: rolling window function **Count**.
- `REDUCE_SUM`: time series aggregation **Sum**.
- `COMPARISON_GT` + `thresholdValue: 0`: **Above threshold** com valor 0.
- `trigger.count: 1`: **Any time series violates**.
- Sem `notificationChannels`: notificacao desabilitada.

### Resultado esperado
- Metrica `pod-image-errors` e policy `Pod Error Alert` criadas.

> Click **Check my progress**: *Create a logs-based metric and alerting policy*.

### Resumo visual

```text
[helloweb Pods] image: <todo> --> InvalidImageName
        |
        | eventos k8s_pod, severity=WARNING
        v
[Cloud Logging] --> [Metric pod-image-errors] (Counter)
                            |
                            v
            [Alert Pod Error Alert]
            (10 min, Count, Sum, > 0, sem notificacao)
```

---

## TAREFA 5: Atualizar e refazer o deploy da aplicacao

### Conceito
Corrigir a imagem no manifesto, **apagar** o deployment com erro e criar de novo a partir do manifesto corrigido.

### Passo 1: Corrigir a imagem no manifesto

```bash
cd "$HOME"
sed -i "s|image: <todo>|image: ${HELLO_IMAGE_V1}|" hello-app/manifests/helloweb-deployment.yaml

grep -n 'image:' hello-app/manifests/helloweb-deployment.yaml
```

### Passo 2: Apagar o deployment atual

```bash
kubectl -n "$NAMESPACE_NAME" delete deployment "$DEPLOYMENT_NAME"
```

### Passo 3: Fazer deploy do manifesto corrigido

```bash
kubectl -n "$NAMESPACE_NAME" create -f hello-app/manifests/helloweb-deployment.yaml
kubectl -n "$NAMESPACE_NAME" rollout status deployment/"$DEPLOYMENT_NAME"
kubectl -n "$NAMESPACE_NAME" get pods
```

### Explicacao dos parametros
- `sed "s|image: <todo>|...|"`: troca o placeholder pela imagem `hello-app:1.0`.
- `delete deployment` + `create -f`: recria o deployment do zero, como pede o lab.
- `rollout status`: espera os Pods ficarem prontos.

### Resultado esperado
- `helloweb` sem erros, Pods em `Running` (Workloads com status OK).

> Click **Check my progress**: *Update and re-deploy your app*.

### Resumo visual

```text
Antes:  [helloweb] image: <todo>           -> InvalidImageName
          |
          | sed (imagem 1.0) + kubectl delete + kubectl create
          v
Depois: [helloweb] image: .../hello-app:1.0 -> Pods Running
```

---

## TAREFA 6: Containerizar o codigo e fazer deploy no cluster

### Conceito
Atualizar o codigo para `Version: 2.0.0`, gerar a imagem com Docker, enviar ao repositorio `REPO_NAME` no Artifact Registry com a tag `v2`, atualizar o deployment `helloweb` e expor via LoadBalancer na porta `8080`.

### Passo 1: Atualizar a versao no `main.go`

```bash
cd "$HOME/hello-app"

sed -n '49p' main.go
sed -i '49s/1\.0\.0/2.0.0/' main.go
sed -n '49p' main.go
```

Se a linha 49 nao contiver a versao, localize com `grep -n 'Version' main.go` e ajuste o numero da linha.

### Passo 2: Build da imagem `v2`

```bash
gcloud auth configure-docker "${REGION}-docker.pkg.dev" --quiet

docker build -t "$HELLO_IMAGE_V2" .
docker images | grep hello-app
```

### Passo 3: Push para o Artifact Registry

```bash
docker push "$HELLO_IMAGE_V2"

gcloud artifacts docker images list "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}" \
  --include-tags --format="table(package,tags)"
```

### Passo 4: Atualizar a imagem do deployment

```bash
export CONTAINER_NAME="$(kubectl -n "$NAMESPACE_NAME" get deployment "$DEPLOYMENT_NAME" \
  -o=jsonpath='{.spec.template.spec.containers[0].name}')"
echo "$CONTAINER_NAME"

kubectl -n "$NAMESPACE_NAME" set image deployment/"$DEPLOYMENT_NAME" \
  "${CONTAINER_NAME}=${HELLO_IMAGE_V2}"
kubectl -n "$NAMESPACE_NAME" rollout status deployment/"$DEPLOYMENT_NAME"
```

### Passo 5: Expor o deployment com LoadBalancer

A porta do container vem do `EXPOSE` do Dockerfile:

```bash
export TARGET_PORT="$(grep -i '^EXPOSE' Dockerfile | awk '{print $2}')"
echo "$TARGET_PORT"

kubectl -n "$NAMESPACE_NAME" expose deployment "$DEPLOYMENT_NAME" \
  --name="$SERVICE_NAME" \
  --type=LoadBalancer \
  --port=8080 \
  --target-port="$TARGET_PORT"
```

### Passo 6: Testar a aplicacao

```bash
until EXTERNAL_IP="$(kubectl -n "$NAMESPACE_NAME" get svc "$SERVICE_NAME" -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')" && [ -n "$EXTERNAL_IP" ]; do
  echo "Aguardando EXTERNAL-IP..."; sleep 5
done
export EXTERNAL_IP

curl "http://${EXTERNAL_IP}:8080"
```

### Explicacao dos parametros
- `sed '49s/1\.0\.0/2.0.0/'`: altera a versao apenas na linha 49.
- `gcloud auth configure-docker REGION-docker.pkg.dev`: configura o Docker para autenticar no Artifact Registry.
- `docker build -t LOCATION-docker.pkg.dev/PROJECT/REPO/hello-app:v2 .`: imagem com o nome no padrao do Artifact Registry e tag `v2`.
- `kubectl set image`: troca a imagem do container e dispara rolling update.
- `expose --type=LoadBalancer --port=8080 --target-port=TARGET_PORT`: Service externo na porta 8080 apontando para a porta do container.

### Resultado esperado

```text
Hello, world!
Version: 2.0.0
Hostname: helloweb-xxxxxxxxxx-xxxxx
```

A pagina pode levar alguns minutos para responder.

> Click **Check my progress**: *Containerize your code and deploy it onto the cluster*.

### Resumo visual

```text
[hello-app/main.go] Version 2.0.0
        |
        | docker build + docker push (tag v2)
        v
[Artifact Registry] REGION-docker.pkg.dev/PROJECT_ID/REPO_NAME/hello-app:v2
        |
        | kubectl set image
        v
[Deployment helloweb] (NAMESPACE_NAME) <-- [Service SERVICE_NAME]
                                            LoadBalancer :8080 -> TARGET_PORT
                                                  ^
                                                  | curl EXTERNAL_IP:8080
                                            [Cliente]
```

---

## Validacao

```bash
# Cluster: canal, autoscaler e Managed Prometheus
gcloud container clusters describe "$CLUSTER_NAME" --zone="$ZONE" \
  --format="yaml(status,releaseChannel.channel,currentNodeCount,nodePools[0].autoscaling,monitoringConfig.managedPrometheusConfig)"

# Workloads do namespace
kubectl -n "$NAMESPACE_NAME" get deploy,pods,svc
kubectl -n "$NAMESPACE_NAME" get podmonitoring

# Imagem atual do helloweb
kubectl -n "$NAMESPACE_NAME" get deployment "$DEPLOYMENT_NAME" \
  -o=jsonpath='{.spec.template.spec.containers[0].image}'; echo

# Metrica e alerta
gcloud logging metrics describe "$METRIC_NAME" --format="value(name,filter)"
gcloud alpha monitoring policies list --filter="displayName=\"${ALERT_NAME}\"" --format="value(displayName,enabled)"

# Aplicacao
curl -s "http://${EXTERNAL_IP}:8080"
```

Checklist esperado:
- cluster com canal `REGULAR`, autoscaling `minNodeCount: 2` / `maxNodeCount: 6` e Managed Prometheus habilitado;
- `prometheus-test` e PodMonitoring `prometheus-test` no namespace do lab;
- metrica `pod-image-errors` e policy `Pod Error Alert`;
- `helloweb` com imagem `.../hello-app:v2` e Service `SERVICE_NAME` respondendo `Version: 2.0.0`.

---

## Resumo Visual da Arquitetura

Compilado: estado final ao termino do lab. Os detalhes de cada etapa estao no `### Resumo visual` das Tarefas 2, 4, 5 e 6.

```text
[GKE CLUSTER_NAME] (ZONE, regular, 3 nodes, autoscaler 2-6, GMP on)
  |
  +-- namespace NAMESPACE_NAME
  |     [prometheus-test] <--scrape-- [PodMonitoring prometheus-test]
  |     [helloweb] image hello-app:v2 <-- [Service SERVICE_NAME LB :8080]
  |                                               ^
  |                                               | curl
  |                                          [Cliente]
  +-- namespace gmp-system: [GMP collectors] --> [Managed Prometheus]

[Artifact Registry REPO_NAME] hello-app:v2  (docker push)
[Cloud Logging] --> [Metric pod-image-errors] --> [Alert Pod Error Alert]
```

---

## Troubleshooting

- Variaveis com `<..._DO_LAB>`:
  - preencha todas com os valores do painel/enunciado antes de rodar; comandos com placeholder falham.
- `create` do cluster demora ou falha por quota:
  - confirme cluster zonal (`--zone`) e os limites `--min-nodes=2 --max-nodes=6`.
- `sed` nao substituiu os `<todo>`:
  - rode `grep -n '<todo>' <arquivo>` e edite manualmente com `nano` usando os valores do enunciado.
- PodMonitoring no namespace errado:
  - aplique com `kubectl -n "$NAMESPACE_NAME" apply -f pod-monitoring.yaml`.
- Logs `InvalidImageName` nao aparecem na query:
  - aguarde 1-2 minutos apos o deploy da Tarefa 3; confira `resource.type="k8s_pod"` e `severity=WARNING`.
- `gcloud alpha monitoring` nao reconhecido:
  - instale o componente alpha ou crie a policy pelo Console (passo do lab).
- `docker push` com `denied` / `unauthorized`:
  - rode `gcloud auth configure-docker "${REGION}-docker.pkg.dev"` e confira se o repositorio esta em `REGION`.
- Pods com `ImagePullBackOff` apos o `set image`:
  - confira o nome completo da imagem (`HELLO_IMAGE_V2`) e se a tag `v2` existe no repositorio.
- `curl` sem resposta:
  - aguarde o LoadBalancer; use a porta `8080` na URL e confira `TARGET_PORT`.

---

## Limpeza (Opcional)

Nao remova o repositorio `REPO_NAME`: ele e pre-criado pelo lab.

```bash
kubectl -n "$NAMESPACE_NAME" delete svc "$SERVICE_NAME"
kubectl delete namespace "$NAMESPACE_NAME"

export POLICY_ID="$(gcloud alpha monitoring policies list --filter="displayName=\"${ALERT_NAME}\"" --format='value(name)')"
gcloud alpha monitoring policies delete "$POLICY_ID" --quiet
gcloud logging metrics delete "$METRIC_NAME" --quiet

gcloud container clusters delete "$CLUSTER_NAME" --zone="$ZONE" --quiet
```

---

## Conceitos-Chave

- Release channel: define a cadencia de upgrades do GKE (Rapid, Regular, Stable).
- Cluster autoscaler: ajusta o numero de nodes entre o minimo e o maximo conforme a demanda.
- Managed Prometheus: coleta gerenciada configurada por PodMonitoring (escopo de namespace).
- `InvalidImageName`: erro de referencia de imagem invalida; aparece em eventos/logs de `k8s_pod` com severidade WARNING.
- Logs-based metric + alerting policy: transformam erros em metrica e disparam incidentes por threshold.
- Artifact Registry: imagens no padrao `LOCATION-docker.pkg.dev/PROJECT/REPO/IMAGE:TAG`.
- `kubectl set image` + `kubectl expose`: atualizam a versao e publicam a aplicacao via LoadBalancer.

---

## Fluxo Final

1. Preencher as variaveis com os valores do lab e conferir o repositorio.
2. Criar o cluster (Regular, 3 nodes, autoscaler 2-6) e obter credenciais.
3. Habilitar Managed Prometheus, criar o namespace, completar e aplicar `prometheus-app.yaml` e `pod-monitoring.yaml`.
4. Baixar o `hello-app` e criar o `helloweb` com `<todo>` para gerar `InvalidImageName`.
5. Criar a metrica `pod-image-errors` e a policy `Pod Error Alert`.
6. Corrigir a imagem (`hello-app:1.0`), apagar e recriar o `helloweb`.
7. Alterar `main.go` para 2.0.0, build e push `hello-app:v2`, `set image` no `helloweb`.
8. Expor `SERVICE_NAME` (LoadBalancer :8080) e validar `Version: 2.0.0`.
