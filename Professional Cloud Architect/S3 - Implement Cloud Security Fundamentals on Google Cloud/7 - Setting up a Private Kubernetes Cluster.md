# Setting up a Private Kubernetes Cluster

## Visao Geral

No Kubernetes Engine, um cluster privado e um cluster que torna o master inacessivel pela internet publica. Em um cluster privado, os nodes nao possuem enderecos IP publicos, apenas enderecos privados, de forma que as workloads executam em um ambiente isolado. Nodes e master se comunicam entre si por VPC peering.

Na API do Kubernetes Engine, os intervalos de enderecos sao representados como blocos CIDR (Classless Inter-Domain Routing).

Neste lab, voce vai aprender a criar um cluster Kubernetes privado.

## Introducao

Este guia converte o desafio para uma execucao 100% via CLI (`gcloud` e `kubectl`), no mesmo padrao editorial dos outros labs.

Objetivo do laboratorio:
- criar um cluster GKE privado com subnet automatica;
- validar ranges primario/secundarios criados por IP alias;
- habilitar acesso ao control plane via Master Authorized Networks;
- criar um segundo cluster privado usando subnet customizada.

Os nomes de recursos definidos pelo enunciado foram preservados: `private-cluster`, `source-instance`, `my-subnet` e `private-cluster2`.

---

## TAREFA 1: Definir regiao, zona e variaveis

### Conceito
Padronizar variaveis reduz erro de digitacao e facilita reaproveitar comandos durante o lab. Em Cloud Shell, refaca essa configuracao a cada nova sessao/reconexao.

### Passos

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export REGION="us-east4"
export ZONE="us-east4-b"

# Recursos exigidos pelo lab
export CLUSTER_1="private-cluster"
export CLUSTER_2="private-cluster2"
export SOURCE_VM="source-instance"
export CUSTOM_SUBNET="my-subnet"

# Configuracoes padrao do gcloud
gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"
```

### Explicacao dos parametros
- `REGION`: regiao onde os recursos regionais (subnets, etc.) serao criados.
- `ZONE`: zona usada para recursos zonais (clusters zonais e VM).
- `gcloud config set compute/region|zone`: define defaults para evitar repetir flags.

### Resultado esperado
- `gcloud config list` exibe `compute/region` e `compute/zone` com os valores escolhidos.

---

## TAREFA 2: Criar o cluster privado inicial

### Conceito
Um cluster privado no GKE usa nodes sem IP publico. O control plane recebe um bloco `/28` dedicado (`--master-ipv4-cidr`) e o IP alias cria/usa ranges secundarios para Pods e Services.

### Passos com comandos CLI

```bash
gcloud beta container clusters create "$CLUSTER_1" \
  --enable-private-nodes \
  --master-ipv4-cidr 172.16.0.16/28 \
  --enable-ip-alias \
  --create-subnetwork "" \
  --machine-type e2-medium \
  --zone "$ZONE"
```

### Explicacao dos parametros
- `--enable-private-nodes`: cria nodes apenas com IP interno.
- `--master-ipv4-cidr 172.16.0.16/28`: bloco reservado para componentes do control plane.
- `--enable-ip-alias`: habilita VPC-native cluster (Pods/Services com ranges secundarios).
- `--create-subnetwork ""`: permite ao GKE criar subnet automaticamente para o cluster.
- `--machine-type e2-medium`: tipo de maquina dos nodes.
- `--zone "$ZONE"`: garante criacao na zona correta do desafio.

### Resultado esperado
- Cluster `private-cluster` em status `RUNNING` (a criacao leva alguns minutos).

> Click **Check my progress**: *Create a private cluster*.

---

## TAREFA 3: Inspecionar subnet e ranges secundarios

### Conceito
Ao usar IP alias com subnet automatica, o GKE cria:
- range primario para nodes;
- range secundario para Pods;
- range secundario para Services.

Tambem habilita `privateIpGoogleAccess`, permitindo saida privada para APIs Google.

### Passos com comandos CLI

```bash
# 1) Listar subnets da rede default
gcloud compute networks subnets list --network default

