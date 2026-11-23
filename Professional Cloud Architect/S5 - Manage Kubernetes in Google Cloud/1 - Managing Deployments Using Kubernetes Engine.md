# Managing Deployments Using Kubernetes Engine

## Visao Geral

Praticas de DevOps usam regularmente multiplos deployments para gerenciar cenarios de entrega de aplicacoes como "Continuous deployment", "Blue-Green deployments", "Canary deployments" e outros. Este lab ensina como escalar e gerenciar containers para atender esses cenarios comuns, em que varios deployments heterogeneos sao usados.

Deployments heterogeneos normalmente conectam dois ou mais ambientes de infraestrutura (ou regioes) distintos para atender uma necessidade tecnica ou operacional. Sao chamados de "hybrid", "multi-cloud" ou "public-private", dependendo do caso.

Desafios comuns de deployments limitados a um unico ambiente ou regiao:

- Recursos esgotados: o ambiente pode nao ter compute, rede e storage suficientes para producao.
- Alcance geografico limitado: usuarios distantes acessam um unico ponto central.
- Disponibilidade limitada: padroes de trafego web exigem tolerancia a falhas e resiliencia.
- Vendor lock-in: abstracoes de plataforma dificultam portar aplicacoes.
- Recursos inflexiveis: opcoes restritas de compute, storage ou rede.

Deployments heterogeneos ajudam nesses pontos, mas precisam de processos programaticos, deterministicos e repetiveis. Procedimentos ad-hoc tornam o deploy fragil, podem perder dados ou derrubar trafego.

Tres cenarios comuns:

- multi-cloud deployments;
- fronting on-premises data;
- processos de CI/CD.

## Introducao

Este guia converte o lab (GSP053) para execucao via CLI com `gcloud` e `kubectl`, mantendo os nomes de recursos do enunciado.

Objetivo do lab:
- usar o `kubectl` para explorar o objeto Deployment;
- criar deployments a partir de arquivos YAML;
- escalar deployments;
- fazer rolling update, pausar, retomar e fazer rollback;
- aplicar estrategias de entrega: **canary** e **blue-green**.

Recursos usados no lab: cluster `bootcamp`, deployments `fortune-app-blue`, `fortune-app-canary` e `fortune-app-green`, e o service `fortune-app`.

---

## Pre-requisitos e Variaveis

### Conceito
Padronizar variaveis reduz erro de digitacao. Em Cloud Shell, refaca essa configuracao a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export ZONE="<ZONA_DO_LAB>"   # ajuste conforme o painel do lab (ex: us-central1-a)

# Recursos do lab
export CLUSTER_NAME="bootcamp"
export APP_SERVICE="fortune-app"
export DEPLOY_BLUE="fortune-app-blue"
export DEPLOY_CANARY="fortune-app-canary"
export DEPLOY_GREEN="fortune-app-green"

# Imagens da aplicacao
export IMAGE_REPO="us-central1-docker.pkg.dev/qwiklabs-resources/spl-lab-apps/fortune-service"
export VERSION_V1="1.0.0"
export VERSION_V2="2.0.0"

gcloud config set compute/zone "$ZONE"
```

### Passo 2: Baixar o codigo do lab

```bash
cd "$HOME"
gcloud storage cp -r gs://spls/gsp053/kubernetes .
cd kubernetes
ls deployments services
```

**Resultado esperado:**
- pastas `deployments/` (`fortune-app-blue.yaml`, `fortune-app-canary.yaml`, `fortune-app-green.yaml`) e `services/` (`fortune-app.yaml`, `fortune-app-blue-service.yaml`, `fortune-app-green-service.yaml`).

### Passo 3: Criar o cluster GKE

```bash
gcloud container clusters create "$CLUSTER_NAME" \
  --zone "$ZONE" \
  --machine-type e2-small \
  --num-nodes 3 \
  --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

### Explicacao dos parametros
- `--machine-type e2-small`: tipo de maquina dos nodes.
- `--num-nodes 3`: quantidade de nodes do node pool padrao.
- `--scopes`: escopos OAuth dos nodes (`projecthosting` e leitura/escrita em Cloud Storage).
- `--zone`: cluster zonal na zona do lab.

