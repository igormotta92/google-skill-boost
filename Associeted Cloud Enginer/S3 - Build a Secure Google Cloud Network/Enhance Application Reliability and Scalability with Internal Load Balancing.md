# Enhance Application Reliability and Scalability with Internal Load Balancing

## Introdução

Este guia segue o mesmo padrão do documento de referência e mostra como executar todo o laboratório via CLI, explicando:

- o que cada recurso cria;
- por que ele é necessário;
- quais parâmetros principais são usados.

O objetivo é montar uma arquitetura de Internal Load Balancer (ILB) regional no Google Cloud para distribuir tráfego interno entre dois backends em zonas diferentes.

---

## Pré-requisitos e Variáveis

### Conceito
Antes de criar recursos, padronizamos variáveis para evitar erros de digitação e facilitar reutilização dos comandos.

### Passo 1: Definir variáveis de ambiente

```bash
# Região/zona do lab
export REGION="us-east4"
export ZONE_A="us-east4-c"
export ZONE_B="us-east4-b"

# Rede e sub-redes existentes no ambiente do lab
export NETWORK="my-internal-app"
export SUBNET_A="subnet-a"
export SUBNET_B="subnet-b"

# Tags e nomes padrão
export BACKEND_TAG="lb-backend"
export TEMPLATE_1="instance-template-1"
export TEMPLATE_2="instance-template-2"
export MIG_1="instance-group-1"
export MIG_2="instance-group-2"
export HEALTH_CHECK="my-ilb-health-check"
export BACKEND_SERVICE="my-ilb-backend-service"
export ILB_IP_NAME="my-ilb-ip"
export ILB_FORWARDING_RULE="my-ilb-fr"
```

### Passo 2: Configurar região padrão no gcloud

```bash
gcloud config set compute/region "$REGION"
```

**Por quê?**
- Define a região padrão para comandos regionais.
- Evita repetição de `--region` em parte dos comandos.

---

## TAREFA 1: Configurar regras de firewall HTTP e Health Check

### Conceito
As regras de firewall garantem:

- tráfego HTTP permitido para os backends;
- tráfego dos verificadores de saúde do Google Cloud permitido.

Sem isso, o load balancer não valida corretamente o estado das VMs e pode marcar backends como indisponíveis.

### Passo 1: Criar regra para HTTP interno (porta 80)

```bash
gcloud compute firewall-rules create app-allow-http \
	--network="$NETWORK" \
	--direction=INGRESS \
	--action=ALLOW \
	--target-tags="$BACKEND_TAG" \
	--source-ranges="10.10.0.0/16" \
	--rules="tcp:80"
```

**Explicação dos parâmetros:**
- `--network`: rede VPC onde a regra será aplicada.
- `--target-tags`: aplica a regra apenas às VMs com tag `lb-backend`.
- `--source-ranges=10.10.0.0/16`: permite origem dos blocos internos do lab.
- `--rules=tcp:80`: libera somente HTTP na porta 80.

### Passo 2: Criar regra para Health Checks do Google Cloud

```bash
gcloud compute firewall-rules create app-allow-health-check \
	--network="$NETWORK" \
	--direction=INGRESS \
	--action=ALLOW \
	--target-tags="$BACKEND_TAG" \
	--source-ranges="130.211.0.0/22,35.191.0.0/16" \
	--rules="tcp"
```

**Explicação dos parâmetros:**
- `--source-ranges=130.211.0.0/22,35.191.0.0/16`: ranges oficiais de health check do Google Cloud.
- `--rules=tcp`: permite probes TCP para validação de saúde.

### Passo 3: Validar regras criadas

```bash
gcloud compute firewall-rules list \
	--filter="name=('app-allow-http' 'app-allow-health-check')" \
	--format="table(name,network,direction,sourceRanges.list():label=SOURCE_RANGES,allowed[].map().firewall_rule().list():label=ALLOW,targetTags.list():label=TARGET_TAGS)"
```

---

## TAREFA 2: Configurar Instance Templates e Managed Instance Groups

### Conceito
- Instance Template define padrão de VM (SO, máquina, rede, script, tags).
- Managed Instance Group (MIG) cria e mantém instâncias idênticas automaticamente.

