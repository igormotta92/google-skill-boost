# Collect Metrics from Exporters using the Managed Service for Prometheus

## Visao Geral

Neste lab, voce usa o Managed Service for Prometheus (GMP) para coletar metricas de outras fontes de infraestrutura por meio de exporters.

Objetivos:

- Fazer deploy de um cluster GKE.
- Configurar o custom resource `PodMonitoring` e a ferramenta `node-exporter`.
- Baixar o binario do GMP (Prometheus com exportacao para o Google Cloud) e executa-lo localmente.
- Aplicar uma configuracao do Prometheus para comecar a coletar metricas.

## Introducao

Este guia converte o lab (GSP1026) para execucao via CLI com `gcloud`, `kubectl` e `curl`, mantendo os nomes de recursos do enunciado.

O lab tem dois cenarios independentes:

- **Coleta gerenciada no GKE** (Tarefas 1 a 4): cluster `gmp-cluster` com `--enable-managed-prometheus`, app de exemplo no namespace `gmp-test` e `PodMonitoring` `prom-example`.
- **Coleta auto-implantada no Cloud Shell** (Tarefas 5 a 7): binario `prometheus` do GMP rodando localmente, `node_exporter` na porta `9100` e um `config.yaml` publicado em `gs://<PROJECT_ID>`.

Recursos usados no lab: cluster `gmp-cluster`, namespace `gmp-test`, PodMonitoring `prom-example`, bucket `gs://<PROJECT_ID>` com `config.yaml`.

---

## Pre-requisitos e Variaveis

### Conceito
Padronizar variaveis reduz erro de digitacao. Em Cloud Shell, refaca essa configuracao a cada nova sessao/reconexao e **em cada aba** nova (variaveis nao sao compartilhadas entre abas).

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export ZONE="<ZONA_DO_LAB>"   # ajuste conforme o painel do lab (o enunciado deixa a zona em branco)

# Recursos do lab
export CLUSTER_NAME="gmp-cluster"
export APP_NAMESPACE="gmp-test"
export GMP_VERSION="v0.2.3"
export GMP_EXAMPLES="https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/${GMP_VERSION}/examples"

# Binario Prometheus (GMP) e node_exporter
export PROM_REPO="https://github.com/GoogleCloudPlatform/prometheus"
export PROM_TAG="v2.28.1-gmp.4"
export PROM_BINARY_URL="https://storage.googleapis.com/kochasoft/gsp1026/prometheus"
export NODE_EXPORTER_VERSION="1.3.1"
export NODE_EXPORTER_DIR="node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64"

# Bucket do config.yaml (nome = project id)
export CONFIG_BUCKET="gs://${PROJECT_ID}"

gcloud config set project "$PROJECT_ID"
gcloud config set compute/zone "$ZONE"
```

### Explicacao dos parametros
- `ZONE`: usada no cluster e no label `location` das metricas exportadas pelo binario.
- `GMP_EXAMPLES`: base dos manifestos oficiais de exemplo (app e PodMonitoring).
- `CONFIG_BUCKET`: o lab cria o bucket com o mesmo nome do projeto.

---

## TAREFA 1: Fazer deploy do cluster GKE

### Conceito
O GKE com `--enable-managed-prometheus` instala os coletores gerenciados do GMP (namespace `gmp-system`). Eles fazem scrape dos alvos definidos em `PodMonitoring` e enviam as series para o Monarch (backend do Cloud Monitoring).

### Passo 1: Criar o cluster

```bash
gcloud beta container clusters create "$CLUSTER_NAME" \
  --num-nodes=1 \
  --zone "$ZONE" \
  --enable-managed-prometheus
