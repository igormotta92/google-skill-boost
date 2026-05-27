# Load Balancers no GCP

## Introdução

Este documento explica os conceitos de Load Balancing no GCP e detalha os passos para configurar dois tipos: **Network Load Balancer** e **HTTP Load Balancer**.

### Diferenças entre os tipos:

- **Network Load Balancer (NLB)**: Trabalha na Camada 4 (Transporte). Ideal para tráfego não-HTTP, altíssima performance e baixa latência.
- **HTTP Load Balancer (ALB)**: Trabalha na Camada 7 (Aplicação). Ideal para tráfego HTTP/HTTPS, permite roteamento baseado em URL e host.



---

## TAREFA 1: Criar Múltiplas Instâncias de Servidor Web

### Conceito
Precisamos de múltiplas instâncias (máquinas virtuais) executando o mesmo serviço (Apache) para que o load balancer possa distribuir o tráfego entre elas.

### Passo 1: Configurar Região e Zona Padrão

```bash
gcloud config set compute/region asia-south1
gcloud config set compute/zone asia-south1-a
```

**Por quê?**
- Define a região/zona padrão para todos os comandos subsequentes, evitando repetir `--region` e `--zone` em cada comando.

### Passo 2: Criar Instância de Servidor Web (web1, web2, web3)

```bash
# gcloud compute instances create web3 \
# gcloud compute instances create web2 \
gcloud compute instances create web1 \
    --zone=asia-south1-a \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
        apt-get update
        apt-get install apache2 -y
        service apache2 restart
        echo "<h3>Web Server: web-number web1</h3>" | tee /var/www/html/index.html'
```

**Explicação dos parâmetros:**
- `--zone=asia-south1-a`: Localização da VM
- `--tags=network-lb-tag`: Tag para identificar e agrupar instâncias (usada em firewall rules e load balancers)
- `--machine-type=e2-small`: Tipo de máquina (processamento e memória)
- `--image-family=debian-12`: Sistema operacional base
- `--metadata=startup-script=...`: Script que roda na primeira inicialização (instala Apache e cria página HTML)

**Resultado esperado:** Instância criada com Apache rodando, acessível via HTTP na porta 80.

### Passo 3: Verificar Instâncias Criadas

```bash
gcloud compute instances list
```

**Saída esperada:**
```
NAME: web1
ZONE: asia-south1-a
MACHINE_TYPE: e2-small
INTERNAL_IP: 10.160.0.5
EXTERNAL_IP: 34.14.162.101
STATUS: RUNNING

NAME: web2
ZONE: asia-south1-a
MACHINE_TYPE: e2-small
INTERNAL_IP: 10.160.0.6
EXTERNAL_IP: 34.93.207.217
STATUS: RUNNING

NAME: web3
ZONE: asia-south1-a
MACHINE_TYPE: e2-small
INTERNAL_IP: 10.160.0.7
EXTERNAL_IP: 35.200.197.183
STATUS: RUNNING
```

**O que é cada coluna:**
- `INTERNAL_IP`: IP privado dentro da VPC (rede interna do GCP)
- `EXTERNAL_IP`: IP público que pode ser acessado da internet

### Passo 4: Criar Firewall Rule

```bash
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80
```

**Por quê?**
- Autoriza tráfego HTTP (porta 80) para instâncias com a tag `network-lb-tag`
- Sem isso, firewall bloquearia toda entrada de tráfego por padrão

### Passo 5: Testar Conectividade

```bash
curl http://35.200.197.183
curl http://34.93.207.217
curl http://34.14.162.101
```

**Resultado esperado:**
Cada curl retorna `<h3>Web Server: web-number webX</h3>` (conteúdo da página HTML criada pelo startup script).

### Resumo Visual - Tarefa 1