Vamos criar:

- `instance-template-1` em `subnet-a`;
- `instance-template-2` em `subnet-b`;
- `instance-group-1` e `instance-group-2` em zonas diferentes da mesma região.

### Passo 1: Criar startup script local

```bash
cat > startup.sh <<'EOF'
#!/bin/bash
apt-get update
apt-get install -y apache2 php libapache2-mod-php
cat <<'PHP' > /var/www/html/index.php
<h1>Internal Load Balancing Lab</h1>
<h2>Client IP</h2>
Your IP address : <?php echo $_SERVER['REMOTE_ADDR']; ?>
<h2>Hostname</h2>
Server Hostname: <?php echo gethostname(); ?>
<h2>Server Location</h2>
Region and Zone: <?php
	$ch = curl_init();
	curl_setopt($ch, CURLOPT_URL, "http://metadata.google.internal/computeMetadata/v1/instance/zone");
	curl_setopt($ch, CURLOPT_HTTPHEADER, array('Metadata-Flavor: Google'));
	curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
	$zone = curl_exec($ch);
	$parts = explode('/', $zone);
	echo end($parts);
?>
PHP
rm -f /var/www/html/index.html
systemctl restart apache2
EOF
```

**Por quê?**
- Instala Apache + PHP.
- Publica página dinâmica para identificar IP do cliente, hostname e zona da VM backend.

### Passo 2: Criar template 1 (subnet-a)

```bash
gcloud compute instance-templates create "$TEMPLATE_1" \
	--machine-type="e2-micro" \
	--network="$NETWORK" \
	--subnet="$SUBNET_A" \
	--no-address \
	--tags="$BACKEND_TAG" \
	--metadata-from-file="startup-script=startup.sh" \
	--image-family="debian-12" \
	--image-project="debian-cloud"

# Verificando se o startup-script foi incluido
gcloud compute instance-templates describe instance-template-1 \
  --format="yaml(properties.metadata.items)"
```

**Explicação dos parâmetros principais:**
- `--no-address`: sem IP externo (acesso interno).
- `--subnet=subnet-a`: aloca VMs no bloco da subnet-a.
- `--tags=lb-backend`: vincula às regras de firewall criadas.
- `--metadata-from-file`: injeta startup script no boot.

### Passo 3: Criar template 2 (subnet-b)

```bash
gcloud compute instance-templates create "$TEMPLATE_2" \
	--machine-type="e2-micro" \
	--network="$NETWORK" \
	--subnet="$SUBNET_B" \
	--no-address \
	--tags="$BACKEND_TAG" \
	--metadata-from-file="startup-script=startup.sh" \
	--image-family="debian-12" \
	--image-project="debian-cloud"

# Verificando se o startup-script foi incluido
gcloud compute instance-templates describe instance-template-1 \
  --format="yaml(properties.metadata.items)"
```

### Passo 4: Criar MIG 1 (zona A)

```bash
gcloud compute instance-groups managed create "$MIG_1" \
	--template="$TEMPLATE_1" \
	--size=1 \
	--zone="$ZONE_A"
```

### Passo 5: Definir autoscaling no MIG 1

```bash
gcloud compute instance-groups managed set-autoscaling "$MIG_1" \
	--zone="$ZONE_A" \
	--min-num-replicas=1 \
	--max-num-replicas=1 \
	--target-cpu-utilization=0.8 \
	--cool-down-period=45
```

### Passo 6: Criar MIG 2 (zona B)

```bash
gcloud compute instance-groups managed create "$MIG_2" \
	--template="$TEMPLATE_2" \
	--size=1 \
	--zone="$ZONE_B"
```

### Passo 7: Definir autoscaling no MIG 2

```bash
gcloud compute instance-groups managed set-autoscaling "$MIG_2" \
	--zone="$ZONE_B" \
	--min-num-replicas=1 \
	--max-num-replicas=1 \
	--target-cpu-utilization=0.8 \
	--cool-down-period=45
```

**Por quê autoscaling 1..1?**
- Mantém o comportamento do lab (uma instância por MIG), mas já com política de autoscaling configurada.

