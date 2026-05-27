# Multiple VPC Networks no GCP

## Introducao

Este documento organiza, em formato de laboratorio, a criacao e validacao de multiplas redes VPC no Google Cloud:

- `managementnet` (custom mode)
- `privatenet` (custom mode)
- `mynetwork` (auto mode, ja existente no lab)

Tambem mostra como testar conectividade entre redes e como uma VM com multiplas interfaces de rede (multi-NIC) atua como ponto de interligacao.

---

## TAREFA 1: Criar redes VPC custom e sub-redes

### Conceito

Em redes **custom mode**, voce cria manualmente cada subnet e controla exatamente os ranges IP por regiao.

### Passo 1: Criar a VPC de gerenciamento

```bash
gcloud compute networks create managementnet \
    --project=qwiklabs-gcp-00-93573252291f \
    --subnet-mode=custom \
    --bgp-routing-mode=regional \
    --bgp-best-path-selection-mode=legacy
```

**Explicacao dos parametros:**

- `--project`: projeto onde a rede sera criada
- `--subnet-mode=custom`: impede criacao automatica de subnets
- `--bgp-routing-mode=regional`: troca de rotas dinamicas restrita a regiao
- `--bgp-best-path-selection-mode=legacy`: mantem o comportamento padrao do lab para selecao de rotas

**Resultado esperado:** rede `managementnet` criada sem subnets, pronta para receber configuracao manual.

### Passo 2: Criar subnet da rede de gerenciamento

```bash
gcloud compute networks subnets create managementsubnet-1 \
    --project=qwiklabs-gcp-00-93573252291f \
    --range=10.130.0.0/20 \
    --stack-type=IPV4_ONLY \
    --network=managementnet \
    --region=us-east1
```

**Explicacao dos parametros:**

- `--range=10.130.0.0/20`: bloco CIDR da subnet
- `--stack-type=IPV4_ONLY`: subnet apenas com IPv4
- `--network=managementnet`: associa a subnet a VPC correta
- `--region=us-east1`: cria a subnet na regiao desejada

**Resultado esperado:** subnet `managementsubnet-1` criada dentro da rede `managementnet`.

### Passo 3: Criar a VPC privada

```bash
gcloud compute networks create privatenet \
    --project=qwiklabs-gcp-00-93573252291f \
    --subnet-mode=custom
```

**Resultado esperado:** rede `privatenet` criada em modo custom, sem subnets automaticas.

### Passo 4: Criar subnets da VPC privada

```bash
gcloud compute networks subnets create privatesubnet-1 \
    --project=qwiklabs-gcp-00-93573252291f \
    --network=privatenet \
    --region=us-east1 \
    --range=172.16.0.0/24

gcloud compute networks subnets create privatesubnet-2 \
    --project=qwiklabs-gcp-00-93573252291f \
    --network=privatenet \
    --region=asia-southeast1 \
    --range=172.20.0.0/20
```

**Explicacao dos parametros:**

- `--network=privatenet`: define a VPC dona das subnets
- `--region`: indica em qual regiao cada subnet sera criada
- `--range`: define o bloco IP interno de cada subnet

**Resultado esperado:**

- `privatesubnet-1` criada em `us-east1`
- `privatesubnet-2` criada em `asia-southeast1`

### Passo 5: Validar redes e sub-redes

```bash
gcloud compute networks list
gcloud compute networks subnets list --sort-by=NETWORK
```

**Saida esperada para `gcloud compute networks list`:**

```bash
NAME: default
SUBNET_MODE: AUTO
BGP_ROUTING_MODE: REGIONAL

NAME: managementnet
SUBNET_MODE: CUSTOM
BGP_ROUTING_MODE: REGIONAL

NAME: mynetwork
SUBNET_MODE: AUTO
BGP_ROUTING_MODE: REGIONAL

NAME: privatenet
SUBNET_MODE: CUSTOM
BGP_ROUTING_MODE: REGIONAL
```

**Saida esperada para `gcloud compute networks subnets list --sort-by=NETWORK`:**

```bash
NAME: default
REGION: us-central1
NETWORK: default
RANGE: 10.128.0.0/20

NAME: managementsubnet-1
REGION: us-east1
NETWORK: managementnet
RANGE: 10.130.0.0/20

NAME: privatesubnet-1
REGION: us-east1
NETWORK: privatenet
RANGE: 172.16.0.0/24

NAME: privatesubnet-2
REGION: asia-southeast1
NETWORK: privatenet
RANGE: 172.20.0.0/20
```