A criacao leva alguns minutos. Ao final, o `gcloud` ja configura o `kubectl` para o cluster.

### Passo 4: Validar acesso ao cluster

```bash
kubectl get nodes
```

**Resultado esperado:**
- 3 nodes em `Ready`.

---

## TAREFA 1: Conhecer o objeto Deployment

### Conceito
Um **Deployment** declara o estado desejado de um conjunto de Pods (imagem, replicas, labels). O controller cria e gerencia **ReplicaSets** para atingir esse estado e permite atualizacoes controladas (rolling update) e rollback.

### Passos com comandos CLI

```bash
# Documentacao do objeto
kubectl explain deployment

# Todos os campos, recursivamente
kubectl explain deployment --recursive

# Campo especifico
kubectl explain deployment.metadata.name
```

### Explicacao dos parametros
- `explain`: mostra a documentacao do schema do recurso direto da API do cluster.
- `--recursive`: lista a arvore completa de campos.
- `deployment.metadata.name`: navega ate um campo especifico.

### Resultado esperado
- Descricao dos campos `apiVersion`, `kind`, `metadata`, `spec` e `status` do Deployment.

---

## TAREFA 2: Criar um deployment

### Conceito
O deployment `fortune-app-blue` cria 3 replicas da imagem `fortune-service:1.0.0`. Um Service do tipo LoadBalancer (`fortune-app`) expoe a aplicacao externamente selecionando Pods com `app: fortune-app`.

### Passo 1: Examinar o YAML

```bash
cat deployments/fortune-app-blue.yaml
```

Pontos principais do arquivo:

```yaml
kind: Deployment
metadata:
  name: fortune-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fortune-app
  template:
    metadata:
      labels:
        app: fortune-app
        track: stable
        version: "1.0.0"
    spec:
      containers:
        - name: fortune-app
          image: "us-central1-docker.pkg.dev/qwiklabs-resources/spl-lab-apps/fortune-service:1.0.0"
          ports:
            - name: http
              containerPort: 8080
```

### Passo 2: Criar o deployment

```bash
kubectl create -f deployments/fortune-app-blue.yaml
```

### Passo 3: Validar Deployment, ReplicaSet e Pods

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

### Passo 4: Expor a aplicacao com um Service

```bash
kubectl create -f services/fortune-app.yaml
kubectl get services "$APP_SERVICE"
```

### Passo 5: Capturar o IP externo em variavel

O `EXTERNAL-IP` pode levar alguns segundos para ser provisionado. O loop abaixo espera ate o IP existir:

```bash
until EXTERNAL_IP="$(kubectl get svc "$APP_SERVICE" -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')" && [ -n "$EXTERNAL_IP" ]; do
  echo "Aguardando EXTERNAL-IP..."; sleep 5
done
export EXTERNAL_IP
echo "$EXTERNAL_IP"
```

### Passo 6: Testar a versao

```bash
curl "http://${EXTERNAL_IP}/version"

# one-liner do lab (sem variavel)
curl http://`kubectl get svc fortune-app -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```

### Explicacao dos parametros
- `kubectl create -f`: cria os recursos definidos no arquivo YAML.
- `get replicasets`: mostra o ReplicaSet `fortune-app-blue-xxxxxxx` gerado pelo Deployment.
- `-o=jsonpath=...`: extrai um campo especifico do objeto (aqui, o IP do load balancer).

### Resultado esperado
- Deployment `fortune-app-blue` com `3/3` replicas prontas.
- Resposta `{"version":"1.0.0"}`.

> Click **Check my progress**: *Create a Kubernetes cluster and deployments (fortune-app)*.

### Passo 7: Escalar o deployment

```bash
# Escalar para 5 replicas
kubectl scale deployment "$DEPLOY_BLUE" --replicas=5
kubectl get pods | grep "$DEPLOY_BLUE" | wc -l

