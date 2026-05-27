# Build a Secure Google Cloud Network - Desafio Final

## Introducao

Este guia converte o challenge lab para execucao via CLI no padrao do template de referencia, com foco em seguranca de rede para o ambiente do site juice-shop.

Objetivo tecnico:
- remover regras de firewall permissivas;
- garantir bastion sem IP publico;
- permitir SSH apenas via IAP para o bastion;
- permitir SSH no juice-shop apenas a partir da subnet de gerenciamento;
- manter apenas HTTP publico no juice-shop.

---

## Pre-requisitos e Variaveis

### Conceito
Padronizar variaveis reduz erro de digitacao e facilita reaproveitamento de comandos.

### Passo 1: Definir variaveis de ambiente

```bash
# Projeto e localizacao
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="us-east4"
export ZONE="us-east4-c"

# Rede/subnet conforme enunciado
export NETWORK="acme-vpc"
export NETWORK_URI="$(gcloud compute networks describe "$NETWORK" --format='value(selfLink)')"
export MGMT_SUBNET="acme-mgmt-subnet"

# Instancias esperadas no lab
export BASTION_NAME="bastion"
export APP_NAME="juice-shop"

# Tags exigidas no desafio
export TAG_SSH_IAP="grant-ssh-iap-ingress-ql-277"
export TAG_HTTP="grant-http-ingress-ql-277"
export TAG_SSH_INTERNAL="grant-ssh-internal-ingress-ql-277"

# Regras de firewall
export FW_IAP_SSH="allow-ssh-iap-bastion"
export FW_HTTP_PUBLIC="allow-http"
export FW_SSH_INTERNAL="allow-ssh-from-mgmt-subnet"

# Range oficial do IAP TCP forwarding
export IAP_RANGE="35.235.240.0/20"
```

### Passo 2: Configurar defaults no gcloud

```bash
gcloud config set project "$PROJECT_ID"
gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"
```

### Passo 3: Descobrir CIDR da subnet de gerenciamento

```bash
export MGMT_SUBNET_CIDR="$(gcloud compute networks subnets describe "$MGMT_SUBNET" \
	--region="$REGION" \
	--format='value(ipCidrRange)')"

echo "$MGMT_SUBNET_CIDR"
```

**Por que?**
- O enunciado pede SSH no app apenas a partir da rede da subnet `acme-mgmt-subnet`.
- O CIDR real pode variar por ambiente/lab.

---

## TAREFA 1: Revisar e remover regras permissivas

### Conceito
Regras amplas (ex.: `0.0.0.0/0` para SSH) aumentam superficie de ataque e podem reprovar na validacao automatica.

### Passo 1: Listar regras candidatas (SSH aberto para internet)

```bash
# O comando abaixo não funcionou (um dia verificar se necessário)
gcloud compute firewall-rules list \
	--filter="network:$NETWORK AND sourceRanges:0.0.0.0/0 AND allowed.tcp:22" \
	--format="table(name,network,direction,sourceRanges.list():label=SOURCE,allowed[].map().firewall_rule().list():label=ALLOW,targetTags.list():label=TARGET_TAGS)"
```

### Passo 2: Remover regras permissivas identificadas

```bash
# Exemplo: substitua pelos nomes retornados no comando anterior
gcloud compute firewall-rules delete <REGRA_PERMISSIVA_1> <REGRA_PERMISSIVA_2> --quiet

gcloud compute firewall-rules delete open-access --quiet
```

**Explicacao dos parametros:**
- `--filter`: restringe a busca para regras com SSH publico.
- `--quiet`: evita prompt interativo de confirmacao.

**Resultado esperado:**
- Nenhuma regra de SSH com origem `0.0.0.0/0` permanece ativa para o ambiente do desafio.

---

## TAREFA 2: Iniciar e endurecer o bastion

### Conceito
O bastion deve ser a unica porta de entrada administrativa e sem IP externo, usando IAP como canal seguro.

### Passo 1: Iniciar a instancia bastion

```bash
gcloud compute instances start "$BASTION_NAME" --zone="$ZONE"
```

### Passo 2: Garantir que o bastion nao tenha IP publico

```bash
# Descobre nome da access config (quando existir)
export BASTION_ACCESS_CFG="$(gcloud compute instances describe "$BASTION_NAME" \
	--zone="$ZONE" \
	--format='value(networkInterfaces[0].accessConfigs[0].name)')"

# Remove o IP externo caso exista
if [ -n "$BASTION_ACCESS_CFG" ]; then
	gcloud compute instances delete-access-config "$BASTION_NAME" \
		--zone="$ZONE" \
		--access-config-name="$BASTION_ACCESS_CFG"
fi
```

### Passo 3: Adicionar tag de SSH via IAP no bastion

```bash
gcloud compute instances add-tags "$BASTION_NAME" \
	--zone="$ZONE" \
	--tags="$TAG_SSH_IAP"
```

### Passo 4: Criar regra de SSH via IAP para bastion