**Resultado esperado (resumo):**

- `default` e `mynetwork` em modo `AUTO`
- `managementnet` e `privatenet` em modo `CUSTOM`

**Diferenca pratica:**

- **AUTO mode:** cria subnets automaticamente em todas as regioes
- **CUSTOM mode:** nao cria subnets automaticamente; voce define tudo manualmente

### Resumo Visual - Tarefa 1

```text
[VPCs no projeto]
   |- default (AUTO)
   |- mynetwork (AUTO)
   |- managementnet (CUSTOM)
   |    \- managementsubnet-1 (10.130.0.0/20 - us-east1)
   \- privatenet (CUSTOM)
        |- privatesubnet-1 (172.16.0.0/24 - us-east1)
        \- privatesubnet-2 (172.20.0.0/20 - asia-southeast1)
```

---

## TAREFA 2: Criar regras de firewall para acesso basico

### Conceito

Sem regras de firewall, testes de conectividade (ICMP/ping, SSH, RDP) podem falhar, mesmo com VM ativa.

### Passo 1: Liberar ICMP, SSH e RDP na managementnet

```bash
gcloud compute firewall-rules create managementnet-allow-icmp-ssh-rdp \
    --project=qwiklabs-gcp-00-93573252291f \
    --direction=INGRESS \
    --priority=1000 \
    --network=managementnet \
    --action=ALLOW \
    --rules=tcp:22,tcp:3389,icmp \
    --source-ranges=0.0.0.0/0
```

**Explicacao dos parametros:**

- `--direction=INGRESS`: regra para trafego de entrada
- `--network=managementnet`: aplica a regra somente nessa VPC
- `--rules=tcp:22,tcp:3389,icmp`: libera SSH, RDP e ping
- `--source-ranges=0.0.0.0/0`: permite trafego vindo de qualquer origem

**Resultado esperado:** instancias na `managementnet` passam a aceitar ping, SSH e RDP.

### Passo 2: Liberar ICMP, SSH e RDP na privatenet

```bash
gcloud compute firewall-rules create privatenet-allow-icmp-ssh-rdp \
    --project=qwiklabs-gcp-00-93573252291f \
    --direction=INGRESS \
    --priority=1000 \
    --network=privatenet \
    --action=ALLOW \
    --rules=icmp,tcp:22,tcp:3389 \
    --source-ranges=0.0.0.0/0
```

**Resultado esperado:** instancias na `privatenet` passam a aceitar ping, SSH e RDP.

### Passo 3: Validar regras criadas

```bash
gcloud compute firewall-rules list --sort-by=NETWORK
```

**Saida esperada:**

```bash
NAME: privatenet-allow-icmp-ssh-rdp
NETWORK: privatenet
DIRECTION: INGRESS
PRIORITY: 1000
ALLOW: icmp,tcp:22,tcp:3389
DISABLED: False

NAME: managementnet-allow-icmp-ssh-rdp
NETWORK: managementnet
DIRECTION: INGRESS
PRIORITY: 1000
ALLOW: tcp:22,tcp:3389,icmp
DISABLED: False
```

---

## TAREFA 3: Criar VMs e validar conectividade inicial

### Conceito

Nesta etapa, cada VM fica conectada apenas na sua propria rede. Sem peering/roteamento adicional, redes diferentes nao se enxergam por IP interno.

### Passo 1: Criar VM na managementnet

```bash
gcloud compute instances create managementnet-vm-1 \
    --zone=us-east1-c \
    --machine-type=e2-micro \
    --subnet=managementsubnet-1
```

**Explicacao dos parametros:**

- `--zone=us-east1-c`: zona onde a VM sera criada
- `--machine-type=e2-micro`: tipo de maquina usado no lab
- `--subnet=managementsubnet-1`: conecta a VM na subnet de gerenciamento

**Resultado esperado:** VM `managementnet-vm-1` criada com IP interno da faixa `10.130.0.0/20`.

### Passo 2: Criar VM na privatenet

