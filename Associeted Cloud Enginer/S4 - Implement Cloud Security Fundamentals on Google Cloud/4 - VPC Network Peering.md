# VPC Network Peering

## Visao Geral

O Google Cloud Virtual Private Cloud (VPC) Network Peering permite conectividade privada entre duas redes VPC, independentemente de estarem no mesmo projeto ou na mesma organizacao.

Com o VPC Network Peering, voce pode construir ecossistemas SaaS (Software as a Service) no Google Cloud, disponibilizando servicos de forma privada entre diferentes VPCs, dentro da mesma organizacao ou entre organizacoes distintas, permitindo que workloads se comuniquem em rede privada.

O VPC Network Peering e util para:

- Organizacoes com varios dominios administrativos de rede.
- Organizacoes que precisam se conectar com outras organizacoes.
- Ambientes com separacao por unidades de negocio, fusoes ou aquisicoes, onde ha multiplos nos organizacionais.

Se voce possui varios dominios administrativos dentro da empresa, o peering permite expor servicos entre VPCs sem usar internet publica. Se voce oferece servicos para outras organizacoes, tambem pode publica-los de forma privada para esses clientes.

Em comparacao com IP externo ou VPN, o VPC Network Peering traz vantagens como:

- Network Latency: menor latencia por usar rede privada em vez de trafego publico.
- Network Security: os servicos nao precisam ficar expostos na internet publica.
- Network Cost: comunicacao por IP interno entre redes pareadas, com potencial reducao de custos de egress. O pricing de rede padrao continua valendo para o trafego.

## Introducao

Este guia converte o desafio para uma execucao 100% via CLI com `gcloud`, mantendo os nomes de recursos exigidos no enunciado.

Objetivo do lab:
- criar duas VPCs customizadas em projetos diferentes;
- criar sub-redes, VMs e firewall em cada projeto;
- configurar peering bidirecional entre `network-a` e `network-b`;
- validar troca de rotas e conectividade entre VMs.

---

## Pre-requisitos e Variaveis

### Conceito
Como o desafio usa dois projetos, o fluxo mais seguro e reproduzivel e trabalhar com dois contextos de Cloud Shell (ou alternar projeto no mesmo shell com cuidado).

### Variaveis base (ajuste conforme ambiente)

```bash
# IDs fornecidos no desafio
export PROJECT_A="qwiklabs-gcp-01-d2c52a1ef9ea"
export PROJECT_B="qwiklabs-gcp-00-05ffcf4d9498"

# Regiao/zona usadas no enunciado
export REGION="us-east1"
export ZONE="us-east1-c"

# Recursos exigidos pelo lab
export NETWORK_A="network-a"
export SUBNET_A="network-a-subnet"
export SUBNET_A_RANGE="10.0.0.0/16"
export VM_A="vm-a"
export FW_A="network-a-fw"

export NETWORK_B="network-b"
export SUBNET_B="network-b-subnet"
export SUBNET_B_RANGE="10.8.0.0/16"
export VM_B="vm-b"
export FW_B="network-b-fw"

# Peerings exigidos pelo lab
export PEER_AB="peer-ab"
export PEER_BA="peer-ba"
```

### Como executar com dois shells

- Shell A: mantenha `PROJECT_A` como contexto padrao.
- Shell B: mantenha `PROJECT_B` como contexto padrao.

Se optar por um unico shell, execute sempre `gcloud config set project ...` antes de cada bloco.

---

## TAREFA 1: Criar rede customizada em ambos os projetos

### Conceito
Cada projeto tera uma VPC custom (`--subnet-mode=custom`) para controle explicito de CIDR e segmentacao. Em seguida, criamos VM e regra de firewall para SSH e ICMP (necessarios para teste de conectividade).

### Passos CLI - Projeto A

```bash
gcloud config set project "$PROJECT_A"

gcloud compute networks create "$NETWORK_A" \
  --subnet-mode=custom

gcloud compute networks subnets create "$SUBNET_A" \
  --network="$NETWORK_A" \
  --range="$SUBNET_A_RANGE" \
  --region="$REGION"

gcloud compute instances create "$VM_A" \
  --zone="$ZONE" \
  --network="$NETWORK_A" \
  --subnet="$SUBNET_A" \
  --machine-type=e2-small

gcloud compute firewall-rules create "$FW_A" \
  --network="$NETWORK_A" \
  --allow=tcp:22,icmp
```

### Explicacao dos parametros principais