```bash
gcloud compute firewall-rules create "$FW_IAP_SSH" \
	--network="$NETWORK" \
	--direction=INGRESS \
	--action=ALLOW \
	--rules=tcp:22 \
	--source-ranges="$IAP_RANGE" \
	--target-tags="$TAG_SSH_IAP"
```

**Explicacao dos parametros:**
- `--source-ranges=35.235.240.0/20`: faixa oficial do IAP TCP forwarding.
- `--target-tags`: limita a regra apenas ao bastion com a tag definida.

**Resultado esperado:**
- Bastion acessivel por SSH somente por IAP.
- Bastion sem IP externo.

---

## TAREFA 3: Expor HTTP publico no juice-shop

### Conceito
A aplicacao deve receber trafego HTTP da internet, mas sem abrir SSH publico.

### Passo 1: Adicionar tag HTTP na instancia juice-shop

```bash
gcloud compute instances add-tags "$APP_NAME" \
	--zone="$ZONE" \
	--tags="$TAG_HTTP"
```

### Passo 2: Criar regra de HTTP publico para juice-shop

```bash
gcloud compute firewall-rules create "$FW_HTTP_PUBLIC" \
	--network="$NETWORK" \
	--direction=INGRESS \
	--action=ALLOW \
	--rules=tcp:80 \
	--source-ranges="0.0.0.0/0" \
	--target-tags="$TAG_HTTP"
```

**Explicacao dos parametros:**
- `--rules=tcp:80`: abre apenas HTTP.
- `--source-ranges=0.0.0.0/0`: publica somente a porta web.
- `--target-tags`: aplica apenas ao juice-shop marcado com tag HTTP.

**Resultado esperado:**
- Porta 80 publica no juice-shop.
- Sem necessidade de SSH publico para a VM de aplicacao.

---

## TAREFA 4: Permitir SSH no juice-shop apenas pela rede de gerenciamento

### Conceito
Administracao da VM de app deve ocorrer via bastion, usando origem interna controlada pela subnet de gerenciamento.

### Passo 1: Adicionar tag de SSH interno no juice-shop

```bash
gcloud compute instances add-tags "$APP_NAME" \
	--zone="$ZONE" \
	--tags="$TAG_SSH_INTERNAL"
```

### Passo 2: Criar regra de SSH interno para juice-shop

```bash
gcloud compute firewall-rules create "$FW_SSH_INTERNAL" \
	--network="$NETWORK" \
	--direction=INGRESS \
	--action=ALLOW \
	--rules=tcp:22 \
	--source-ranges="$MGMT_SUBNET_CIDR" \
	--target-tags="$TAG_SSH_INTERNAL"
```

**Explicacao dos parametros:**
- `--source-ranges=$MGMT_SUBNET_CIDR`: restringe SSH a rede da `acme-mgmt-subnet`.
- `--target-tags=$TAG_SSH_INTERNAL`: limita a abertura SSH apenas para a VM de app marcada.

**Resultado esperado:**
- SSH em juice-shop permitido somente para origem da subnet de gerenciamento.

---

## TAREFA 5: Conectar por IAP ao bastion e depois ao juice-shop

### Conceito
Fluxo seguro de administracao: operador -> IAP -> bastion -> juice-shop.

### Passo 1: Conectar ao bastion via IAP

```bash
gcloud compute ssh "$BASTION_NAME" \
	--zone="$ZONE" \
	--tunnel-through-iap
```

**Explicacao dos parametros:**
- `--tunnel-through-iap`: Estabelece túnel SSH via Identity-Aware Proxy, permitindo conexão sem IP público.

Se houver falha de tunel:

```bash
gcloud compute ssh "$BASTION_NAME" \
	--zone="$ZONE" \
	--tunnel-through-iap \
	--troubleshoot
```

### Passo 2: Do bastion, conectar ao juice-shop

```bash
# dentro do bastion
export PROJECT_ID="qwiklabs-gcp-01-dbf3f8716ba2"
export ZONE="us-east4-c"
export APP_NAME="juice-shop"

gcloud compute ssh "$APP_NAME" \
  --zone="$ZONE" \
  --project="$PROJECT_ID" \
  --internal-ip
```

**Explicacao dos parametros:**
- `--internal-ip`: Força a conexão SSH utilizando apenas o IP interno da instância, sem necessidade de IP público. Útil dentro de um bastion ou rede privada.

**O que e um bastion (simples):**
Um bastion (ou bastião) é originalmente um termo de arquitetura militar referente a uma fortificação saliente projetada para proteger muralhas, mas hoje o termo é amplamente utilizado na área de Tecnologia da Informação (TI) para designar um ponto de acesso seguro (servidor ou gateway) para redes privadas.1. Na Tecnologia da Informação (TI). Em redes de computadores, um bastion host (ou jump box) é um servidor especialmente projetado e protegido contra ataques. Ele atua como uma porta de entrada controlada, permitindo que administradores acessem recursos internos ou máquinas virtuais (VMs) em redes privadas ou em nuvem sem expô-los diretamente à internet