# 2) Guardar o nome da subnet criada automaticamente para o cluster
export SUBNET_NAME="$(gcloud compute networks subnets list \
  --network default \
  --regions "$REGION" \
  --filter="name~^gke-${CLUSTER_1}-subnet" \
  --format='value(name)' | head -n1)"
echo "$SUBNET_NAME"

# 3) Descrever detalhes da subnet
gcloud compute networks subnets describe "$SUBNET_NAME" --region "$REGION"
```

### Explicacao dos parametros
- `--network default`: filtra subnets da VPC default.
- `SUBNET_NAME`: subnet gerada pelo GKE (ex.: `gke-private-cluster-subnet-xxxx`), capturada pelo filtro `name~^gke-private-cluster-subnet`.
- `--region "$REGION"`: subnets sao recursos regionais.

### Resultado esperado
- No `describe`, validar:
  - `ipCidrRange` (range primario).
  - `secondaryIpRanges` com ranges de Pods e Services.
  - `privateIpGoogleAccess: true`.

---

## TAREFA 4: Habilitar Master Authorized Networks

### Conceito
Por padrao, o acesso ao control plane fica restrito aos ranges internos do cluster/subnet (range primario dos nodes e range secundario dos Pods). Para administrar o cluster de fora dessa faixa, e necessario autorizar CIDRs explicitos.

### Passos com comandos CLI

```bash
# 1) Criar VM de origem para testes e administracao
gcloud compute instances create "$SOURCE_VM" \
  --zone "$ZONE" \
  --machine-type e2-medium \
  --scopes "https://www.googleapis.com/auth/cloud-platform"
```

> Click **Check my progress**: *Create a VM instance*.

```bash
# 2) Obter IP externo (NAT IP) da VM direto em variavel
export SOURCE_NAT_IP="$(gcloud compute instances describe "$SOURCE_VM" --zone "$ZONE" \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)')"

# 3) Montar a faixa autorizada (IP/32 = somente esse IP)
export MY_EXTERNAL_RANGE="${SOURCE_NAT_IP}/32"
echo "$MY_EXTERNAL_RANGE"

# 4) Autorizar acesso ao master do cluster privado
gcloud container clusters update "$CLUSTER_1" \
  --enable-master-authorized-networks \
  --master-authorized-networks "$MY_EXTERNAL_RANGE" \
  --zone "$ZONE"
```

Equivalente do lab para conferir o IP: `gcloud compute instances describe source-instance --zone=$ZONE | grep natIP`.

### Explicacao dos parametros
- `--scopes cloud-platform`: libera escopo amplo para comandos administrativos na VM.
- `--master-authorized-networks`: lista CIDRs permitidos a acessar o endpoint do master.
- `IP/32`: autoriza somente um IP especifico (boa pratica para lab/teste).

### Resultado esperado
- Atualizacao do cluster concluida com Master Authorized Networks ativo.

> Click **Check my progress**: *Authorize your external address range*.

---

## TAREFA 4 (continuacao): Acessar cluster e validar ausencia de IP externo nos nodes

### Conceito
A validacao principal do cluster privado e confirmar que os nodes nao recebem IP externo.

### Passos com comandos CLI

```bash
# 1) Acessar a VM de origem (responda Y e de ENTER nas perguntas de passphrase)
gcloud compute ssh "$SOURCE_VM" --zone "$ZONE"
```

No shell da VM (`source-instance`), execute. **As variaveis do Cloud Shell nao existem dentro da VM**, por isso o `ZONE` e exportado de novo:

```bash
# 2) Variaveis dentro da VM
export ZONE="us-east4-b"

# 3) Instalar kubectl e plugin de autenticacao GKE
sudo apt-get update
sudo apt-get install -y kubectl google-cloud-cli-gke-gcloud-auth-plugin

# 4) Baixar credenciais do cluster
gcloud container clusters get-credentials private-cluster --zone "$ZONE"

