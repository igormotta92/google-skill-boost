# Debug Apps on Google Kubernetes Engine

## Visao Geral

Cloud Logging e sua ferramenta companheira, Cloud Monitoring, sao produtos completos e profundamente integrados ao Google Kubernetes Engine (GKE). Este lab mostra como o Cloud Logging funciona com clusters e aplicacoes no GKE, alem de boas praticas de coleta de logs em casos de uso comuns.

Objetivos:

- Usar o Cloud Monitoring para detectar problemas.
- Usar o Cloud Logging para fazer troubleshooting de uma aplicacao rodando no GKE.

A aplicacao de exemplo e um demo de microservicos (Hipster Shop / Online Boutique) com varias dependencias entre servicos. Voce gera trafego com um load generator e usa Logging, Monitoring e GKE para perceber o erro (alerta/metrica), identificar a causa raiz nos logs e confirmar a correcao.

## Introducao

Este guia converte o lab (GSP736) para execucao via CLI com `gcloud`, `kubectl` e `curl`, mantendo os nomes de recursos do enunciado. Os passos que o lab faz no Console (Logs Explorer, Alerting, Locust) aparecem como "Console (passo do lab)" com o equivalente CLI ao lado.

Objetivo do lab:
- conectar no cluster `central` (pre-criado pelo lab);
- fazer deploy do Hipster Shop a partir de `release/kubernetes-manifests.yaml`;
- criar a metrica baseada em logs `Error_Rate_SLI`;
- criar a alerting policy `Error Rate SLI` (threshold `0.5`, sem canal de notificacao);
- gerar carga com Locust, confirmar o incidente e investigar os logs;
- encontrar a causa raiz (`ENABLE_RELOAD` no `productcatalogservice`) e corrigir.

---

## Pre-requisitos e Variaveis

### Conceito
Padronizar variaveis reduz erro de digitacao. Em Cloud Shell, refaca essa configuracao a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID

# Cluster pre-criado pelo lab
export CLUSTER_NAME="central"

# Aplicacao
export APP_REPO="https://github.com/xiangshen-dk/microservices-demo.git"
export APP_DIR="$HOME/microservices-demo"
export MANIFEST="release/kubernetes-manifests.yaml"

# Observabilidade
export METRIC_NAME="Error_Rate_SLI"
export ALERT_NAME="Error Rate SLI"

gcloud config set project "$PROJECT_ID"
```

### Passo 2: Descobrir a zona do cluster

O enunciado usa `ZONE` sem informar o valor. Capture direto do cluster:

```bash
export ZONE="$(gcloud container clusters list \
  --filter="name=${CLUSTER_NAME}" \
  --format='value(location)')"
echo "$ZONE"
```

Se vier vazio, o cluster ainda nao foi criado; aguarde e rode de novo (ou ajuste conforme o painel do lab).

---

## TAREFA 1: Preparar a infraestrutura

### Conceito
O cluster `central` e provisionado pelo lab. Antes de usar o `kubectl`, e preciso esperar o status `RUNNING` e baixar as credenciais para o kubeconfig.

### Passo 1: Aguardar o cluster ficar RUNNING

```bash
gcloud container clusters list

until [ "$(gcloud container clusters describe "$CLUSTER_NAME" --zone "$ZONE" --format='value(status)')" = "RUNNING" ]; do
  echo "Aguardando cluster ${CLUSTER_NAME}..."; sleep 15
done
```

### Passo 2: Obter credenciais

```bash
gcloud container clusters get-credentials "$CLUSTER_NAME" --zone "$ZONE"
```

### Passo 3: Validar os nodes

```bash
kubectl get nodes
```

### Explicacao dos parametros
- `clusters list` / `describe --format='value(status)'`: mostram o status do cluster (`PROVISIONING` -> `RUNNING`).
- `get-credentials`: gera a entrada do kubeconfig para o `kubectl` falar com o cluster.

### Resultado esperado
- `kubeconfig entry generated for central.`
- Nodes `gke-central-default-pool-...` em `Ready` (4 no exemplo do lab).

---

## TAREFA 2: Fazer deploy da aplicacao

### Conceito
O Hipster Shop e um conjunto de microservicos (frontend, cart, checkout, productcatalog, recommendation, etc.) com um `loadgenerator` (Locust). Dois Services do tipo LoadBalancer expoem a aplicacao (`frontend-external`) e o Locust (`loadgenerator-external`).

### Passo 1: Clonar o repositorio e aplicar o manifesto

```bash
cd "$HOME"
git clone "$APP_REPO"
cd "$APP_DIR"