Alternativa sem gcloud dentro do bastion (usando IP interno):

```bash
# dentro do bastion
APP_INTERNAL_IP="$(gcloud compute instances describe "$APP_NAME" --zone="$ZONE" --format='value(networkInterfaces[0].networkIP)')"
ssh "$USER@$APP_INTERNAL_IP"
```

**Resultado esperado:**
- Acesso SSH ao bastion apenas via IAP.
- Acesso SSH ao juice-shop realizado a partir do bastion.

---

## Validacao e Testes

### 1) Confirmar bastion sem IP externo

```bash
gcloud compute instances describe "$BASTION_NAME" \
	--zone="$ZONE" \
	--format="get(networkInterfaces[0].accessConfigs)"
```

Esperado: vazio/nulo.

### 2) Confirmar regras de firewall relevantes

```bash
gcloud compute firewall-rules list \
	--filter="name=($FW_IAP_SSH OR $FW_HTTP_PUBLIC OR $FW_SSH_INTERNAL)" \
	--format="table(name,direction,sourceRanges.list():label=SOURCE,allowed[].map().firewall_rule().list():label=ALLOW,targetTags.list():label=TARGET_TAGS)"
```

### 3) Confirmar tags nas instancias

```bash
gcloud compute instances describe "$BASTION_NAME" --zone="$ZONE" \
	--format="get(tags.items)"

gcloud compute instances describe "$APP_NAME" --zone="$ZONE" \
	--format="get(tags.items)"
```

### 4) Confirmar HTTP publico no juice-shop

```bash
APP_EXTERNAL_IP="$(gcloud compute instances describe "$APP_NAME" --zone="$ZONE" --format='value(networkInterfaces[0].accessConfigs[0].natIP)')"
curl -I "http://$APP_EXTERNAL_IP"
```

Esperado: resposta HTTP (ex.: `HTTP/1.1 200 OK`).

### 5) Confirmar fluxo SSH esperado

```bash
# Local -> bastion via IAP
gcloud compute ssh "$BASTION_NAME" --zone="$ZONE" --tunnel-through-iap --command='hostname'

# Opcional: validar reachability do app a partir do bastion
gcloud compute ssh "$BASTION_NAME" --zone="$ZONE" --tunnel-through-iap --command="nc -zv $APP_NAME 22"
```

---

## Troubleshooting

### Erro ao usar IAP

```bash
gcloud compute ssh "$BASTION_NAME" --zone="$ZONE" --tunnel-through-iap --troubleshoot
```

Verifique:
- API do IAP habilitada.
- Regra de firewall com origem `35.235.240.0/20` e tag correta no bastion.
- Permissao IAM para usar IAP TCP tunneling no projeto.

### SSH no juice-shop nao conecta a partir do bastion

Comandos uteis:

```bash
gcloud compute firewall-rules describe "$FW_SSH_INTERNAL"
gcloud compute instances describe "$APP_NAME" --zone="$ZONE" --format='yaml(tags,networkInterfaces)'
gcloud compute networks subnets describe "$MGMT_SUBNET" --region="$REGION" --format='value(ipCidrRange)'
```

Verifique:
- tag `$TAG_SSH_INTERNAL` aplicada no juice-shop.
- `source-ranges` igual ao CIDR real da `acme-mgmt-subnet`.
- ausencia de regra conflitante de deny.

---

## Limpeza (Opcional)

```bash
gcloud compute firewall-rules delete "$FW_IAP_SSH" "$FW_HTTP_PUBLIC" "$FW_SSH_INTERNAL" --quiet
```

Se quiser remover tags adicionadas durante o desafio:

```bash
gcloud compute instances remove-tags "$BASTION_NAME" --zone="$ZONE" --tags="$TAG_SSH_IAP"
gcloud compute instances remove-tags "$APP_NAME" --zone="$ZONE" --tags="$TAG_HTTP,$TAG_SSH_INTERNAL"
```

---

## Conceitos-Chave para Relembrar

| Conceito | O que e | Quando usar |
|---|---|---|
| IAP TCP Forwarding | Tunel seguro sem IP publico na VM | Acesso administrativo via identidade IAM |
| Bastion Host | Ponto unico de administracao | Reduz exposicao direta de VMs internas |
| Network Tag | Alvo logico de regras de firewall | Aplicar acesso minimo por funcao da VM |
| Firewall Rule (Ingress) | Controle de entrada por origem/porta | Enforcar principio do menor privilegio |
| Subnet CIDR Restrito | Origem interna controlada | SSH interno somente por rede de gestao |

---

## Fluxo Final de Acesso

1. Operador conecta no bastion via IAP (`35.235.240.0/20` + IAM).
2. Bastion, sem IP publico, recebe SSH administrativo de forma controlada.
3. Do bastion, operador acessa juice-shop por SSH interno.
4. Juice-shop expone apenas HTTP (porta 80) para a internet.
5. Regras permissivas antigas de SSH publico sao removidas.