### Passo 8: Criar VM utilitária para testes internos

```bash
gcloud compute instances create utility-vm \
	--zone="$ZONE_A" \
	--machine-type="e2-micro" \
	--network-interface="subnet=$SUBNET_A,private-network-ip=10.10.20.50,no-address"
```

**Explicação:**
- VM de apoio para testar acesso interno aos backends e ao ILB.
- IP interno fixo `10.10.20.50` como no roteiro.

### Passo 9: Verificar backends diretamente

```bash
gcloud compute instances list --filter="name~'^instance-group-[12]'" \
	--format="table(name,zone,networkInterfaces[0].networkIP,status)"
```

Conecte na utility VM:

```bash
gcloud compute ssh utility-vm --zone="$ZONE_A"
```

Dentro da VM utilitária, teste os IPs internos dos backends:

```bash
curl 10.10.20.2
curl 10.10.30.2
```

Saída esperada: página com Client IP, Server Hostname e Server Location.

---

## TAREFA 3: Configurar o Internal Load Balancer (ILB)

### Conceito
O ILB expõe um endpoint privado único dentro da VPC e distribui conexões TCP para backends saudáveis, mantendo o serviço resiliente e escalável.

### Passo 1: Criar health check TCP

```bash
gcloud compute health-checks create tcp "$HEALTH_CHECK" \
	--region="$REGION" \
	--port=80
```

**Por quê?**
- Verifica continuamente se cada backend responde na porta 80.
- Apenas instâncias saudáveis recebem tráfego.

### Passo 2: Criar backend service regional interno

```bash
gcloud compute backend-services create "$BACKEND_SERVICE" \
	--load-balancing-scheme=INTERNAL \
	--protocol=TCP \
	--region="$REGION" \
	--health-checks="$HEALTH_CHECK" \
	--health-checks-region="$REGION"
```

**Explicação dos parâmetros:**
- `--load-balancing-scheme=INTERNAL`: define ILB privado.
- `--protocol=TCP`: passthrough L4 (alinhado ao tipo do lab).
- `--health-checks`: associa checagem de saúde ao backend service.

### Passo 3: Adicionar MIG 1 no backend service

```bash
gcloud compute backend-services add-backend "$BACKEND_SERVICE" \
	--region="$REGION" \
	--instance-group="$MIG_1" \
	--instance-group-zone="$ZONE_A"
```

### Passo 4: Adicionar MIG 2 no backend service

```bash
gcloud compute backend-services add-backend "$BACKEND_SERVICE" \
	--region="$REGION" \
	--instance-group="$MIG_2" \
	--instance-group-zone="$ZONE_B"
```

### Passo 5: Reservar IP interno estático do ILB

```bash
gcloud compute addresses create "$ILB_IP_NAME" \
	--region="$REGION" \
	--subnet="$SUBNET_B" \
	--addresses="10.10.30.5"
```

**Por quê?**
- Define endpoint interno fixo para consumo por outros serviços na VPC.

### Passo 6: Criar forwarding rule do ILB (porta 80)

```bash
gcloud compute forwarding-rules create "$ILB_FORWARDING_RULE" \
	--load-balancing-scheme=INTERNAL \
	--network="$NETWORK" \
	--subnet="$SUBNET_B" \
	--address="10.10.30.5" \
	--ports=80 \
	--region="$REGION" \
	--backend-service="$BACKEND_SERVICE" \
	--backend-service-region="$REGION"
```

**Explicação dos parâmetros:**
- `--subnet=subnet-b`: frontend do ILB nessa subnet.
- `--address=10.10.30.5`: IP interno estático do balanceador.
- `--backend-service`: serviço que contém os dois MIGs.

### Passo 7: Validar criação do ILB

```bash
gcloud compute forwarding-rules describe "$ILB_FORWARDING_RULE" \
	--region="$REGION" \
	--format="table(name,IPAddress,IPProtocol,loadBalancingScheme,backendService)"
```

---

## TAREFA 4: Testar o Internal Load Balancer

### Conceito
Aqui confirmamos que o IP interno do ILB encaminha tráfego para os dois backends, conforme saúde e distribuição do serviço.

### Passo 1: Acessar utility-vm