# 5) Validar IPs dos nodes
kubectl get nodes -o yaml | grep -A4 addresses
kubectl get nodes -o wide
```

Para sair da VM:

```bash
exit
```

### Explicacao dos parametros
- `get-credentials`: atualiza kubeconfig para administrar o cluster.
- `kubectl get nodes -o wide`: mostra coluna `EXTERNAL-IP` dos nodes.

### Resultado esperado
- Nodes com `InternalIP` preenchido.
- `ExternalIP` vazio/ausente.

---

## TAREFA 5: Limpeza parcial do ambiente (cluster 1)

### Conceito
Remover recursos que nao serao mais usados evita custo e conflito de nomes.

### Passos com comandos CLI

```bash
gcloud container clusters delete "$CLUSTER_1" \
  --zone "$ZONE" \
  --quiet
```

### Explicacao dos parametros
- `--quiet`: evita confirmacao interativa (equivale a responder `Y`).
- Exclusao apenas do `private-cluster` nesta etapa, conforme roteiro do desafio. A `source-instance` continua sendo usada na Tarefa 6.

### Resultado esperado
- Cluster `private-cluster` removido.

> Click **Check my progress**: *Delete the Kubernetes cluster*.

---

## TAREFA 6: Criar cluster privado com subnet customizada

### Conceito
Neste fluxo, voce controla explicitamente o range primario da subnet (nodes) e os ranges secundarios (Pods/Services), em vez de deixar o GKE criar automaticamente.

### Passos com comandos CLI

```bash
# 1) Criar subnet customizada com ranges secundarios
gcloud compute networks subnets create "$CUSTOM_SUBNET" \
  --network default \
  --range 10.0.4.0/22 \
  --enable-private-ip-google-access \
  --region "$REGION" \
  --secondary-range my-svc-range=10.0.32.0/20,my-pod-range=10.4.0.0/14
```

> Click **Check my progress**: *Create a subnetwork and secondary ranges in us-east4 region*.

```bash
# 2) Criar novo cluster privado usando a subnet customizada
gcloud beta container clusters create "$CLUSTER_2" \
  --enable-private-nodes \
  --enable-ip-alias \
  --master-ipv4-cidr 172.16.0.32/28 \
  --subnetwork "$CUSTOM_SUBNET" \
  --services-secondary-range-name my-svc-range \
  --cluster-secondary-range-name my-pod-range \
  --zone "$ZONE" \
  --machine-type e2-medium
```

> Click **Check my progress**: *Create a private cluster that uses your subnetwork*.

```bash
# 3) Capturar novamente o NAT IP da source-instance
export SOURCE_NAT_IP="$(gcloud compute instances describe "$SOURCE_VM" --zone "$ZONE" \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)')"
export MY_EXTERNAL_RANGE="${SOURCE_NAT_IP}/32"
echo "$MY_EXTERNAL_RANGE"

# 4) Autorizar o range para o cluster 2
gcloud container clusters update "$CLUSTER_2" \
  --enable-master-authorized-networks \
  --master-authorized-networks "$MY_EXTERNAL_RANGE" \
  --zone "$ZONE"
```

> Click **Check my progress**: *Authorize your external address range for private cluster in custom subnetwork*.

```bash
# 5) Conectar via VM e validar nodes privados no cluster 2
gcloud compute ssh "$SOURCE_VM" --zone "$ZONE"
```

No shell da VM (kubectl e plugin ja instalados na Tarefa 4):

```bash
export ZONE="us-east4-b"
gcloud container clusters get-credentials private-cluster2 --zone "$ZONE"
kubectl get nodes -o yaml | grep -A4 addresses
kubectl get nodes -o wide
```

Saia da VM:

```bash
exit
```

### Explicacao dos parametros
- `--range 10.0.4.0/22`: range primario para nodes da subnet.
- `--secondary-range ...`: define faixas separadas para Services e Pods.
- `--services-secondary-range-name`: associa faixa de Services ao cluster.
- `--cluster-secondary-range-name`: associa faixa de Pods ao cluster.

### Resultado esperado
- `private-cluster2` em `RUNNING`.
- Nodes sem IP externo, com `InternalIP` dentro de `10.0.4.0/22` (ex.: `10.0.4.3`).
- Acesso ao master liberado para o range dos nodes (`10.0.4.0/22`), o range de Pods (`10.4.0.0/14`) e o `/32` da `source-instance`.

---

## Validacao e Testes

Use os comandos abaixo para validar o estado final do desafio:

```bash
# Clusters
gcloud container clusters list \
  --format="table(name,location,status,privateClusterConfig.enablePrivateNodes)"