- `--subnet-mode=custom`: desabilita sub-redes automaticas e exige criacao manual.
- `--range`: define CIDR da sub-rede.
- `--zone`: zona da VM.
- `--allow=tcp:22,icmp`: libera SSH e ping para validacao entre VMs.

### Resultado esperado

- VPC `network-a` criada no `PROJECT_A`.
- Sub-rede `network-a-subnet` com CIDR `10.0.0.0/16`.
- VM `vm-a` criada e em estado `RUNNING`.
- Firewall `network-a-fw` ativo.

### Passos CLI - Projeto B

```bash
gcloud config set project "$PROJECT_B"

gcloud compute networks create "$NETWORK_B" \
  --subnet-mode=custom

gcloud compute networks subnets create "$SUBNET_B" \
  --network="$NETWORK_B" \
  --range="$SUBNET_B_RANGE" \
  --region="$REGION"

gcloud compute instances create "$VM_B" \
  --zone="$ZONE" \
  --network="$NETWORK_B" \
  --subnet="$SUBNET_B" \
  --machine-type=e2-small

gcloud compute firewall-rules create "$FW_B" \
  --network="$NETWORK_B" \
  --allow=tcp:22,icmp
```

### Explicacao dos parametros principais

- Mesmo racional do Projeto A, mudando apenas nomes e CIDR (`10.8.0.0/16`).

### Resultado esperado

- VPC `network-b` criada no `PROJECT_B`.
- Sub-rede `network-b-subnet` com CIDR `10.8.0.0/16`.
- VM `vm-b` criada e em estado `RUNNING`.
- Firewall `network-b-fw` ativo.

---

## TAREFA 2: Configurar sessao de VPC Network Peering

### Conceito
Peering de VPC e sempre configurado dos dois lados. Um lado sozinho fica `INACTIVE` ate o par correspondente ser criado no outro projeto.

### Passos CLI - Criar peering de A para B

```bash
gcloud config set project "$PROJECT_A"

gcloud compute networks peerings create "$PEER_AB" \
  --network="$NETWORK_A" \
  --peer-project="$PROJECT_B" \
  --peer-network="$NETWORK_B"
```

### Passos CLI - Criar peering de B para A

```bash
gcloud config set project "$PROJECT_B"

gcloud compute networks peerings create "$PEER_BA" \
  --network="$NETWORK_B" \
  --peer-project="$PROJECT_A" \
  --peer-network="$NETWORK_A"
```

### Explicacao dos parametros principais

- `--network`: VPC local onde o peering sera anexado.
- `--peer-project`: projeto remoto (lado par).
- `--peer-network`: nome da VPC remota.

### Resultado esperado

- `peer-ab` visivel em `network-a`.
- `peer-ba` visivel em `network-b`.
- Estado de peering `ACTIVE` nos dois lados.
- Rotas implicitas para CIDRs remotos aparecem nas tabelas de rota.

---

## TAREFA 3: Testar conectividade entre VMs

### Conceito
Com peering ativo e firewall liberando ICMP, VMs em VPCs pareadas devem se comunicar por IP interno.

### Passo 1: Obter IP interno da vm-a

```bash
gcloud compute instances describe "$VM_A" \
  --project="$PROJECT_A" \
  --zone="$ZONE" \
  --format='get(networkInterfaces[0].networkIP)'
```

Guarde o valor retornado em `VM_A_INTERNAL_IP`.

### Passo 2: Testar ping a partir da vm-b

Opcao 1: SSH interativo e ping manual.

```bash
gcloud compute ssh "$VM_B" \
  --project="$PROJECT_B" \
  --zone="$ZONE"

# Dentro da VM_B:
ping -c 5 <VM_A_INTERNAL_IP>
```

Opcao 2: comando unico sem abrir shell interativo.

```bash
gcloud compute ssh "$VM_B" \
  --project="$PROJECT_B" \
  --zone="$ZONE" \
  --command="ping -c 5 <VM_A_INTERNAL_IP>"
```

### Resultado esperado

- `5 packets transmitted, 5 received`.
- `0% packet loss`.
- RTT com latencia baixa (ambiente/regiao dependente).

---

## Validacao

Execute os comandos abaixo para validar cada etapa de forma objetiva.

### Validacao de redes e sub-redes