```bash
gcloud compute ssh utility-vm --zone="$ZONE_A"
```

### Passo 2: Testar endpoint interno do ILB

```bash
curl 10.10.30.5
```

### Passo 3: Testar distribuição de respostas

```bash
for i in {1..10}; do curl -s 10.10.30.5 | grep -E 'Server Hostname|Server Location'; echo; done
```

**Resultado esperado:**
- Respostas alternando entre instâncias dos dois MIGs.
- Campo `Server Location` indicando zonas diferentes.

---

## Resumo Visual da Arquitetura

```text
[utility-vm 10.10.20.50] (subnet-a)
					 |
					 | curl 10.10.30.5:80
					 v
[Internal Load Balancer - 10.10.30.5] (subnet-b, us-east4)
					 |
					 v
[Backend Service Regional + TCP Health Check]
			 |                           |
			 v                           v
[MIG instance-group-1]       [MIG instance-group-2]
[subnet-a / ZONE_A]          [subnet-b / ZONE_B]
```

---

## Comandos de Troubleshooting

### Listar recursos principais

```bash
gcloud compute firewall-rules list --filter="name~'app-allow'"
gcloud compute instance-templates list --filter="name~'instance-template-[12]'"
gcloud compute instance-groups managed list --filter="name~'instance-group-[12]'"
gcloud compute health-checks list --filter="name=$HEALTH_CHECK"
gcloud compute backend-services list --filter="name=$BACKEND_SERVICE"
gcloud compute forwarding-rules list --filter="name=$ILB_FORWARDING_RULE"
gcloud compute addresses list --filter="name=$ILB_IP_NAME"
```

### Ver saúde dos backends

```bash
gcloud compute backend-services get-health "$BACKEND_SERVICE" \
	--region="$REGION"
```

### Ver detalhes do MIG

```bash
gcloud compute instance-groups managed describe "$MIG_1" --zone="$ZONE_A"
gcloud compute instance-groups managed describe "$MIG_2" --zone="$ZONE_B"
```

---

## Limpeza (Opcional)

```bash
gcloud compute forwarding-rules delete "$ILB_FORWARDING_RULE" --region="$REGION" --quiet
gcloud compute addresses delete "$ILB_IP_NAME" --region="$REGION" --quiet
gcloud compute backend-services delete "$BACKEND_SERVICE" --region="$REGION" --quiet
gcloud compute health-checks delete "$HEALTH_CHECK" --region="$REGION" --quiet

gcloud compute instance-groups managed delete "$MIG_1" --zone="$ZONE_A" --quiet
gcloud compute instance-groups managed delete "$MIG_2" --zone="$ZONE_B" --quiet

gcloud compute instance-templates delete "$TEMPLATE_1" --quiet
gcloud compute instance-templates delete "$TEMPLATE_2" --quiet

gcloud compute instances delete utility-vm --zone="$ZONE_A" --quiet

gcloud compute firewall-rules delete app-allow-http --quiet
gcloud compute firewall-rules delete app-allow-health-check --quiet

rm -f startup.sh
```

---

## Conceitos-Chave para Relembrar

| Conceito | O que é | Quando usar |
|---|---|---|
| Firewall Rule | Controle de tráfego por origem/porta/tag | Sempre para segurança e conectividade mínima |
| Instance Template | Modelo de VM padronizada | Criar VMs idênticas com consistência |
| Managed Instance Group | Grupo gerenciado de VMs | Alta disponibilidade, autohealing e autoscaling |
| Health Check | Verificação contínua de saúde | Evitar envio de tráfego para backend indisponível |
| Backend Service (Regional) | Define política e membros de backend | ILB regional com distribuição entre grupos |
| Forwarding Rule (Internal) | Endpoint privado do balanceador | Expor IP interno único para os consumidores |

---

## Fluxo Final do Tráfego

1. Cliente interno (utility-vm) acessa `10.10.30.5:80`.
2. Forwarding rule interna recebe a conexão.
3. Backend service seleciona um backend saudável.
4. MIG 1 ou MIG 2 responde com página contendo hostname e zona.
5. Health checks mantêm somente instâncias saudáveis no pool de tráfego.