```

### Passo 2: Obter credenciais

```bash
gcloud container clusters get-credentials "$CLUSTER_NAME" --zone="$ZONE"
kubectl get nodes
```

### Explicacao dos parametros
- `--num-nodes=1`: cluster minimo para o lab.
- `--enable-managed-prometheus`: habilita a coleta gerenciada do GMP.
- `get-credentials`: configura o kubeconfig para o `kubectl`.

### Resultado esperado
- Cluster `gmp-cluster` em `RUNNING` com 1 node `Ready`.
- Pods de coleta gerenciada no namespace `gmp-system`.

---

## TAREFA 2: Criar o namespace

### Conceito
O namespace `gmp-test` isola a aplicacao de exemplo e o `PodMonitoring` (que so faz scrape no proprio namespace).

### Passo 1: Criar o namespace

```bash
kubectl create ns "$APP_NAMESPACE"
kubectl get ns "$APP_NAMESPACE"
```

### Passo 2: Conferir os componentes do GMP

```bash
kubectl get pods -n gmp-system
```

### Resultado esperado
- Namespace `gmp-test` em `Active`.
- Coletores do GMP (`collector-*`, `gmp-operator-*`) rodando em `gmp-system`.

> Click **Check my progress**: *Check if prometheus has been deployed*.

---

## TAREFA 3: Fazer deploy da aplicacao de exemplo

### Conceito
O manifesto oficial cria uma aplicacao com **3 replicas** que expoe metricas Prometheus na porta nomeada `metrics`, com o label `app: prom-example`.

### Passo 1: Aplicar o manifesto

```bash
kubectl -n "$APP_NAMESPACE" apply -f "${GMP_EXAMPLES}/example-app.yaml"
```

### Passo 2: Validar

```bash
kubectl -n "$APP_NAMESPACE" get pods -l app=prom-example
kubectl -n "$APP_NAMESPACE" rollout status deployment/prom-example
```

### Resultado esperado
- 3 Pods `prom-example-*` em `Running`.

---

## TAREFA 4: Configurar um recurso PodMonitoring

### Conceito
O **PodMonitoring** e o custom resource do GMP que define o target scraping: seleciona Pods por label e faz scrape de uma porta/caminho em um intervalo. Ele atua **apenas no namespace onde foi criado**; para varios namespaces, repita o CR em cada um ou use **ClusterPodMonitoring**.

Manifesto aplicado pelo lab (`pod-monitoring.yaml`):

```yaml
apiVersion: monitoring.googleapis.com/v1alpha1
kind: PodMonitoring
metadata:
  name: prom-example
spec:
  selector:
    matchLabels:
      app: prom-example
  endpoints:
  - port: metrics
    interval: 30s
```

### Passo 1: Aplicar o PodMonitoring

```bash
kubectl -n "$APP_NAMESPACE" apply -f "${GMP_EXAMPLES}/pod-monitoring.yaml"
```

### Passo 2: Validar

```bash
kubectl get podmonitoring -A
kubectl -n "$APP_NAMESPACE" describe podmonitoring prom-example
```

### Explicacao dos parametros
- `selector.matchLabels.app: prom-example`: escolhe os Pods da app de exemplo.
- `endpoints.port: metrics`: porta nomeada no container; o caminho padrao e `/metrics`.
- `interval: 30s`: frequencia do scrape.
- `targetLabels` (opcional): relabel simplificado para levar labels do Pod para as series; nao pode sobrescrever os labels obrigatorios do recurso `prometheus_target`.

### Resultado esperado
- `prom-example` listado em `gmp-test`.
- O coletor gerenciado passa a fazer scrape dos 3 Pods. Para consultar: **Monitoring > Managed Prometheus** (ou Metrics Explorer com PromQL).

### Resumo visual

```text
[GKE cluster gmp-cluster] (ZONE, 1 node, --enable-managed-prometheus)
  |
  +-- namespace gmp-test
  |     [Deployment prom-example] 3 Pods (app: prom-example, porta metrics)
  |              ^
  |              | scrape /metrics a cada 30s
  |     [PodMonitoring prom-example]
  |
  +-- namespace gmp-system
        [GMP collectors] --envia series--> [Managed Service for Prometheus]
                                           (Monarch / Cloud Monitoring)
