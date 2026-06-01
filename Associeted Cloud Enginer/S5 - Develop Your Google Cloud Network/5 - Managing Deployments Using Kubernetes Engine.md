# Managing Deployments Using Kubernetes Engine

## Visão Geral

Práticas de DevOps usam múltiplos deployments para cenários como Continuous Deployment, Blue-Green e Canary. Neste lab, você aprende a escalar e gerenciar containers para executar esses cenários com workloads heterogêneos de forma controlada e repetível.

## Introdução

Este guia converte o lab para uma execução orientada a CLI, priorizando comandos repetíveis e validação por etapa.

Objetivos práticos:

- Usar `kubectl` para inspecionar e operar deployments.
- Criar/usar manifests YAML de deployment e service.
- Lançar, atualizar e escalar deployments.
- Praticar estilos de rollout: rolling update, canary e blue-green.

## Pré-requisitos e Variáveis

### Conceito

Padronizar variáveis reduz erro manual e facilita repetir os passos em outros ambientes.

### Passos com comandos CLI

```bash
# Ajuste conforme ambiente
export ZONE="<SUA_ZONE>"
export CLUSTER_NAME="bootcamp"
export REPO_DIR="$HOME/kubernetes"
export DEPLOYMENT="fortune-app-blue"
export SERVICE="fortune-app"

gcloud config set compute/zone "$ZONE"

gcloud storage cp -r gs://spls/gsp053/kubernetes .
cd "$REPO_DIR"

gcloud container clusters create "$CLUSTER_NAME" \
  --machine-type e2-small \
  --num-nodes 3 \
  --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

### Explicação dos parâmetros

- `gcloud config set compute/zone`: define zona padrão para recursos zonais.
- `gcloud storage cp -r`: baixa o código e manifests do lab.
- `gcloud container clusters create`: cria cluster GKE.
- `--machine-type`: tipo de VM de cada node.
- `--num-nodes`: quantidade inicial de nodes.
- `--scopes`: escopos de acesso no node pool.

### Resultado esperado

- Cluster `bootcamp` criado e acessível pelo `kubectl`.
- Diretório local `kubernetes` com manifests de deployment/service.

## TAREFA 1: Explorar o objeto Deployment

### Conceito

Antes de aplicar manifests, entender campos de `Deployment` acelera troubleshooting e atualizações seguras.

### Passos com comandos CLI

```bash
kubectl explain deployment
kubectl explain deployment --recursive
kubectl explain deployment.metadata.name
```

### Explicação dos parâmetros

- `explain`: mostra schema e descrição de campos da API.
- `--recursive`: expande todos os subcampos.

### Resultado esperado

- Clareza sobre campos importantes: `spec.replicas`, `selector`, `template`, `containers.image`.

## TAREFA 2: Criar deployment e service base

### Conceito

O deployment `fortune-app-blue` será a base de produção inicial (versão 1.0.0), exposta por um Service do tipo LoadBalancer.

### Passos com comandos CLI

```bash
cat deployments/fortune-app-blue.yaml

kubectl create -f deployments/fortune-app-blue.yaml
kubectl get deployments
kubectl get replicasets
kubectl get pods

kubectl create -f services/fortune-app.yaml
kubectl get services "$SERVICE"

# Aguarde EXTERNAL-IP e teste
curl "http://$(kubectl get svc "$SERVICE" -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/version"
```

### Explicação dos parâmetros

- `kubectl create -f`: cria recursos a partir do YAML.
- `kubectl get deployments|replicasets|pods`: valida cadeia Deployment -> ReplicaSet -> Pods.
- `kubectl get svc -o=jsonpath=...`: extrai IP externo para teste rápido.

### Resultado esperado

- Deployment `fortune-app-blue` com 3 réplicas.
- Service `fortune-app` com IP externo.
- Endpoint `/version` retornando JSON com `1.0.0`.

## TAREFA 3: Escalar deployment

### Conceito

Escala horizontal altera número de réplicas sem alterar imagem ou configuração funcional.

### Passos com comandos CLI

```bash
kubectl scale deployment "$DEPLOYMENT" --replicas=5
kubectl get pods | grep "$DEPLOYMENT" | wc -l

kubectl scale deployment "$DEPLOYMENT" --replicas=3
kubectl get pods | grep "$DEPLOYMENT" | wc -l
```

### Explicação dos parâmetros

- `kubectl scale`: altera `spec.replicas` em tempo de execução.
- `--replicas`: número alvo de pods.

### Resultado esperado

- Contagem de pods sobe para 5 e volta para 3.

## TAREFA 4: Rolling update, pausa, resume e rollback

### Conceito

Rolling update troca versões gradualmente para reduzir indisponibilidade. Pausa/resume permitem controle fino, e rollback reverte mudanças com histórico do deployment.

### Passos com comandos CLI

```bash
# Atualiza imagem para 2.0.0 de forma não interativa
kubectl set image deployment/fortune-app-blue \
  fortune-app=us-central1-docker.pkg.dev/qwiklabs-resources/spl-lab-apps/fortune-service:2.0.0

# Atualiza variável de ambiente APP_VERSION para 2.0.0
kubectl set env deployment/fortune-app-blue APP_VERSION=2.0.0

kubectl get replicaset
kubectl rollout history deployment/fortune-app-blue

# Pausa rollout
kubectl rollout pause deployment/fortune-app-blue
kubectl rollout status deployment/fortune-app-blue

# Inspeciona versões por pod (pode mostrar mix 1.0.0/2.0.0 durante pausa)
for p in $(kubectl get pods -l app=fortune-app -o=jsonpath='{.items[*].metadata.name}'); do
  echo "$p"
  curl -s "http://$(kubectl get pod "$p" -o=jsonpath='{.status.podIP}')/version"
  echo