kubectl apply -f "$MANIFEST"
```

### Passo 2: Aguardar todos os Pods em Running

```bash
kubectl get pods

kubectl wait --for=condition=Ready pods --all --timeout=600s
```

> Click **Check my progress**: *Deploy an application*.

### Passo 3: Capturar o IP externo do frontend

Comando do lab:

```bash
export EXTERNAL_IP=$(kubectl get service frontend-external | awk 'BEGIN { cnt=0; } { cnt+=1; if (cnt > 1) print $4; }')
```

Alternativa que espera o IP ser provisionado:

```bash
until EXTERNAL_IP="$(kubectl get svc frontend-external -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')" && [ -n "$EXTERNAL_IP" ]; do
  echo "Aguardando EXTERNAL-IP do frontend..."; sleep 5
done
export EXTERNAL_IP
echo "$EXTERNAL_IP"
```

### Passo 4: Testar a aplicacao

```bash
curl -o /dev/null -s -w "%{http_code}\n" "http://${EXTERNAL_IP}"
```

### Explicacao dos parametros
- `kubectl apply -f`: cria/atualiza todos os Deployments e Services do manifesto.
- `kubectl wait --for=condition=Ready pods --all`: bloqueia ate todos os Pods ficarem prontos.
- `-o=jsonpath='{.status.loadBalancer.ingress[0].ip}'`: extrai o IP do load balancer.
- `curl -w "%{http_code}"`: imprime apenas o status HTTP.

### Resultado esperado
- 12 Pods em `Running` (adservice, cartservice, checkoutservice, currencyservice, emailservice, frontend, loadgenerator, paymentservice, productcatalogservice, recommendationservice, redis-cart, shippingservice).
- `curl` retornando `200`.
- Console: **Kubernetes Engine > Workloads** com todos os Pods OK e **Gateways, Services & Ingress > Services** com todos os Services OK.

### Resumo visual

```text
[Cliente / browser]                [Locust UI]
        |                               |
        v                               v
[Service frontend-external]   [Service loadgenerator-external]
 (LoadBalancer EXTERNAL_IP)    (LoadBalancer LOADGEN_IP)
        |                               |
        v                               | gera carga
[frontend] <----------------------------+
   |            |               |
   v            v               v
[recommendation] [productcatalog] [cart, checkout, ads, ...]
   |                 ^
   +--- chama -------+
        (cluster central / namespace default)
```

---

## TAREFA 3: Abrir a aplicacao

### Conceito
Validar no navegador que o frontend (Online Boutique) esta respondendo.

### Passo 1: Obter a URL

```bash
echo "http://${EXTERNAL_IP}"
```

Console (passo do lab): **Kubernetes Engine > Gateways, Services & Ingress > Services** > `frontend-external` > clique no IP em **Endpoints**.

### Resultado esperado
- Pagina do Online Boutique com os tiles de produtos.

---

## TAREFA 4: Criar uma metrica baseada em logs

### Conceito
Uma **logs-based metric** e uma metrica customizada no Cloud Monitoring gerada a partir de entradas de log. Aqui ela conta os erros (`severity=ERROR`) do `recommendationservice`, e sera usada no alerta.

### Passo 1: Conferir a query no Logs Explorer (opcional)

Console (passo do lab): **Logging > Logs Explorer** > habilite **Show query** e use:

```text
resource.type="k8s_container"
severity=ERROR
labels."k8s-pod/app": "recommendationservice"
```

Equivalente CLI (sem erros ainda, o resultado deve vir vazio):

```bash
export ERROR_FILTER='resource.type="k8s_container"
severity=ERROR
labels."k8s-pod/app": "recommendationservice"'

gcloud logging read "$ERROR_FILTER" --limit=5 --freshness=1h
```

### Passo 2: Criar a metrica `Error_Rate_SLI`

Console (passo do lab): **Actions > Create Metric** > nome `Error_Rate_SLI` > **Create Metric**.

Equivalente CLI:

```bash
gcloud logging metrics create "$METRIC_NAME" \
  --description="Erros do recommendationservice (SLI)" \
  --log-filter="$ERROR_FILTER"