```

---

## TAREFA 5: Baixar o binario do Prometheus (GMP)

### Conceito
Alem da coleta gerenciada, o GMP oferece um **binario Prometheus modificado** que roda fora do cluster (aqui, no Cloud Shell) e exporta as series para o mesmo backend gerenciado.

### Passo 1: Clonar o repositorio e baixar o binario

```bash
cd "$HOME"
git clone "$PROM_REPO" && cd prometheus
git checkout "$PROM_TAG"

wget "$PROM_BINARY_URL"
chmod a+x prometheus
./prometheus --version
```

### Explicacao dos parametros
- `git checkout v2.28.1-gmp.4`: versao do fork do GMP; traz o `documentation/examples/prometheus.yml` usado na Tarefa 6.
- `chmod a+x`: torna o binario executavel.

### Resultado esperado
- Binario `./prometheus` executavel dentro de `~/prometheus`.

---

## TAREFA 6: Executar o binario do Prometheus

### Conceito
O binario precisa dos labels `project-id` e `location` para gravar as series no projeto correto. Com a config de exemplo, o Prometheus faz scrape de si mesmo (`localhost:9090`).

### Passo 1: Rodar o binario (aba 1 do Cloud Shell)

```bash
cd "$HOME/prometheus"

./prometheus \
  --config.file=documentation/examples/prometheus.yml \
  --export.label.project-id="$PROJECT_ID" \
  --export.label.location="$ZONE"
```

O processo fica em primeiro plano nessa aba.

### Passo 2: Validar no Managed Prometheus

Console (passo do lab): **Monitoring > Managed Prometheus** > query PromQL `up` > deve aparecer a instancia `localhost` rodando.

Equivalente CLI (em outra aba, com as variaveis redefinidas):

```bash
curl -s "http://localhost:9090/api/v1/query?query=up" | jq '.data.result[] | {instance: .metric.instance, value: .value[1]}'
```

### Explicacao dos parametros
- `--config.file`: arquivo de scrape config (aqui, o exemplo do repositorio).
- `--export.label.project-id`: projeto de destino das series no GMP.
- `--export.label.location`: label de localizacao obrigatorio das series.

### Resultado esperado
- Query `up` retornando `1` para `localhost:9090`.

### Resumo visual

```text
[Cloud Shell - aba 1]
  [./prometheus :9090] (config: documentation/examples/prometheus.yml)
        |    ^
        |    | scrape de si mesmo (job prometheus)
        |    +
        |
        | export (project-id=PROJECT_ID, location=ZONE)
        v
[Managed Service for Prometheus] --> PromQL: up -> instance localhost
```

---

## TAREFA 7: Baixar e executar o node exporter

### Conceito
O **node_exporter** expoe metricas do host (CPU, memoria, disco, rede) no formato Prometheus na porta `9100`. Um novo `config.yaml` faz o binario do Prometheus coletar dessa porta.

### Passo 1: Rodar o node_exporter (aba 2 do Cloud Shell)

Abra uma nova aba no Cloud Shell (**+**) e redefina as variaveis do bloco "Pre-requisitos e Variaveis". Depois:

```bash
cd "$HOME"
wget "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/${NODE_EXPORTER_DIR}.tar.gz"
tar xvfz "${NODE_EXPORTER_DIR}.tar.gz"
cd "$NODE_EXPORTER_DIR"
./node_exporter
```

**Resultado esperado:** log `msg="Listening on" address=:9100`.

Teste rapido (em uma terceira aba ou apos parar o processo):

```bash
curl -s http://localhost:9100/metrics | grep '^node_cpu_seconds_total' | head -n 3
```

### Passo 2: Criar o `config.yaml` (aba 1)

Na aba 1, pare o Prometheus com `Ctrl+C` e crie o arquivo no diretorio `~/prometheus`.

Comando do lab: `vi config.yaml` e colar o conteudo abaixo. Equivalente com `cat`:

```bash
cd "$HOME/prometheus"

cat > config.yaml <<EOF
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ['localhost:9100']
EOF

cat config.yaml
```

### Passo 3: Publicar o `config.yaml` no bucket

```bash
gcloud storage buckets create --project="$PROJECT_ID" "$CONFIG_BUCKET"
gcloud storage cp config.yaml "$CONFIG_BUCKET"
gcloud storage buckets add-iam-policy-binding "$CONFIG_BUCKET" \
  --member=allUsers \
  --role=roles/storage.objectViewer