# Master authorized networks do cluster 2
gcloud container clusters describe private-cluster2 --zone "$ZONE" \
  --format="yaml(masterAuthorizedNetworksConfig)"

# Subnet customizada e ranges
gcloud compute networks subnets describe my-subnet --region "$REGION" \
  --format="yaml(name,ipCidrRange,privateIpGoogleAccess,secondaryIpRanges)"

# VM de origem
gcloud compute instances describe source-instance --zone "$ZONE" \
  --format="table(name,status,networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs[0].natIP)"
```

Checklist de validacao:
- `private-cluster` removido (apos tarefa de limpeza).
- `private-cluster2` ativo e privado.
- `my-subnet` com `privateIpGoogleAccess: true`.
- Ranges secundarios `my-svc-range` e `my-pod-range` presentes.
- Nodes do cluster 2 sem `EXTERNAL-IP`.

---

## Troubleshooting

- Erro de permissao ao acessar o master:
  - confirme `MY_EXTERNAL_RANGE` com NAT IP atual da `source-instance`.
  - se o NAT IP mudou, rode novamente `clusters update` com o novo `/32`.

- Erro de zona/regiao inconsistente:
  - valide `echo "$REGION"` e `echo "$ZONE"`.
  - confirme `gcloud config list` para evitar criar recurso na localizacao errada.

- `kubectl` falhando por plugin de autenticacao:
  - reinstale `google-cloud-cli-gke-gcloud-auth-plugin` na `source-instance`.

- `get-credentials` dentro da VM reclama de zona/localizacao:
  - a VM nao herda as variaveis do Cloud Shell; rode `export ZONE=us-east4-b` dentro da VM.

- `SUBNET_NAME` vazio na Tarefa 3:
  - o cluster ainda nao terminou de criar ou o filtro nao bateu; liste com `gcloud compute networks subnets list --network default` e exporte o nome manualmente.

- Falha por range CIDR sobreposto:
  - ajuste ranges da subnet customizada para blocos nao utilizados no projeto (ajuste conforme ambiente).

---

## Limpeza (Opcional)

Se quiser encerrar totalmente o ambiente apos o lab:

```bash
gcloud container clusters delete private-cluster2 --zone "$ZONE" --quiet
gcloud compute instances delete source-instance --zone "$ZONE" --quiet
gcloud compute networks subnets delete my-subnet --region "$REGION" --quiet
```

---

## Conceitos-Chave

- Cluster privado GKE: nodes sem IP publico.
- `master-ipv4-cidr`: bloco dedicado ao control plane.
- IP alias: separa ranges para nodes, Pods e Services.
- `privateIpGoogleAccess`: acesso privado a APIs Google sem IP externo.
- Master Authorized Networks: lista explicita de CIDRs permitidos no endpoint do master.

---

## Fluxo Final

1. Definir `REGION=us-east4` e `ZONE=us-east4-b`.
2. Criar `private-cluster` com subnet automatica.
3. Inspecionar subnet/ranges gerados.
4. Criar `source-instance` e autorizar NAT IP no master.
5. Conectar com `kubectl` e validar nodes sem IP externo.
6. Remover `private-cluster`.
7. Criar `my-subnet` customizada.
8. Criar `private-cluster2` usando ranges customizados.
9. Autorizar acesso ao master do cluster 2 e validar novamente.