```

### Passo 3: Validar

```bash
gcloud logging metrics describe "$METRIC_NAME"
```

### Explicacao dos parametros
- `--log-filter`: mesma query do Logs Explorer; cada entrada que casa incrementa o contador.
- Metrica do tipo counter: aparece no Monitoring como `logging/user/Error_Rate_SLI`.

### Resultado esperado
- Metrica listada em **Logs-based Metrics > User-defined Metrics**.

> Click **Check my progress**: *Create a logs-based metric*.

---

## TAREFA 5: Criar uma alerting policy

### Conceito
A alerting policy observa a metrica `logging/user/Error_Rate_SLI` (recurso **Kubernetes Container**) e abre um incidente quando a taxa (`Rate`) passa de `0.5`. Sem canal de notificacao neste lab; em producao, configure email, SMS, Pub/Sub, webhook, etc.

### Passo 1: Criar a policy

Console (passo do lab): **Monitoring > Alerting > Create Policy** > **Select a metric** (desmarque **Active**) > filtro `Error_Rate` > **Kubernetes Container > Logs-Based Metric > logging/user/Error_Rate_SLI** > **Apply** > Rolling window function **Rate** > **Next** > Threshold `0.5` > **Next** > desabilite **Use notification channel** > nome `Error Rate SLI` > **Create Policy**.

Equivalente CLI (arquivo de policy gerado com `cat` e variaveis):

```bash
cat > error-rate-sli-policy.json <<EOF
{
  "displayName": "${ALERT_NAME}",
  "combiner": "OR",
  "conditions": [
    {
      "displayName": "Kubernetes Container - logging/user/${METRIC_NAME}",
      "conditionThreshold": {
        "filter": "resource.type = \"k8s_container\" AND metric.type = \"logging.googleapis.com/user/${METRIC_NAME}\"",
        "aggregations": [
          {
            "alignmentPeriod": "300s",
            "perSeriesAligner": "ALIGN_RATE"
          }
        ],
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0.5,
        "duration": "0s",
        "trigger": { "count": 1 }
      }
    }
  ],
  "enabled": true
}
EOF

gcloud alpha monitoring policies create --policy-from-file=error-rate-sli-policy.json
```

### Passo 2: Validar

```bash
gcloud alpha monitoring policies list \
  --filter="displayName=\"${ALERT_NAME}\"" \
  --format="table(displayName,enabled,conditions[0].conditionThreshold.thresholdValue)"
```

### Explicacao dos parametros
- `metric.type = logging.googleapis.com/user/Error_Rate_SLI`: metrica criada na Tarefa 4.
- `perSeriesAligner: ALIGN_RATE`: equivale a "Rolling window function = Rate".
- `alignmentPeriod: 300s`: janela de 5 minutos.
- `COMPARISON_GT` + `thresholdValue: 0.5`: dispara quando a taxa de erros passa de 0.5.
- Sem `notificationChannels`: equivale a desabilitar **Use notification channel**.

### Resultado esperado
- Policy `Error Rate SLI` listada e habilitada; nenhum incidente enquanto nao ha erros.

> Click **Check my progress**: *Create an alerting policy*.

### Passo 3: Disparar o erro com o Locust

O build da aplicacao tem um bug proposital que aparece sob carga.

Capturar o IP do Locust:

```bash
until LOADGEN_IP="$(kubectl get svc loadgenerator-external -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')" && [ -n "$LOADGEN_IP" ]; do
  echo "Aguardando IP do loadgenerator..."; sleep 5
done
export LOADGEN_IP
echo "Locust: http://${LOADGEN_IP}  |  Host: http://${EXTERNAL_IP}"
```

Console/UI (passo do lab): abra `http://LOADGEN_IP` > **Number of users** `300` > **Hatch/Spawn rate** `30` > **Host** `http://EXTERNAL_IP` (sem porta) > **Start swarming**.

Equivalente CLI via API do Locust (envia os nomes de campo das versoes antiga e nova; se nao iniciar, use a UI):