# Voltar para 3 replicas
kubectl scale deployment "$DEPLOY_BLUE" --replicas=3
kubectl get pods | grep "$DEPLOY_BLUE" | wc -l
```

### Explicacao dos parametros
- `kubectl scale --replicas=N`: atualiza `spec.replicas` do deployment.
- `grep ... | wc -l`: conta os Pods do deployment.

### Resultado esperado
- `5` Pods apos o scale up (pode levar ~1 minuto) e `3` apos o scale down.

### Resumo visual

```text
[Service fortune-app - LoadBalancer] (EXTERNAL_IP:80)
        |
        | selector app: fortune-app
        v
[Deployment fortune-app-blue] --cria--> [ReplicaSet fortune-app-blue-xxxx]
                                                  |
                                                  v
                                   [Pod 1.0.0] [Pod 1.0.0] [Pod 1.0.0]
                                   kubectl scale: 3 -> 5 -> 3 replicas
```

---

## TAREFA 3: Rolling update

### Conceito
Ao alterar o template de Pods (ex.: imagem), o Deployment cria um **novo ReplicaSet** e substitui os Pods gradualmente, sem indisponibilidade. Cada alteracao vira uma **revisao** no historico, permitindo pausar, retomar e fazer rollback.

### Passo 1: Disparar o rolling update

Opcao do lab (editor interativo):

```bash
kubectl edit deployment "$DEPLOY_BLUE"
```

No editor (`vi`), pressione `i` e altere:
- `image: ".../fortune-service:1.0.0"` para `image: ".../fortune-service:2.0.0"`;
- na secao `env`, a variavel `APP_VERSION`: `value: "1.0.0"` para `value: "2.0.0"`.

Salve com `Esc`, `:wq`, `Enter`.

Opcao CLI nao interativa (equivalente, altera imagem e `APP_VERSION` em **uma unica revisao**):

```bash
kubectl patch deployment "$DEPLOY_BLUE" -p "{
  \"spec\": {\"template\": {\"spec\": {\"containers\": [{
    \"name\": \"fortune-app\",
    \"image\": \"${IMAGE_REPO}:${VERSION_V2}\",
    \"env\": [{\"name\": \"APP_VERSION\", \"value\": \"${VERSION_V2}\"}]
  }]}}}
}"
```

Observacao: evite rodar `kubectl set image` e `kubectl set env` separados; cada um gera uma revisao, e o `rollout undo` voltaria para um estado intermediario (imagem 2.0.0 com `APP_VERSION=1.0.0`).

### Passo 2: Acompanhar ReplicaSets e historico

```bash
kubectl get replicaset
kubectl rollout history deployment/"$DEPLOY_BLUE"
```

### Passo 3: Pausar o rollout

```bash
kubectl rollout pause deployment/"$DEPLOY_BLUE"
kubectl rollout status deployment/"$DEPLOY_BLUE"
```

O `status` pode retornar imediatamente "successfully rolled out"; isso confirma o pause, nao o fim da atualizacao.

### Passo 4: Conferir versao de cada Pod

Comando do lab (curl no IP de cada Pod; use `Ctrl+C` para sair se travar):

```bash
for p in $(kubectl get pods -l app=fortune-app -o=jsonpath='{.items[*].metadata.name}'); do
  echo $p && curl -s http://$(kubectl get pod $p -o=jsonpath='{.status.podIP}')/version; echo
done
```

Alternativa que nao depende de alcancar o IP do Pod a partir do Cloud Shell:

```bash
kubectl get pods -l app=fortune-app \
  -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

**Resultado esperado:** mistura de Pods `1.0.0` e `2.0.0` (rollout pausado no meio).

### Passo 5: Retomar o rollout

```bash
kubectl rollout resume deployment/"$DEPLOY_BLUE"
kubectl rollout status deployment/"$DEPLOY_BLUE"
```

### Passo 6: Rollback

Simulando um bug na nova versao:

```bash
kubectl rollout undo deployment/"$DEPLOY_BLUE"
kubectl rollout status deployment/"$DEPLOY_BLUE"

curl "http://${EXTERNAL_IP}/version"
```

### Explicacao dos parametros
- `kubectl edit` / `kubectl patch`: alteram o template de Pods e disparam o rolling update.
- `rollout history`: lista as revisoes do deployment.
- `rollout pause` / `rollout resume`: congelam e retomam a substituicao de Pods.
- `rollout status`: acompanha o progresso ate a conclusao.
- `rollout undo`: volta para a revisao anterior.