```
[Cliente na Internet]
        ↓
[Firewall Rule: allow tcp:80 para tag network-lb-tag]
        ↓
[GCP VPC - asia-south1-a]
        ↓
    web1              web2              web3
  (10.160.0.5)    (10.160.0.6)      (10.160.0.7)
  34.14.162.101   34.93.207.217     35.200.197.183
     Apache          Apache            Apache
     Porta 80        Porta 80          Porta 80
        ↓              ↓                 ↓
[Todos acessíveis individualmente via IP externo]
```

---

## TAREFA 2: Configurar Network Load Balancer

### Conceito
Network Load Balancer (Layer 4) distribui conexões TCP/UDP entre as instâncias de backend. É mais rápido mas menos inteligente que HTTP Load Balancer (não lê o conteúdo da requisição).

### Passo 1: Criar Endereço IP Estático

```bash
gcloud compute addresses create network-lb-ip-1 \
  --region asia-south1
```

**Por quê?**
- O Load Balancer precisa de um IP fixo
- Se não reservarmos, o IP poderia mudar após deletar/recriar o load balancer
- Clientes conectam a esse IP, que é roteado para os servidores de backend

### Passo 2: Criar Health Check HTTP

```bash
gcloud compute http-health-checks create basic-check
```

**Por quê?**
- O load balancer precisa saber se cada instância de backend está saudável
- Periodicamente testa `/` (raiz) da porta 80
- Se uma instância não responder, para de enviar tráfego para ela

### Passo 3: Criar Target Pool

```bash
gcloud compute target-pools create www-pool \
  --region asia-south1 --http-health-check basic-check
```

**Por quê?**
- Target Pool é um grupo que agrupa instâncias de backend
- Vincula o health check ao pool
- Aqui é onde o load balancer busca as instâncias para balancear

### Passo 4: Adicionar Instâncias ao Target Pool

```bash
gcloud compute target-pools add-instances www-pool \
    --instances web1,web2,web3
```

**Por quê?**
- Associa as instâncias individuais ao pool
- Agora o load balancer sabe para quais máquinas enviar tráfego

### Passo 5: Criar Forwarding Rule

```bash
gcloud compute forwarding-rules create www-rule \
    --region asia-south1 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
```

**Por quê?**
- Forwarding Rule é a "entrada" do load balancer
- Define: qual IP recebe tráfego (`--address`), em qual porta (`--ports`), e para qual pool redireciona (`--target-pool`)
- Quando alguém acessa `network-lb-ip-1:80`, o tráfego é distribuído entre web1, web2, web3

### Passo 6: Obter IP do Load Balancer

```bash
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule \
  --region asia-south1 --format="json" | jq -r .IPAddress)
echo $IPADDRESS
```

**Resultado esperado:**
```
34.47.249.144
```

### Passo 7: Testar Load Balancer

```bash
while true; do curl -m1 34.47.249.144; done
```

**Por quê?**
- Cada requisição vai para um servidor diferente (round-robin)
- Loop infinito mostra a distribuição de carga
- `-m1` limita a 1 segundo por requisição

**Resultado esperado:**
```
Page served from: lb-backend-xxxxx
Page served from: lb-backend-yyyyy
Page served from: lb-backend-xxxxx
...
```

### Resumo Visual - Tarefa 2

```
[Cliente na Internet]
        ↓
[Forwarding Rule - 34.47.249.144:80]
        ↓
[Network Load Balancer (Layer 4 - TCP/UDP)]
        ↓
[Target Pool + Health Check (basic-check)]
        ↓
[Round Robin Distribution]
        ↓
    web1              web2              web3
(10.160.0.5)      (10.160.0.6)      (10.160.0.7)
   Apache           Apache            Apache
   Porta 80         Porta 80          Porta 80
     ✓               ✓                 ✓
  (Saudável)     (Saudável)      (Saudável)
```

**Fluxo:**
1. Cliente conecta a `34.47.249.144:80`
2. Forwarding Rule recebe a conexão
3. Target Pool balanceia entre web1, web2, web3
4. Health Check verifica continuamente se todas estão vivas
5. Se uma cair, não recebe mais tráfego

---

## TAREFA 3: Criar HTTP Load Balancer (Application Load Balancer)