```bash
curl -s -X POST "http://${LOADGEN_IP}/swarm" \
  -d "user_count=300" -d "spawn_rate=30" \
  -d "locust_count=300" -d "hatch_rate=30" \
  -d "host=http://${EXTERNAL_IP}"
```

Acompanhar a taxa de falhas:

```bash
curl -s "http://${LOADGEN_IP}/stats/requests" | jq '{users: .user_count, fail_ratio: .fail_ratio}'
```

**Resultado esperado:** aba **Failures** do Locust com muitos erros `500`; produtos do site lentos ou com `500 Internal Server Error`.

### Passo 4: Confirmar o alerta e os erros

Console (passo do lab): **Monitoring > Alerting** > incidente de `logging/user/Error_Rate_SLI` (pode levar ate 5 minutos) > abra o incidente > **Logs Explorer** > filtre **Severity = Error** > expanda um erro > `textPayload` > **Add field to summary line**.

Equivalente CLI para ver os erros do `recommendationservice`:

```bash
gcloud logging read "$ERROR_FILTER" --limit=10 --freshness=15m \
  --format="table(timestamp,textPayload)"
```

**Resultado esperado:** muitos erros de conexao do `RecommendationService` com servicos downstream (ex.: `Connect Failed`). Como frontend e recommendation chamam o `ProductCatalogService`, ele e o principal suspeito.

### Passo 5: Investigar o productcatalogservice

Console (passo do lab): **Monitoring > Dashboards > GKE** (secao Workloads) e **Kubernetes Engine > Workloads > productcatalogservice** > abas **Logs** / **Container logs**.

Equivalente CLI:

```bash
# Pod reiniciando (coluna RESTARTS crescendo)
kubectl get pods -l app=productcatalogservice

# Eventos e motivo das reinicializacoes
kubectl describe deployment productcatalogservice | tail -n 20

# Logs do container atual e do que crashou
kubectl logs deploy/productcatalogservice --tail=20
kubectl logs deploy/productcatalogservice --previous --tail=20
```

**Resultado esperado:** muitas mensagens `successfully parsed product catalog json` em pouco tempo e, possivelmente, `panic: runtime error: invalid memory address or nil pointer dereference` (`SIGSEGV`).

### Passo 6: Encontrar a causa no codigo

```bash
cd "$APP_DIR"

grep -nri 'successfully parsed product catalog json' src
grep -n 'reloadCatalog\|ENABLE_RELOAD' src/productcatalogservice/server.go
```

Console (passo do lab): **Open Editor** > `microservices-demo/src/productcatalogservice/server.go`, linha indicada pelo `grep` (ex.: 237, metodo `readCatalogFile`).

Analise:
- se `reloadCatalog` for `true`, o servico rele e parseia o catalogo **a cada requisicao**;
- `reloadCatalog` e controlado pela variavel de ambiente `ENABLE_RELOAD`, que loga o estado na inicializacao.

### Passo 7: Confirmar nos logs que o reload esta ligado

Console (passo do lab): no Logs Explorer, adicione `jsonPayload.message:"catalog reloading"` a query do container.

Equivalente CLI:

```bash
gcloud logging read "resource.type=\"k8s_container\"
resource.labels.location=\"${ZONE}\"
resource.labels.cluster_name=\"${CLUSTER_NAME}\"
resource.labels.namespace_name=\"default\"
labels.\"k8s-pod/app\"=\"productcatalogservice\"
jsonPayload.message:\"catalog reloading\"" \
  --limit=5 --freshness=1h \
  --format="table(timestamp,jsonPayload.message)"
```

**Resultado esperado:** mensagem `Enable catalog reloading`. Conclusao: o overhead de recarregar o catalogo em toda requisicao derruba o servico sob carga e gera os `500` no frontend.

### Resumo visual

```text
[Locust 300 users] --carga--> [frontend] --> [recommendationservice]
                                  |                    |
                                  v                    v
                         [productcatalogservice] <-----+
                          ENABLE_RELOAD=1
                          reload do catalogo a cada request
                          -> CrashLoop / panic SIGSEGV
                                  |
                                  v
[Cloud Logging] severity=ERROR (recommendationservice)
        |
        v
[Logs-based metric Error_Rate_SLI] --> [Alert policy Error Rate SLI]
                                        (Rate > 0.5) --> [Incidente]
```