done

# Retoma rollout
kubectl rollout resume deployment/fortune-app-blue
kubectl rollout status deployment/fortune-app-blue

# Rollback para versão anterior
kubectl rollout undo deployment/fortune-app-blue
curl "http://$(kubectl get svc "$SERVICE" -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/version"
```

### Explicação dos parâmetros

- `kubectl set image`: altera imagem do container sem editar YAML manualmente.
- `kubectl set env`: altera variável de ambiente do deployment.
- `kubectl rollout pause|resume|undo`: controla ciclo do rollout.
- `kubectl rollout history`: exibe revisões.

### Resultado esperado

- Nova revisão criada para 2.0.0.
- Rollout pausado e retomado com sucesso.
- Rollback retorna resposta `1.0.0`.

## TAREFA 5: Canary deployment

### Conceito

Canary envia pequena parcela do tráfego para versão nova, mantendo maioria na versão estável.

### Passos com comandos CLI

```bash
cat deployments/fortune-app-canary.yaml
kubectl create -f deployments/fortune-app-canary.yaml
kubectl get deployments

for i in {1..10}; do
  curl -s "http://$(kubectl get svc "$SERVICE" -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/version"
  echo
done
```

### Explicação dos parâmetros

- `fortune-app-canary.yaml`: cria deployment adicional (versão 2.0.0) com menos réplicas.
- Service permanece com seletor `app: fortune-app`, atendendo pods blue e canary.

### Resultado esperado

- Respostas misturadas: maioria `1.0.0` e pequena parte `2.0.0`.

## TAREFA 6: Blue-Green deployment

### Conceito

Blue-Green separa completamente versão atual (blue) e nova versão (green), com troca instantânea de tráfego via Service selector.

### Passos com comandos CLI

```bash
# Garante service apontando para blue (1.0.0)
kubectl apply -f services/fortune-app-blue-service.yaml

# Cria green (2.0.0)
kubectl create -f deployments/fortune-app-green.yaml

# Ainda deve servir blue
curl "http://$(kubectl get svc "$SERVICE" -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/version"

# Troca para green
kubectl apply -f services/fortune-app-green-service.yaml
curl "http://$(kubectl get svc "$SERVICE" -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/version"

# Rollback blue-green
kubectl apply -f services/fortune-app-blue-service.yaml
curl "http://$(kubectl get svc "$SERVICE" -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/version"
```

### Explicação dos parâmetros

- `kubectl apply -f`: aplica alterações idempotentes nos Services.
- Troca de seletor no Service muda destino do tráfego sem recriar o endpoint.

### Resultado esperado

- Com service blue: respostas sempre `1.0.0`.
- Com service green: respostas sempre `2.0.0`.
- Reaplicando blue: rollback imediato para `1.0.0`.

## Validação e Testes

Use esta bateria ao final para confirmar todos os critérios:

```bash
kubectl get nodes
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
kubectl get svc fortune-app
kubectl rollout history deployment/fortune-app-blue
kubectl describe deployment fortune-app-blue

# Verificar versão exposta
curl "http://$(kubectl get svc fortune-app -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/version"
```

Checklist rápido:

- Cluster GKE criado com 3 nodes.
- Deployment base e service criados.
- Escala horizontal testada.
- Rolling update/pause/resume/undo executados.
- Canary e Blue-Green validados com `curl`.

## Troubleshooting

```bash
# Pods e eventos
kubectl get pods -A
kubectl describe pod <POD_NAME>
kubectl get events --sort-by=.lastTimestamp

# Service sem EXTERNAL-IP
kubectl get svc fortune-app -w

# Revisões e rollout travado
kubectl rollout status deployment/fortune-app-blue
kubectl rollout history deployment/fortune-app-blue
```

Problemas comuns:

- EXTERNAL-IP pendente por alguns minutos: comportamento normal no provisionamento do LoadBalancer.
- `curl` sem resposta: confira readiness dos pods e seletor do service.
- Versão inesperada em canary: esperado haver mistura de respostas.

## Limpeza (Opcional)

```bash
kubectl delete -f deployments/fortune-app-canary.yaml --ignore-not-found
kubectl delete -f deployments/fortune-app-green.yaml --ignore-not-found
kubectl delete -f services/fortune-app-green-service.yaml --ignore-not-found
kubectl delete -f services/fortune-app-blue-service.yaml --ignore-not-found
kubectl delete -f services/fortune-app.yaml --ignore-not-found
kubectl delete -f deployments/fortune-app-blue.yaml --ignore-not-found

gcloud container clusters delete "$CLUSTER_NAME" --zone "$ZONE" --quiet
```

## Conceitos-Chave

| Conceito | O que faz | Quando usar |
|---|---|---|
| Deployment | Gerencia réplicas e revisões de pods | Ciclo de vida de aplicações stateless |
| ReplicaSet | Mantém número desejado de pods | Camada interna de controle do Deployment |
| Service | Abstrai descoberta e balanceamento | Expor app interna/externamente |
| Rolling Update | Atualiza gradualmente versões | Reduzir risco e downtime |
| Canary | Envia pequena parcela para nova versão | Validação progressiva em produção |
| Blue-Green | Alterna tráfego entre dois ambientes | Troca rápida e rollback imediato |

## Fluxo Final

1. Provisionar cluster e baixar manifests.
2. Publicar `fortune-app-blue` e expor por service.
3. Escalar réplicas conforme necessidade.
4. Atualizar versão com rolling update e controlar rollout.
5. Validar estratégia canary.
6. Validar estratégia blue-green com rollback.