### Resultado esperado
- Apos o `resume`, todos os Pods em `2.0.0`.
- Apos o `undo`, resposta `{"version":"1.0.0"}` (o rollback pode levar alguns instantes).

### Resumo visual

```text
[fortune-app-blue]
        |
        | kubectl edit / patch  (1.0.0 -> 2.0.0)
        v
[ReplicaSet antigo 1.0.0] --pods migram-->  [ReplicaSet novo 2.0.0]
        ^              (pause / resume)              |
        |                                            |
        +--------- kubectl rollout undo -------------+
```

---

## TAREFA 4: Canary deployment

### Conceito
Um **canary** roda a nova versao para um subconjunto de usuarios em producao. O deployment `fortune-app-canary` usa os mesmos labels `app: fortune-app`, entao o Service `fortune-app` distribui o trafego entre Pods do blue (1.0.0) e do canary (2.0.0), na proporcao do numero de replicas.

### Passo 1: Examinar e criar o canary

```bash
cat deployments/fortune-app-canary.yaml

kubectl create -f deployments/fortune-app-canary.yaml
kubectl get deployments
```

### Passo 2: Verificar a distribuicao de trafego

```bash
for i in {1..10}; do curl -s "http://${EXTERNAL_IP}/version"; echo; done
```

Rode algumas vezes.

### Explicacao dos parametros
- Selector do Service `app: fortune-app`: casa Pods dos dois deployments (`fortune-app-blue` e `fortune-app-canary`).
- Proporcao de replicas: define aproximadamente a fatia de trafego que vai para o canary.

### Resultado esperado
- Dois deployments: `fortune-app-blue` e `fortune-app-canary`.
- Maioria das respostas `1.0.0` e uma parcela menor `2.0.0`.

> Click **Check my progress**: *Canary Deployment*.

### Resumo visual

```text
              [Service fortune-app]
              (selector app: fortune-app)
                      |
          +-----------+-----------+
          |                       |
          v                       v
[fortune-app-blue]        [fortune-app-canary]
[1.0.0 - maioria]         [2.0.0 - fatia menor]
```

---

## TAREFA 5: Blue-green deployment

### Conceito
No **blue-green**, duas versoes completas rodam lado a lado (blue = 1.0.0, green = 2.0.0) e o trafego e trocado **de uma vez** alterando o selector do Service. O rollback e apenas reapontar o Service para o blue.

### Passo 1: Apontar o Service apenas para o blue

```bash
kubectl apply -f services/fortune-app-blue-service.yaml
```

### Passo 2: Criar o deployment green

```bash
kubectl create -f deployments/fortune-app-green.yaml
kubectl rollout status deployment/"$DEPLOY_GREEN"

# Ainda deve responder 1.0.0
curl "http://${EXTERNAL_IP}/version"
```

### Passo 3: Trocar o trafego para o green

```bash
kubectl apply -f services/fortune-app-green-service.yaml

# Agora sempre 2.0.0
for i in {1..5}; do curl -s "http://${EXTERNAL_IP}/version"; echo; done
```

### Passo 4: Rollback blue-green

```bash
kubectl apply -f services/fortune-app-blue-service.yaml

# De volta a 1.0.0
for i in {1..5}; do curl -s "http://${EXTERNAL_IP}/version"; echo; done
```

### Explicacao dos parametros
- `kubectl apply -f`: atualiza o Service existente com o novo selector (blue ou green), mantendo o mesmo IP externo.
- `rollout status`: garante que o green esta pronto antes de receber trafego.

### Resultado esperado
- Antes da troca: sempre `1.0.0`.
- Apos apontar para o green: sempre `2.0.0`.
- Apos o rollback: sempre `1.0.0`.

> Click **Check my progress**: *Blue-green deployment*.

### Resumo visual

```text
              [Service fortune-app]
                      |
   fortune-app-blue-service.yaml  |  fortune-app-green-service.yaml
          +-----------+-----------+
          |                       |
          v                       v
[fortune-app-blue]        [fortune-app-green]
[1.0.0]                   [2.0.0]

rollback = kubectl apply -f services/fortune-app-blue-service.yaml
```

---