---

## TAREFA 6: Corrigir o problema e verificar o resultado

### Conceito
A correcao e remover a variavel `ENABLE_RELOAD` do `productcatalogservice` no manifesto e reaplicar. So o Deployment alterado e reconfigurado; os outros ficam `unchanged`.

### Passo 1: Localizar a variavel no manifesto

```bash
cd "$APP_DIR"
grep -A1 -ni ENABLE_RELOAD "$MANIFEST"
```

### Passo 2: Remover as duas linhas

Comando do lab (numeros de linha fixos, confira com o `grep` antes):

```bash
sed -i -e '373,374d' "$MANIFEST"
```

Alternativa que captura a linha em variavel (nao depende do numero fixo):

```bash
export RELOAD_LINE="$(grep -n 'name: ENABLE_RELOAD' "$MANIFEST" | cut -d: -f1)"
echo "$RELOAD_LINE"
sed -i -e "${RELOAD_LINE},$((RELOAD_LINE + 1))d" "$MANIFEST"

# Confirmar que sumiu (sem saida)
grep -ni ENABLE_RELOAD "$MANIFEST"
```

Use **apenas uma** das duas opcoes; rodar as duas apaga linhas a mais.

### Passo 3: Reaplicar o manifesto

```bash
kubectl apply -f "$MANIFEST"
kubectl rollout status deployment/productcatalogservice
```

### Passo 4: Verificar a correcao

```bash
# Pod estavel (RESTARTS parado) - aguarde 2-3 minutos
kubectl get pods -l app=productcatalogservice

# Sem mensagens repetidas de parse do catalogo
kubectl logs deploy/productcatalogservice --tail=20

# Frontend respondendo
curl -o /dev/null -s -w "%{http_code}\n" "http://${EXTERNAL_IP}"

# Locust: resetar estatisticas e acompanhar a taxa de falhas
curl -s "http://${LOADGEN_IP}/stats/reset"
sleep 60
curl -s "http://${LOADGEN_IP}/stats/requests" | jq '{users: .user_count, fail_ratio: .fail_ratio}'
```

Console (passo do lab): **Workloads > productcatalogservice** sem crashes; **Container logs** sem `successfully parsed product catalog json` repetido; Locust > **Reset Stats** com failures em `0%`.

### Explicacao dos parametros
- `sed -i -e 'N,Md'`: apaga as linhas N a M do arquivo.
- `kubectl apply`: reaplica o manifesto; so `productcatalogservice` aparece como `configured`.
- `rollout status`: espera o novo Pod ficar pronto.
- `/stats/reset`: equivale ao botao **Reset Stats** do Locust.

### Resultado esperado
- `productcatalogservice` estavel, sem restarts.
- Site responsivo, sem `500`.
- `fail_ratio` em `0` (ou proximo) apos o reset. Se ainda houver `500`, aguarde mais alguns minutos.

### Resumo visual

```text
Antes:  [productcatalogservice] ENABLE_RELOAD=1
          -> reload por request -> crash -> 500 no frontend

        sed remove ENABLE_RELOAD  +  kubectl apply

Depois: [productcatalogservice] catalogo carregado 1x
          -> Pod estavel -> frontend 200 -> Locust 0% failures
```

---

## Validacao

```bash
# Cluster e Pods
gcloud container clusters list --format="table(name,location,status)"
kubectl get pods

# Variavel removida do Deployment em execucao
kubectl get deployment productcatalogservice \
  -o=jsonpath='{.spec.template.spec.containers[0].env[*].name}'; echo

# Metrica e alerta
gcloud logging metrics describe "$METRIC_NAME" --format="value(name,filter)"
gcloud alpha monitoring policies list --filter="displayName=\"${ALERT_NAME}\"" --format="value(displayName,enabled)"

# Aplicacao
curl -o /dev/null -s -w "%{http_code}\n" "http://${EXTERNAL_IP}"
```

Checklist esperado:
- cluster `central` em `RUNNING` e todos os Pods em `Running`;
- `ENABLE_RELOAD` ausente das variaveis do `productcatalogservice`;
- metrica `Error_Rate_SLI` e policy `Error Rate SLI` criadas;
- frontend retornando `200`.

---