```

Validar:

```bash
gcloud storage ls "${CONFIG_BUCKET}/config.yaml"
curl -s "https://storage.googleapis.com/${PROJECT_ID}/config.yaml"
```

> Click **Check my progress**: *Check if config.yaml is configured correctly*.

### Passo 4: Reexecutar o Prometheus com o novo config (aba 1)

```bash
./prometheus \
  --config.file=config.yaml \
  --export.label.project-id="$PROJECT_ID" \
  --export.label.location="$ZONE"
```

### Passo 5: Visualizar as metricas do node_exporter

Console (passo do lab): no Cloud Shell, **Web preview > Change port** `9090` > **Change and Preview** > no editor PromQL digite `node_` e selecione `node_cpu_seconds_total` para ver o grafico.

Equivalente CLI (em outra aba):

```bash
# Alvo do job node ativo
curl -s "http://localhost:9090/api/v1/targets" | jq '.data.activeTargets[] | {job: .labels.job, url: .scrapeUrl, health: .health}'

# Series de node_cpu_seconds_total
curl -s "http://localhost:9090/api/v1/query?query=node_cpu_seconds_total" | jq '.data.result | length'
```

### Explicacao dos parametros
- `scrape_interval: 15s`: intervalo global de scrape.
- `job_name: node` + `targets: ['localhost:9100']`: coleta do node_exporter local.
- `buckets create` / `cp`: publicam o `config.yaml` no bucket com o nome do projeto.
- `allUsers` + `roles/storage.objectViewer`: torna o objeto publico para leitura (exigido pelo check do lab; evite em producao).

### Resultado esperado
- Target `node` com `health: up` apontando para `localhost:9100`.
- Metricas `node_*` disponiveis no PromQL (ex.: `node_cpu_seconds_total` com grafico).

### Resumo visual

```text
[Cloud Shell - aba 2]                 [Cloud Shell - aba 1]
[node_exporter :9100] <--scrape 15s-- [./prometheus :9090]
                         job: node     (config.yaml)
                                            |
                                            | export (project-id, location)
                                            v
                              [Managed Service for Prometheus]