```bash
gcloud compute instances create privatenet-vm-1 \
    --zone=us-east1-c \
    --machine-type=e2-micro \
    --subnet=privatesubnet-1
```

**Resultado esperado:** VM `privatenet-vm-1` criada com IP interno da faixa `172.16.0.0/24`.

### Passo 3: Listar VMs

```bash
gcloud compute instances list --sort-by=ZONE
```

**Saida esperada:**

```bash
NAME: managementnet-vm-1
ZONE: us-east1-c
MACHINE_TYPE: e2-micro
INTERNAL_IP: 10.130.0.2
EXTERNAL_IP: 34.23.6.53
STATUS: RUNNING

NAME: privatenet-vm-1
ZONE: us-east1-c
MACHINE_TYPE: e2-micro
INTERNAL_IP: 172.16.0.2
EXTERNAL_IP: 34.23.73.76
STATUS: RUNNING
```

**Mapa de exemplo das VMs usadas no teste:**

| VM Name | IP Externo | Network | IP Interno |
|---------|------------|---------|------------|
| managementnet-vm-1 | 34.23.6.53 | managementnet | 10.130.0.2 |
| mynet-vm-1 | 34.138.15.97 | mynetwork | 10.142.0.2 |
| mynet-vm-2 | 34.142.159.173 | mynetwork | 10.148.0.2 |
| privatenet-vm-1 | 34.23.73.76 | privatenet | 172.16.0.2 |

### Passo 4: Testar a partir da mynet-vm-1

**Teste por IP externo (esperado: sucesso):**

```bash
ping -c 3 34.142.159.173
ping -c 3 34.23.6.53
ping -c 3 34.23.73.76
```

**Exemplo de resultado esperado:**

```bash
64 bytes from 34.142.159.173: icmp_seq=1 ttl=115 time=2.31 ms
64 bytes from 34.142.159.173: icmp_seq=2 ttl=115 time=1.98 ms
64 bytes from 34.142.159.173: icmp_seq=3 ttl=115 time=2.12 ms

--- 34.142.159.173 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
```

**Teste por IP interno (esperado):**

- `10.148.0.2` (mynetwork): sucesso
- `10.130.0.2` (managementnet): erro
- `172.16.0.2` (privatenet): erro

```bash
ping -c 3 10.148.0.2
ping -c 3 10.130.0.2
ping -c 3 172.16.0.2
```

**Exemplo de resultado esperado:**

```bash
PING 10.148.0.2 (10.148.0.2) 56(84) bytes of data.
64 bytes from 10.148.0.2: icmp_seq=1 ttl=64 time=0.54 ms
64 bytes from 10.148.0.2: icmp_seq=2 ttl=64 time=0.49 ms
64 bytes from 10.148.0.2: icmp_seq=3 ttl=64 time=0.51 ms

PING 10.130.0.2 (10.130.0.2) 56(84) bytes of data.
From 10.142.0.2 icmp_seq=1 Destination Host Unreachable

PING 172.16.0.2 (172.16.0.2) 56(84) bytes of data.
From 10.142.0.2 icmp_seq=1 Destination Host Unreachable
```

**Por que isso acontece?**

Porque as redes sao separadas e, neste ponto, ainda nao existe uma VM/appliance com multiplas interfaces para rotear trafego entre elas.

---

## TAREFA 4: Criar VM multi-NIC (appliance) e repetir os testes

### Conceito

Uma VM com multiplas interfaces pode estar conectada simultaneamente em redes diferentes, funcionando como ponto de acesso/transito entre sub-redes diretamente conectadas.

### Passo 1: Criar `vm-appliance` com 3 interfaces

```bash
gcloud compute instances create vm-appliance \
    --zone=us-east1-c \
    --machine-type=e2-standard-4 \
    --network-interface=subnet=managementsubnet-1 \
    --network-interface=subnet=privatesubnet-1 \
    --network-interface=subnet=mynetwork
```

**Explicacao dos parametros:**

- `--machine-type=e2-standard-4`: instancia com mais recursos para atuar como appliance
- `--network-interface=subnet=...`: adiciona uma interface de rede por subnet
- a primeira interface informada vira a interface primaria (`nic0`)

**Resultado esperado:** VM `vm-appliance` criada com tres interfaces, cada uma em uma rede diferente.

### Passo 2: Acessar por SSH e validar interfaces/rotas