### Conceito
HTTP Load Balancer (Layer 7) é mais inteligente: lê o conteúdo das requisições HTTP, permitindo roteamento avançado (por URL, por host, por header, etc.). É mais lento que NLB mas oferece mais funcionalidades.

### Passo 1: Criar Instance Template

```bash
gcloud compute instance-templates create lb-backend-template \
   --region=asia-south1 \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-12 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
```

**Por quê?**
- Template é um "modelo" que define como criar novas instâncias
- Define OS, tipo de máquina, tags, e startup script
- Usado por Managed Instance Groups para criar instâncias de forma consistente

**Novidades neste template:**
- `a2ensite` e `a2enmod ssl`: Ativa suporte a HTTPS
- Metadata query para buscar o hostname da instância (mostra qual instância respondeu)

### Passo 2: Criar Managed Instance Group

```bash
gcloud compute instance-groups managed create lb-backend-group \
   --base-instance-name=lb-backend \
   --template=lb-backend-template \
   --size=2 \
   --region=asia-south1
```

**Por quê?**
- Managed Instance Group (MIG) é como um "template automático" que cria/gerencia instâncias
- Cria 2 instâncias inicialmente (`--size=2`)
- Se uma instância falhar, MIG cria uma nova automaticamente
- Muito mais prático que criar instâncias manualmente

### Passo 3: Criar Firewall Rule para Health Check

```bash
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```

**Por quê?**
- Health Check vem de IPs específicos do GCP (ranges acima)
- Precisa autorizar essas IPs para que o health check consiga acessar as instâncias
- Sem isso, health check falha e load balancer não sabe se as instâncias estão vivas

**Ranges:**
- `130.211.0.0/22`: Range do GCP para health checks
- `35.191.0.0/16`: Range do GCP para load balancer

### Passo 4: Criar Endereço IP Estático Global

```bash
gcloud compute addresses create lb-ipv4-1 \
   --ip-version=IPV4 \
   --global
```

**Por quê?**
- HTTP Load Balancer é global (não regional como NLB)
- Precisa de um IP global fixo
- `--global` significa que pode ser acessado de qualquer região

### Passo 5: Criar Health Check HTTP

```bash
gcloud compute health-checks create http http-basic-check \
   --port=80
```

**Por quê?**
- Define como o load balancer verifica se as instâncias estão saudáveis
- Faz requests HTTP para `/` na porta 80
- Se receber resposta 200-299, a instância está ok

### Passo 6: Criar Backend Service

```bash
gcloud compute backend-services create web-backend-service \
   --protocol=HTTP \
   --health-checks=http-basic-check \
   --global
```

**Por quê?**
- Backend Service agrupa as instâncias de backend e suas configurações
- Define o protocolo (HTTP), health check, e outras políticas
- `--global` porque será usado por HTTP Load Balancer global

### Passo 7: Adicionar Backend Group ao Backend Service

```bash
gcloud compute backend-services add-backend web-backend-service \
   --instance-group=lb-backend-group \
   --instance-group-region=asia-south1 \
   --global
```

**Por quê?**
- Associa o Managed Instance Group ao Backend Service
- Agora o load balancer sabe onde estão as instâncias

### Passo 8: Criar URL Map

```bash
gcloud compute url-maps create web-map-http \
   --default-service=web-backend-service
```

**Por quê?**
- URL Map define regras de roteamento baseadas na URL/host
- Neste caso, temos apenas uma rota padrão que envia para `web-backend-service`
- Em cenários avançados, você poderia ter: `/api/*` → backend-api, `/static/*` → backend-cdn, etc.

### Passo 9: Criar Target HTTP Proxy

```bash
gcloud compute target-http-proxies create http-lb-proxy \
   --url-map=web-map-http
```

**Por quê?**
- Proxy é a entidade que lê requisições HTTP e as roteia conforme o URL Map
- Efetivamente é o "gerenciador" do load balancer no nível HTTP