```bash
gcloud compute networks list --project="$PROJECT_A" --filter="name=$NETWORK_A"
gcloud compute networks subnets list --project="$PROJECT_A" --filter="name=$SUBNET_A"

gcloud compute networks list --project="$PROJECT_B" --filter="name=$NETWORK_B"
gcloud compute networks subnets list --project="$PROJECT_B" --filter="name=$SUBNET_B"
```

### Validacao de VMs

```bash
gcloud compute instances list --project="$PROJECT_A" --filter="name=$VM_A"
gcloud compute instances list --project="$PROJECT_B" --filter="name=$VM_B"
```

### Validacao de peering

```bash
gcloud compute networks peerings list \
  --project="$PROJECT_A" \
  --network="$NETWORK_A"

gcloud compute networks peerings list \
  --project="$PROJECT_B" \
  --network="$NETWORK_B"
```

Verifique o campo `STATE` como `ACTIVE`.

### Validacao de rotas implicitas

```bash
gcloud compute routes list --project="$PROJECT_A" \
  --filter="network:$NETWORK_A"

gcloud compute routes list --project="$PROJECT_B" \
  --filter="network:$NETWORK_B"
```

Esperado:
- no `PROJECT_A`, rota de peering para `10.8.0.0/16` via `peer-ab`;
- no `PROJECT_B`, rota de peering para `10.0.0.0/16` via `peer-ba`.

---

## Troubleshooting

### Peering ficou INACTIVE

Possiveis causas:
- peering criado apenas em um lado;
- nomes de VPC/projeto remoto incorretos;
- contexto de projeto errado no shell.

Comandos de checagem:

```bash
gcloud config get-value project

gcloud compute networks peerings list \
  --project="$PROJECT_A" \
  --network="$NETWORK_A"

gcloud compute networks peerings list \
  --project="$PROJECT_B" \
  --network="$NETWORK_B"
```

### Ping falhou

Possiveis causas:
- firewall ICMP ausente;
- uso de IP externo em vez de IP interno;
- VM em estado diferente de `RUNNING`.

Comandos de checagem:

```bash
gcloud compute firewall-rules list --project="$PROJECT_A" --filter="name=$FW_A"
gcloud compute firewall-rules list --project="$PROJECT_B" --filter="name=$FW_B"

gcloud compute instances list --project="$PROJECT_A" --filter="name=$VM_A"
gcloud compute instances list --project="$PROJECT_B" --filter="name=$VM_B"
```

---

## Limpeza (opcional)

Se quiser remover todos os recursos apos o teste:

```bash
# Projeto B
gcloud compute instances delete "$VM_B" --project="$PROJECT_B" --zone="$ZONE" --quiet
gcloud compute networks peerings delete "$PEER_BA" --project="$PROJECT_B" --network="$NETWORK_B" --quiet
gcloud compute firewall-rules delete "$FW_B" --project="$PROJECT_B" --quiet
gcloud compute networks subnets delete "$SUBNET_B" --project="$PROJECT_B" --region="$REGION" --quiet
gcloud compute networks delete "$NETWORK_B" --project="$PROJECT_B" --quiet

# Projeto A
gcloud compute instances delete "$VM_A" --project="$PROJECT_A" --zone="$ZONE" --quiet
gcloud compute networks peerings delete "$PEER_AB" --project="$PROJECT_A" --network="$NETWORK_A" --quiet
gcloud compute firewall-rules delete "$FW_A" --project="$PROJECT_A" --quiet
gcloud compute networks subnets delete "$SUBNET_A" --project="$PROJECT_A" --region="$REGION" --quiet
gcloud compute networks delete "$NETWORK_A" --project="$PROJECT_A" --quiet
```

---

## Conceitos-chave

- VPC custom: maior controle de segmentacao e CIDR.
- VPC Network Peering: conectividade privada entre VPCs sem VPN.
- Peering bidirecional: precisa existir configuracao em ambos os lados.
- Rotas implicitas de peering: aparecem automaticamente apos estado `ACTIVE`.
- Validacao operacional: `list`, `describe`, `peerings list`, `routes list` e `ping` comprovam funcionamento ponta a ponta.

---

## Fluxo Final

1. Definir variaveis e contexto de projeto.
2. Criar VPC, sub-rede, VM e firewall no Projeto A.
3. Repetir no Projeto B.
4. Criar peering `peer-ab` (A -> B).
5. Criar peering `peer-ba` (B -> A).
6. Validar estado `ACTIVE` e rotas de peering.
7. Testar conectividade ICMP entre `vm-b` e IP interno da `vm-a`.