```bash
sudo ifconfig
ip route
```

**Exemplo de resultado esperado:**

```bash
eth0: inet 10.130.0.x
eth1: inet 172.16.0.x
eth2: inet 10.142.0.x

default via 10.130.0.1 dev eth0
10.130.0.0/20 dev eth0 proto kernel scope link
172.16.0.0/24 dev eth1 proto kernel scope link
10.142.0.0/20 dev eth2 proto kernel scope link
```

### Passo 3: Testar DNS interno

```bash
ping -c 3 privatenet-vm-1
```

**Observacao:**

No GCP, o DNS interno resolve hostname da instancia para a interface primaria (`nic0`).

**Resultado esperado:** o hostname `privatenet-vm-1` resolve para o IP interno primario da VM e o ping responde com sucesso.

### Passo 4: Repetir testes por IP interno a partir da `vm-appliance`

```bash
ping -c 3 10.148.0.2
ping -c 3 10.130.0.2
ping -c 3 172.16.0.2
```

**Resultado esperado:** sucesso para os tres destinos, pois todos sao subnets diretamente conectadas a interfaces da `vm-appliance`.

**Exemplo de resultado esperado:**

```bash
PING 10.148.0.2 (10.148.0.2) 56(84) bytes of data.
64 bytes from 10.148.0.2: icmp_seq=1 ttl=64 time=0.61 ms

PING 10.130.0.2 (10.130.0.2) 56(84) bytes of data.
64 bytes from 10.130.0.2: icmp_seq=1 ttl=64 time=0.72 ms

PING 172.16.0.2 (172.16.0.2) 56(84) bytes of data.
64 bytes from 172.16.0.2: icmp_seq=1 ttl=64 time=0.68 ms
```

### Observacao importante sobre roteamento padrao

Em instancias com varias NICs:

- cada interface recebe rota da subnet local
- existe uma rota default unica, associada a interface primaria (`eth0`)

Sem configuracao manual de roteamento/policy routing, trafego para destinos nao diretamente conectados tende a sair pela interface primaria, podendo causar falhas.

# mynet-vm-2 : 10.148.0.2
ping -c 3 '10.148.0.2'
ERRO

Note: This does not work! In a multiple interface instance, every interface gets a route for the subnet that it is in. In addition, the instance gets a single default route that is associated with the primary interface eth0. Unless manually configured otherwise, any traffic leaving an instance for any destination other than a directly connected subnet will leave the instance via the default route on eth0.

---

## Comandos uteis de verificacao

### Redes e sub-redes

```bash
gcloud compute networks list
gcloud compute networks subnets list --sort-by=NETWORK
```

### Firewall

```bash
gcloud compute firewall-rules list --sort-by=NETWORK
```

### Instancias

```bash
gcloud compute instances list --sort-by=ZONE
gcloud compute instances describe managementnet-vm-1 --zone=us-east1-c
gcloud compute ssh managementnet-vm-1 --zone=us-east1-c
```

---

## Conceitos-chave para relembrar

| Conceito | O que e | Quando usar |
|----------|---------|-------------|
| **VPC AUTO mode** | Rede com subnets criadas automaticamente por regiao | Labs simples e setup rapido |
| **VPC CUSTOM mode** | Rede sem subnets automaticas | Ambientes com controle total de CIDR e regiao |
| **Firewall Rule** | Regra de entrada/saida de trafego | Sempre que precisar liberar acesso (ICMP/SSH/RDP/HTTP etc.) |
| **Subnet** | Segmento IP dentro da VPC | Organizar ranges por regiao/ambiente |
| **VM Multi-NIC** | Instancia com mais de uma interface de rede | Appliance, roteamento, transito entre redes |
| **DNS interno GCP** | Resolve nome da VM para IP interno da interface primaria | Comunicacao interna por hostname |

---

## Resumo final do fluxo

1. Criar `managementnet` e `privatenet` em modo custom
2. Criar subnets com ranges especificos
3. Liberar ICMP/SSH/RDP via firewall
4. Criar VMs de teste em redes diferentes
5. Validar que IP externo funciona e IP interno entre redes isoladas falha
6. Criar `vm-appliance` com 3 NICs
7. Repetir testes internos para validar conectividade atraves de interfaces diretamente conectadas