[config.yaml] --gcloud storage cp--> [gs://PROJECT_ID/config.yaml]
                                      (allUsers: objectViewer)
```

---

## Validacao

```bash
# Cluster e coleta gerenciada
gcloud container clusters describe "$CLUSTER_NAME" --zone "$ZONE" \
  --format="value(status,monitoringConfig.managedPrometheusConfig.enabled)"
kubectl get pods -n gmp-system
kubectl get podmonitoring -A
kubectl -n "$APP_NAMESPACE" get pods -l app=prom-example

# Bucket e config publico
gcloud storage ls "${CONFIG_BUCKET}/config.yaml"
gcloud storage buckets get-iam-policy "$CONFIG_BUCKET" --format="yaml(bindings)"

# Prometheus local + node_exporter (com os processos rodando)
curl -s "http://localhost:9090/api/v1/query?query=up" | jq '.data.result[].metric.job'
```

Checklist esperado:
- `gmp-cluster` em `RUNNING` com managed Prometheus habilitado;
- namespace `gmp-test` com 3 Pods `prom-example` e o PodMonitoring `prom-example`;
- `gs://<PROJECT_ID>/config.yaml` publico para leitura;
- job `node` com `up = 1` e metricas `node_*` consultaveis.

---

## Resumo Visual da Arquitetura

Compilado: os dois cenarios do lab e o estado final. Os detalhes de cada etapa estao no `### Resumo visual` das Tarefas 4, 6 e 7.

```text
Cenario 1 - coleta gerenciada (GKE)
[gmp-cluster / gmp-test]
  [prom-example x3] <--scrape-- [PodMonitoring] / [GMP collectors]
                                        |
                                        v
                       [Managed Service for Prometheus] <---+
                                                            |
Cenario 2 - coleta auto-implantada (Cloud Shell)            |
  [node_exporter :9100] <--scrape-- [./prometheus :9090] ---+
                                     (config.yaml)
  [gs://PROJECT_ID/config.yaml]  (copia publica do config)
```

---

## Troubleshooting

- `--zone` vazio / erro de zona no `clusters create`:
  - o enunciado deixa a zona em branco; preencha `ZONE` com o valor do painel do lab.
- Pods do GMP ausentes em `gmp-system`:
  - confirme `--enable-managed-prometheus` no cluster (`monitoringConfig.managedPrometheusConfig.enabled`); aguarde alguns minutos apos a criacao.
- `kubectl get podmonitoring` nao encontra o recurso:
  - aplique o manifesto com `-n gmp-test`; o PodMonitoring so atua no namespace em que foi criado.
- Binario `./prometheus` com `Permission denied`:
  - rode `chmod a+x prometheus` dentro de `~/prometheus`.
- Erro de permissao/projeto ao exportar metricas:
  - confirme `--export.label.project-id="$PROJECT_ID"` e que a variavel nao esta vazia na aba atual.
- `address already in use` em `:9090` ou `:9100`:
  - ja existe um processo rodando; pare com `Ctrl+C` na aba correspondente (ou `pkill prometheus` / `pkill node_exporter`).
- Variaveis vazias na segunda aba:
  - variaveis nao sao compartilhadas entre abas; rode novamente o bloco de variaveis.
- `add-iam-policy-binding` com `allUsers` negado:
  - pode haver politica de public access prevention; confira a mensagem de erro e o bucket (`gcloud storage buckets describe "$CONFIG_BUCKET"`).
- Web preview nao abre o Prometheus:
  - confirme que o binario esta rodando na aba 1 e que a porta de preview e `9090`.

---

## Limpeza (Opcional)

```bash
# Parar processos locais (Ctrl+C nas abas, ou)
pkill prometheus; pkill node_exporter

# Remover app e PodMonitoring
kubectl -n "$APP_NAMESPACE" delete -f "${GMP_EXAMPLES}/pod-monitoring.yaml"
kubectl -n "$APP_NAMESPACE" delete -f "${GMP_EXAMPLES}/example-app.yaml"
kubectl delete ns "$APP_NAMESPACE"

# Remover bucket e cluster
gcloud storage rm -r "$CONFIG_BUCKET"
gcloud container clusters delete "$CLUSTER_NAME" --zone "$ZONE" --quiet
```

---

## Conceitos-Chave

- Managed Service for Prometheus (GMP): Prometheus gerenciado pelo Google, com armazenamento no Monarch e consulta via PromQL.
- Coleta gerenciada: coletores do GMP no GKE configurados por `PodMonitoring` / `ClusterPodMonitoring`.
- PodMonitoring: seleciona Pods por label e define porta, caminho e intervalo de scrape; escopo de namespace.
- ClusterPodMonitoring: mesma interface, mas descobre Pods em todos os namespaces.
- Coleta auto-implantada: binario Prometheus do GMP rodando fora do cluster e exportando para o mesmo backend (`--export.label.project-id` / `location`).
- Exporters: processos que expoem metricas de sistemas terceiros no formato Prometheus (ex.: `node_exporter` na porta 9100).

---

## Fluxo Final

1. Definir variaveis (incluindo a `ZONE` do painel do lab).
2. Criar `gmp-cluster` com `--enable-managed-prometheus` e obter credenciais.
3. Criar o namespace `gmp-test` e conferir os coletores em `gmp-system`.
4. Aplicar a app de exemplo (3 replicas) e o PodMonitoring `prom-example`.
5. Clonar o repo do Prometheus GMP, fazer checkout `v2.28.1-gmp.4` e baixar o binario.
6. Rodar `./prometheus` com a config de exemplo e validar `up`.
7. Rodar o `node_exporter` (aba 2), criar o `config.yaml`, publicar em `gs://PROJECT_ID` com leitura publica.
8. Reexecutar o Prometheus com `config.yaml` e consultar `node_cpu_seconds_total`.