### Passo 10: Criar Forwarding Rule

```bash
gcloud compute forwarding-rules create http-lb-forwarding-rule \
   --load-balancing-scheme=external \
   --global \
   --target-http-proxy=http-lb-proxy \
   --address=lb-ipv4-1 \
   --ports=80
```

**Por quê?**
- É a "entrada" final: tráfego chegando em `lb-ipv4-1:80` é recebido aqui
- Redireciona para o `http-lb-proxy`
- `--load-balancing-scheme=external`: Tráfego da internet (não interno)
- `--global`: Disponível em todas as regiões

### Resumo Visual - Tarefa 3

```
[Cliente na Internet]
        ↓
[Forwarding Rule - IP Global:80]
        ↓
[HTTP Load Balancer (Layer 7 - HTTP/HTTPS)]
        ↓
[Target HTTP Proxy - Lê conteúdo HTTP]
        ↓
[URL Map - Roteamento baseado em URL/Host]
        ↓
[Backend Service + Health Check (http-basic-check)]
        ↓
[Managed Instance Group (MIG - auto-scaling)]
        ↓
   lb-backend-1        lb-backend-2
  (10.160.x.x)       (10.160.x.x)
   Apache e2-medium   Apache e2-medium
     Port 80            Port 80
      ✓                 ✓
 (Auto-criada)    (Auto-criada)
```

**Diferenças em relação à Tarefa 2:**
- **Layer 7 vs Layer 4**: Lê requisições HTTP completas
- **URL Map**: Pode rotear `/api/*` para um backend, `/static/*` para outro
- **Managed Instance Group**: Auto-scaling automático, não precisa criar VMs manualmente
- **Global**: Disponível em todas as regiões (não apenas regional)
- **Backend Service**: Mais robusto que Target Pool, com mais configurações

---

## Comandos Úteis para Limpeza/Troubleshooting

### Listar recursos
```bash
gcloud compute instances list
gcloud compute instance-templates list
gcloud compute instance-groups managed list
gcloud compute backend-services list
gcloud compute forwarding-rules list
gcloud compute addresses list
```

### Deletar um template
```bash
gcloud compute instance-templates delete lb-backend-template --quiet
```

### Deletar uma instância
```bash
gcloud compute instances delete web1 --zone=asia-south1-a --quiet
```

### Ver detalhes de uma instância
```bash
gcloud compute instances describe web1 --zone=asia-south1-a
```

### Acessar via SSH
```bash
gcloud compute ssh web1 --zone=asia-south1-a
```

---

## Conceitos-Chave para Relembrar

| Conceito | O quê é | Quando usar |
|----------|---------|-------------|
| **Forwarding Rule** | Entrada de tráfego | Sempre necessário, é o "portão" |
| **Backend Service** | Agrupa backends + configurações | Em HTTP LB, agrupa instâncias |
| **Target Pool** | Versão antiga de Backend Service | Network LB (Layer 4) |
| **Instance Template** | Modelo para criar VMs | Quando precisa criar muitas VMs iguais |
| **Managed Instance Group** | Gerencia VMs automaticamente | Auto-scaling, high availability |
| **Health Check** | Testa se backend está vivo | Sempre, para detectar falhas |
| **URL Map** | Define roteamento por URL | HTTP LB (Layer 7) |
| **HTTP Proxy** | Lê HTTP e roteia | HTTP LB, para roteamento L7 |

---

## Diferença: Network vs HTTP Load Balancer

| Aspecto | Network LB | HTTP LB |
|--------|-----------|---------|
| **Camada OSI** | Layer 4 (Transporte) | Layer 7 (Aplicação) |
| **Protocolos** | TCP, UDP | HTTP, HTTPS |
| **Latência** | Muito baixa | Mais alta |
| **Roteamento** | Port/protocol | URL, hostname, header |
| **Use case** | Alta performance, gaming, streaming | APIs REST, web apps |
| **Backend** | Target Pool | Backend Service |
| **Escopo** | Regional | Global |