## Validacao

```bash
# Cluster
gcloud container clusters list --format="table(name,location,status,currentNodeCount)"

# Deployments e replicas
kubectl get deployments -o wide

# Historico de revisoes do blue
kubectl rollout history deployment/"$DEPLOY_BLUE"

# Selector atual do Service (blue ou green)
kubectl get svc "$APP_SERVICE" -o=jsonpath='{.spec.selector}'; echo

# Versao servida
curl "http://${EXTERNAL_IP}/version"
```

Checklist esperado:
- cluster `bootcamp` com 3 nodes;
- deployments `fortune-app-blue`, `fortune-app-canary` e `fortune-app-green`;
- Service `fortune-app` com IP externo, apontando para o blue apos o rollback final.

---

## Resumo Visual da Arquitetura

Compilado: topologia geral e estado final ao termino do lab. Os diagramas de cada estrategia estao no `### Resumo visual` das Tarefas 2 a 5.

```text
[Cloud Shell] --kubectl--> [GKE cluster bootcamp] (ZONE, 3 nodes e2-small)

[Cliente / curl]
        |
        | curl http://EXTERNAL_IP/version
        v
[Service fortune-app - LoadBalancer] (EXTERNAL_IP:80)
        |
        | selector = blue (apos rollback da Tarefa 5)
        v
[fortune-app-blue]  3 Pods 1.0.0 :8080   <-- recebe o trafego

[fortune-app-canary]  Pods 2.0.0         (existe, fora do selector)
[fortune-app-green]   Pods 2.0.0         (existe, fora do selector)
```

---

## Troubleshooting

- `EXTERNAL-IP` em `<pending>`:
  - o load balancer ainda esta sendo criado; aguarde e rode `kubectl get svc fortune-app` novamente (ou use o loop do Passo 5 da Tarefa 2).
- `EXTERNAL_IP` vazio apos reconectar o Cloud Shell:
  - variaveis nao persistem; rode de novo o bloco de variaveis e o loop de captura do IP.
- Loop de `curl` por IP de Pod travado:
  - o Cloud Shell pode nao alcancar o IP do Pod; use `Ctrl+C` e a alternativa com `custom-columns`.
- `kubectl edit` abriu o editor e voce nao sabe sair:
  - `Esc`, depois `:wq` + `Enter` para salvar, ou `:q!` para descartar.
- `rollout undo` voltou para estado inesperado:
  - confira `kubectl rollout history` e use `kubectl rollout undo deployment/fortune-app-blue --to-revision=<N>`.
- `AlreadyExists` no `kubectl create`:
  - o recurso ja foi criado; use `kubectl get` para confirmar ou `kubectl apply -f` para atualizar.

---

## Limpeza (Opcional)

```bash
kubectl delete -f services/fortune-app.yaml --ignore-not-found
kubectl delete deployment "$DEPLOY_BLUE" "$DEPLOY_CANARY" "$DEPLOY_GREEN" --ignore-not-found

gcloud container clusters delete "$CLUSTER_NAME" --zone "$ZONE" --quiet
```

---

## Conceitos-Chave

- Deployment: estado desejado dos Pods; gerencia ReplicaSets e revisoes.
- ReplicaSet: mantem o numero de replicas de um template especifico de Pod.
- Scale: altera `spec.replicas` sem trocar a versao.
- Rolling update: troca gradual de Pods com historico, pause/resume e rollback (`rollout undo`).
- Canary: nova versao recebe uma fatia do trafego pelo mesmo selector do Service.
- Blue-green: duas versoes completas; troca de trafego instantanea mudando o selector do Service.

---

## Fluxo Final

1. Definir variaveis, baixar o codigo e criar o cluster `bootcamp`.
2. Explorar o objeto Deployment com `kubectl explain`.
3. Criar `fortune-app-blue` + Service `fortune-app` e validar `1.0.0`.
4. Escalar para 5 e voltar para 3 replicas.
5. Rolling update para `2.0.0`, pausar, conferir Pods, retomar e fazer rollback.
6. Criar `fortune-app-canary` e observar trafego dividido.
7. Blue-green: apontar Service para blue, criar green, trocar para green e fazer rollback para blue.