## Resumo Visual da Arquitetura

Compilado: topologia geral e estado final ao termino do lab. Os detalhes do erro e da correcao estao no `### Resumo visual` das Tarefas 2, 5 e 6.

```text
[Cloud Shell] --kubectl--> [GKE cluster central] (ZONE, namespace default)

[Cliente] --> [frontend-external LB] --> [frontend] --> [demais servicos]
[Locust]  --> [loadgenerator-external LB]   |
                                            v
                              [productcatalogservice]
                               (sem ENABLE_RELOAD, estavel)

[Pods] --logs--> [Cloud Logging]
                      |
                      v
           [Metric Error_Rate_SLI] --> [Alert Error Rate SLI]
                                       (Rate > 0.5, sem notificacao)
```

---

## Troubleshooting

- `ZONE` vazio no Passo 2 das variaveis:
  - cluster ainda provisionando; aguarde e rode de novo o `clusters list`.
- Pods presos em `Pending` / `ContainerCreating`:
  - aguarde; rode `kubectl get pods` ate todos ficarem `Running` antes do check.
- `EXTERNAL_IP` ou `LOADGEN_IP` vazio:
  - load balancer ainda sendo criado; use os loops `until` ou rode novamente apos reconectar o Cloud Shell.
- `gcloud alpha monitoring` pede para instalar componente / nao reconhecido:
  - confirme a instalacao do componente alpha ou crie a policy pelo Console (passo do lab).
- Policy criada mas sem incidente:
  - o alerta pode levar ate 5 minutos; confirme que o Locust esta gerando carga e que ha erros com `gcloud logging read "$ERROR_FILTER"`.
- `curl` na API do Locust nao inicia o swarm:
  - versoes diferentes do Locust usam campos diferentes; use a UI em `http://LOADGEN_IP`.
- `sed` apagou linhas erradas:
  - restaure com `git checkout -- release/kubernetes-manifests.yaml` e repita usando a alternativa com `RELOAD_LINE`.
- Ainda ha `500` apos a correcao:
  - aguarde mais 2-3 minutos; confirme `kubectl rollout status` e que o Pod parou de reiniciar.

---

## Limpeza (Opcional)

Nao remova o cluster `central`: ele e pre-criado pelo lab.

```bash
# Parar a carga do Locust
curl -s "http://${LOADGEN_IP}/stop"

# Remover a aplicacao
kubectl delete -f "$APP_DIR/$MANIFEST"

# Remover alerta e metrica
export POLICY_ID="$(gcloud alpha monitoring policies list --filter="displayName=\"${ALERT_NAME}\"" --format='value(name)')"
gcloud alpha monitoring policies delete "$POLICY_ID" --quiet
gcloud logging metrics delete "$METRIC_NAME" --quiet
```

---

## Conceitos-Chave

- Cloud Logging no GKE: logs de containers coletados automaticamente com labels de cluster, namespace e Pod.
- Logs-based metric: transforma entradas de log (ex.: erros) em metrica do Cloud Monitoring.
- Alerting policy: condicao sobre uma metrica (aqui, `Rate > 0.5`) que abre incidentes; em producao, associar canais de notificacao.
- SLI/SLO: taxa de erros como indicador de disponibilidade do servico.
- Troubleshooting em microservicos: partir do sintoma (alerta no frontend/recommendation), seguir as dependencias ate o servico suspeito e confirmar a causa nos logs e no codigo.
- Correcao via manifesto: alterar o YAML e `kubectl apply` reconfigura apenas o Deployment alterado.

---

## Fluxo Final

1. Definir variaveis e capturar a zona do cluster `central`.
2. Aguardar o cluster `RUNNING`, obter credenciais e validar nodes.
3. Clonar o `microservices-demo` e aplicar o manifesto; validar Pods e `curl` 200.
4. Criar a metrica baseada em logs `Error_Rate_SLI`.
5. Criar a alerting policy `Error Rate SLI` (Rate > 0.5, sem notificacao).
6. Gerar carga no Locust (300 usuarios, rate 30) e confirmar o incidente.
7. Investigar logs e Pods ate o `productcatalogservice` com `ENABLE_RELOAD`.
8. Remover `ENABLE_RELOAD` do manifesto, reaplicar e confirmar a correcao.
