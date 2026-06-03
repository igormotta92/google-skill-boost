# Configuração Completa do Ambiente Griffin com WordPress e Kubernetes

## Visão Geral

Este guia converte o desafio final em comandos executáveis via CLI gcloud. O objetivo é preparar um ambiente de desenvolvimento completo para WordPress utilizando Google Cloud, incluindo:

- Criar duas VPCs (desenvolvimento e produção) com subnets isoladas
- Configurar um bastion host para acesso gerenciado
- Provisionar uma instância Cloud SQL para banco de dados WordPress
- Implantar um cluster Kubernetes para hospedagem de WordPress
- Configurar monitoramento e controle de acesso

## Introdução

O ambiente Griffin requer uma arquitetura robusta com separação entre desenvolvimento e produção. Este guia executa cada componente via CLI gcloud, facilitando reprodução, automação e controle de versão.

---

## Pré-requisitos e Variáveis

### Conceito

Padronizamos variáveis para evitar erros de digitação e manter consistência entre comandos.

### Passo 1: Definir variáveis de ambiente

```bash
# Configuração do projeto e região
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="europe-west3"  # Ajuste conforme ambiente
export ZONE="europe-west3-c"  # Ajuste conforme ambiente

# VPCs e Subnets
export DEV_VPC="griffin-dev-vpc"
export DEV_SUBNET_WP="griffin-dev-wp"
export DEV_SUBNET_MGMT="griffin-dev-mgmt"
export PROD_VPC="griffin-prod-vpc"
export PROD_SUBNET_WP="griffin-prod-wp"
export PROD_SUBNET_MGMT="griffin-prod-mgmt"

# Instâncias e recursos
export BASTION_HOST="griffin-bastion"
export DEV_SQL_INSTANCE="griffin-dev-db"
export K8S_CLUSTER="griffin-dev"
export DB_USER="wp_user"
export DB_PASSWORD="stormwind_rules"
export DB_NAME="wordpress"
```

### Passo 2: Configurar região padrão no gcloud

```bash
gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"
```

---

## TAREFA 1: Criar VPC de Desenvolvimento com Subnets

### Conceito

A VPC de desenvolvimento isolada encapsula recursos da aplicação WordPress. As subnets separadas por função (WordPress e Gerenciamento) permitem segmentação de tráfego e políticas de firewall granulares.

### Passo 1: Criar a VPC de desenvolvimento

```bash
gcloud compute networks create "$DEV_VPC" \
  --subnet-mode=custom \
  --bgp-routing-mode=regional
```

**Explicação dos parâmetros:**
- `--subnet-mode=custom`: permite criar subnets manualmente com CIDRs específicos.
- `--bgp-routing-mode=regional`: roteamento regional para rede híbrida futura.

### Passo 2: Criar subnet para WordPress

```bash
gcloud compute networks subnets create "$DEV_SUBNET_WP" \
  --network="$DEV_VPC" \
  --region="$REGION" \
  --range="192.168.16.0/20"
```

**Explicação dos parâmetros:**
- `--network`: VPC à qual esta subnet será associada.
- `--range`: bloco CIDR da subnet; `192.168.16.0/20` reserva 4096 endereços para aplicações WordPress.

### Passo 3: Criar subnet de gerenciamento

```bash
gcloud compute networks subnets create "$DEV_SUBNET_MGMT" \
  --network="$DEV_VPC" \
  --region="$REGION" \
  --range="192.168.32.0/20"
```

**Explicação dos parâmetros:**
- `--network`: VPC à qual esta subnet será associada.
- `--range`: bloco CIDR `192.168.32.0/20`; subnet isolada para bastion e ferramentas administrativas.

### Passo 4: Validar criação da VPC de desenvolvimento

```bash
gcloud compute networks describe "$DEV_VPC"
gcloud compute networks subnets list --filter="network:$DEV_VPC"
```

**Resultado esperado:** VPC exibida com suas propriedades; duas subnets listadas com ranges corretos.

---

## TAREFA 2: Criar VPC de Produção com Subnets

### Conceito

A VPC de produção segue a mesma estrutura da VPC de desenvolvimento, garantindo consistência operacional e facilidade de escalabilidade futura.

### Passo 1: Criar a VPC de produção

```bash
gcloud compute networks create "$PROD_VPC" \
  --subnet-mode=custom \
  --bgp-routing-mode=regional
```

### Passo 2: Criar subnet para WordPress

```bash
gcloud compute networks subnets create "$PROD_SUBNET_WP" \
  --network="$PROD_VPC" \
  --region="$REGION" \
  --range="192.168.48.0/20"
```

**Explicação dos parâmetros:**
- `--network`: VPC de produção à qual esta subnet será associada.
- `--range`: bloco CIDR `192.168.48.0/20`; 4096 endereços para aplicações em produção.

### Passo 3: Criar subnet de gerenciamento

```bash
gcloud compute networks subnets create "$PROD_SUBNET_MGMT" \
  --network="$PROD_VPC" \
  --region="$REGION" \
  --range="192.168.64.0/20"
```

**Explicação dos parâmetros:**
- `--network`: VPC de produção à qual esta subnet será associada.
- `--range`: bloco CIDR `192.168.64.0/20`; subnet isolada para administração da produção.

### Passo 4: Validar criação da VPC de produção

```bash
gcloud compute networks describe "$PROD_VPC"
gcloud compute networks subnets list --filter="network:$PROD_VPC"
```

**Resultado esperado:** VPC e subnets de produção criadas corretamente.

---

## TAREFA 3: Criar Bastion Host com Múltiplas Interfaces

### Conceito

O bastion (jump host) com duas interfaces de rede permite acesso administrativo a ambos os ambientes (desenvolvimento e produção) através de um único ponto de entrada seguro.

### Passo 1: Criar regra de firewall para SSH via IAP (dev)

```bash
gcloud compute firewall-rules create "allow-ssh-from-iap" \
  --network="$DEV_VPC" \
  --allow=tcp:22 \
  --source-ranges="35.235.240.0/20" \
  --description="Permite SSH via Identity-Aware Proxy"
```

**Explicação dos parâmetros:**
- `--network`: VPC à qual a regra será aplicada.
- `--allow=tcp:22`: protocolo e porta liberados; `tcp:22` é a porta padrão do SSH.
- `--source-ranges=35.235.240.0/20`: range oficial do IAP do Google Cloud; restringe a origem a esse CIDR.
- `--description`: texto descritivo exibido na listagem de regras para facilitar auditoria.

### Passo 2: Criar regra de firewall para SSH via IAP (prod)

```bash
gcloud compute firewall-rules create "allow-ssh-from-iap-prod" \
  --network="$PROD_VPC" \
  --allow=tcp:22 \
  --source-ranges="35.235.240.0/20"
  --description="Permite SSH via Identity-Aware Proxy"
```

**Explicação dos parâmetros:**
- `--network`: aplica a regra à VPC de produção.
- `--allow=tcp:22`: libera a porta SSH.
- `--source-ranges=35.235.240.0/20`: permite entrada apenas a partir dos servidores IAP do Google.

### Passo 3: Criar instância bastion com duas interfaces de rede

No GCP, interfaces de rede adicionais devem ser definidas **no momento da criação** da instância. Use dois `--network-interface` no mesmo comando:

```bash
gcloud compute instances create "$BASTION_HOST" \
  --zone="$ZONE" \
  --machine-type="e2-medium" \
  --network-interface="subnet=$DEV_SUBNET_MGMT,private-network-ip=192.168.32.10,no-address" \
  --network-interface="subnet=$PROD_SUBNET_MGMT,private-network-ip=192.168.64.10,no-address" \
  --image-family="debian-12" \
  --image-project="debian-cloud" \
  --tags="bastion"
```

**Explicação dos parâmetros:**
- `--network-interface` *(primeira)*: define a NIC principal (nic0) na subnet de gerenciamento da VPC dev; `private-network-ip` fixa o IP interno em `192.168.32.10`; `no-address` omite IP público.
- `--network-interface` *(segunda)*: define a NIC secundária (nic1) na subnet de gerenciamento da VPC prod; `private-network-ip` fixa o IP em `192.168.64.10`; `no-address` omite IP público.
- `--image-family=debian-12`: seleciona a família de imagem mais recente do Debian 12.
- `--image-project=debian-cloud`: projeto público do Google que hospeda imagens Debian oficiais.
- `--tags=bastion`: aplica network tag à instância; permite direcionar regras de firewall a esse host especificamente.

### Passo 5: Validar bastion com duas interfaces

```bash
gcloud compute instances describe "$BASTION_HOST" \
  --zone="$ZONE" \
  --format="table(name,networkInterfaces[].network,networkInterfaces[].subnetwork,networkInterfaces[].networkIP)"
```

**Explicação dos parâmetros:**
- `--format`: controla a saída do comando; `table(...)` exibe os campos indicados em formato tabular; a notação `[]` itera sobre a lista de NICs, exibindo rede, subnet e IP de cada uma.

**Resultado esperado:** Bastion com 2 NICs em subnets diferentes.

---

## TAREFA 4: Criar e Configurar Cloud SQL Instance

### Conceito

A instância Cloud SQL hospeda o banco de dados WordPress. Configuraremos usuários, bancos e permissões necessários para a aplicação.

### Passo 1: Criar instância MySQL

```bash
gcloud sql instances create "$DEV_SQL_INSTANCE" \
  --database-version=MYSQL_8_0 \
  --tier=db-f1-micro \
  --region="$REGION"
```
> **Observação:** para criar a instância com IP privado usando `--network`, é obrigatório habilitar o **Private Service Access (Service Networking)** na VPC antes. Sem esse pré-requisito, o comando falha com `SERVICE_NETWORKING_NOT_ENABLED`.
>
> Exemplo do parâmetro de rede privada:
> `--network="projects/$PROJECT_ID/global/networks/$DEV_VPC"`

**Explicação dos parâmetros:**
- `--database-version=MYSQL_8_0`: versão do engine de banco de dados; `MYSQL_8_0` é a versão estável mais recente do MySQL no Cloud SQL.
- `--tier=db-f1-micro`: tipo de máquina da instância SQL; tier econômico adequado para desenvolvimento.
- `--network`: vincula a instância à VPC de desenvolvimento via IP privado (Private Service Access), evitando tráfego pela internet pública.

### Passo 2: Aguardar disponibilidade (verificar status)

```bash
gcloud sql instances describe "$DEV_SQL_INSTANCE" \
  --format="value(state)"
```

**Explicação dos parâmetros:**
- `--format=value(state)`: extrai apenas o campo `state` da resposta, sem cabeçalhos ou formatação extra; útil para checar o status em scripts.

**Resultado esperado:** RUNNABLE

### Passo 3: Criar usuário root (se necessário)

```bash
gcloud sql users create root \
  --instance="$DEV_SQL_INSTANCE" \
  --password="stormwind_rules"
```

**Explicação dos parâmetros:**
- `--instance`: nome da instância Cloud SQL onde o usuário será criado.
- `--password`: senha do usuário; recomenda-se variável de ambiente em ambientes reais para evitar exposição no histórico do shell.

### Passo 4: Conectar e executar comandos SQL

Conectar via Cloud Shell:

```bash
gcloud sql connect "$DEV_SQL_INSTANCE" \
  --user=root
```

**Explicação dos parâmetros:**
- `--user`: usuário com o qual o cliente MySQL será autenticado na instância; deve ter permissão de acesso configurada previamente.

Dentro do client MySQL, executar:

```sql
CREATE DATABASE wordpress;
CREATE USER "wp_user"@"%" IDENTIFIED BY "stormwind_rules";
GRANT ALL PRIVILEGES ON wordpress.* TO "wp_user"@"%";
FLUSH PRIVILEGES;
EXIT;
```

### Passo 5: Validar criação de banco e usuário

```bash
gcloud sql databases list --instance="$DEV_SQL_INSTANCE"
gcloud sql users list --instance="$DEV_SQL_INSTANCE"
```

**Resultado esperado:** Banco "wordpress" listado; usuário "wp_user" com acesso.

---

## TAREFA 5: Criar Cluster Kubernetes na VPC de Desenvolvimento

### Conceito

O cluster GKE em 2 nós hospedará os containers WordPress. Será criado na subnet de WordPress da VPC de desenvolvimento.

### Passo 1: Criar cluster GKE

```bash
gcloud container clusters create "$K8S_CLUSTER" \
  --zone="$ZONE" \
  --num-nodes=2 \
  --machine-type="e2-standard-4" \
  --network="$DEV_VPC" \
  --subnetwork="$DEV_SUBNET_WP" \
  --enable-ip-alias \
  --enable-autorepair \
  --enable-autoupgrade
```

**Explicação dos parâmetros:**
- `--num-nodes=2`: número de nós (VMs worker) por zona no node pool padrão.
- `--network`: VPC na qual o cluster será criado; os nós receberão IPs dessa rede.
- `--subnetwork`: subnet específica dentro da VPC onde os nós serão provisionados.
- `--enable-ip-alias`: habilita IP aliasing (VPC-native cluster), permitindo que pods recebam IPs diretamente da subnet, melhorando roteamento e integração com outros serviços GCP.
- `--enable-autorepair`: o GKE monitora e repara automaticamente nós com falha de saúde.
- `--enable-autoupgrade`: mantém a versão do Kubernetes dos nós atualizada automaticamente com patches de segurança e melhorias.

### Passo 2: Obter credenciais do cluster

```bash
gcloud container clusters get-credentials "$K8S_CLUSTER" \
  --zone="$ZONE"
```

**Resultado:** atualiza kubeconfig com acesso ao cluster.

### Passo 3: Validar cluster

```bash
kubectl get nodes
kubectl get namespaces
```

**Resultado esperado:** 2 nós listados; namespaces padrão visíveis.

---

## TAREFA 6: Preparar Cluster Kubernetes para WordPress

### Conceito

Configuraremos secrets para credenciais de banco e volumes persistentes necessários para WordPress.

### Passo 1: Copiar arquivos de configuração

```bash
gsutil cp -r gs://spls/gsp321/wp-k8s ./
```

**Explicação:**
- Copia YAML de deployment, service e environment do WordPress.

### Passo 2: Configurar `wp-env.yaml` com secret e volume

O arquivo baixado (`wp-k8s/wp-env.yaml`) ja contem os recursos esperados do lab (Secret `database` e PVC `wordpress-volumeclaim`).
Atualize somente os valores de usuario e senha dentro do bloco `stringData`:

```yaml
stringData:
  # WORDPRESS_DB_HOST: 127.0.0.1:3306
  WORDPRESS_DB_USER: wp_user
  WORDPRESS_DB_PASSWORD: stormwind_rules
```

Aplicar o arquivo de forma idempotente (cria se nao existir, atualiza se ja existir):

```bash
kubectl apply -f wp-k8s/wp-env.yaml
```

Se quiser validar que o volume e o secret estao presentes:

```bash
kubectl get secret database
kubectl get pvc wordpress-volumeclaim
```

### Passo 3: Criar key de service account para Cloud SQL Proxy

```bash
gcloud iam service-accounts keys create key.json \
  --iam-account=cloud-sql-proxy@$PROJECT_ID.iam.gserviceaccount.com
```

**Explicação dos parâmetros:**
- `--iam-account`: e-mail da service account para a qual a chave será gerada; a chave resultante (`key.json`) permite que o Cloud SQL Proxy autentique com as permissões dessa conta.

### Passo 4: Adicionar key ao cluster como secret

```bash
kubectl create secret generic cloudsql-instance-credentials \
  --from-file=key.json \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Explicação dos parâmetros:**
- `--from-file`: cria o secret a partir de um arquivo local; o conteúdo de `key.json` é armazenado como entrada do secret com o nome do arquivo como chave.
- `--dry-run=client -o yaml | kubectl apply -f -`: torna o comando idempotente para evitar erro `AlreadyExists` em reexecuções.

### Passo 5: Validar secrets criados

<!-- **Nota:** Se receber erro "You must be logged in to the server (Unauthorized)", configure a credencial do cluster:

```bash
gcloud container clusters get-credentials $K8S_CLUSTER --zone $ZONE
``` -->

Então execute os comandos de validação:

```bash
kubectl get secrets
kubectl describe secret database
kubectl describe secret cloudsql-instance-credentials
```

**Resultado esperado:** Ambos os secrets listados e acessíveis.

---

## TAREFA 7: Criar Deployment WordPress

### Conceito

O deployment utiliza a imagem de WordPress com sidecar do Cloud SQL Proxy para conexão segura ao banco.

### Passo 1: Obter nome de conexão da instância SQL

```bash
gcloud sql instances describe "$DEV_SQL_INSTANCE" \
  --format="value(connectionName)"
```

**Saída esperada:** PROJECT_ID:REGION:griffin-dev-db

Armazene este valor:

```bash
export SQL_CONNECTION_NAME=$(gcloud sql instances describe "$DEV_SQL_INSTANCE" \
  --format="value(connectionName)")
```

### Passo 2: Editar wp-deployment.yaml

Substituir `YOUR_SQL_INSTANCE` pelo nome de conexão:

```bash
sed -i "s|YOUR_SQL_INSTANCE|$SQL_CONNECTION_NAME|g" wp-k8s/wp-deployment.yaml
```

### Passo 3: Aplicar deployment

```bash
kubectl apply -f wp-k8s/wp-deployment.yaml
```

### Passo 4: Aguardar pods em execução

```bash
kubectl rollout status deployment/wordpress
kubectl get pods
```

### Passo 5: Aplicar service (LoadBalancer)

```bash
kubectl apply -f wp-k8s/wp-service.yaml

# verificar serviço
kubectl get svc wordpress -w
```

### Passo 6: Obter IP do LoadBalancer

```bash
kubectl get svc
```

**Resultado esperado:** Serviço "wordpress" com EXTERNAL-IP atribuído.

Se aparecer erro `services "wordpress" not found` nos passos seguintes, valide nome e namespace do service:

```bash
kubectl get svc -A
kubectl get svc -A | grep -i wordpress
```

Se não existir nenhum service do WordPress, reaplique:

```bash
kubectl apply -f wp-k8s/wp-service.yaml
kubectl get svc -w
```

### Passo 7: Testar acesso ao WordPress

```bash
EXTERNAL_IP=$(kubectl get svc wordpress \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

**Resultado esperado:** Página de instalação do WordPress retornada.

---

## TAREFA 8: Ativar Monitoramento

### Conceito

Uptime checks monitoram disponibilidade do site WordPress e alertam sobre indisponibilidade.

### Passo 1: Obter URL externa do WordPress

```bash
WORDPRESS_URL=$(kubectl get svc wordpress \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

### Passo 2: Criar uptime check

```bash
gcloud monitoring uptime create "WordPress Dev Site" \
  --resource-type=uptime-url \
  --resource-labels="host=$WORDPRESS_URL,project_id=$PROJECT_ID" \
  --port=80 \
  --path="/" \
  --regions=usa-iowa,usa-oregon,usa-virginia
```

**Explicação dos parâmetros:**
- `"WordPress Dev Site"` (argumento posicional): nome legível exibido no console do Cloud Monitoring para identificar o uptime check.
- `--resource-type=uptime-url`: tipo de recurso monitorado; indica que é uma URL HTTP/HTTPS externa.
- `--resource-labels`: define os labels do recurso monitorado; para `uptime-url`, use `host` (IP/hostname do alvo) e `project_id` (ID do projeto).
- `--port`: porta de destino da verificação; `80` para HTTP.
- `--path`: caminho da requisição HTTP; `"/"` verifica a raiz da aplicação.
- `--regions`: lista de localidades de origem das verificações; este comando exige pelo menos 3 localidades (ex.: `usa-iowa,usa-oregon,usa-virginia`).

### Passo 3: Validar uptime check

```bash
gcloud monitoring uptime list-configs \
  --format="table(displayName,monitoredResource.type)"
```

**Resultado esperado:** Uptime check listado como "WordPress Dev Site".

---

## TAREFA 9: Conceder Acesso a Engenheiro Adicional

### Conceito

O novo engenheiro recebe role de Editor para gerenciar todos os recursos do projeto.

### Passo 1: Obter email do segundo usuário

O segundo usuário é fornecido pelo lab. Ajuste conforme necessário:

```bash
export SECOND_USER_EMAIL="student-03-8a959eb41952@qwiklabs.net"  # Ajuste conforme ambiente
```

### Passo 2: Conceder role de Editor

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="user:$SECOND_USER_EMAIL" \
  --role="roles/editor"
```

**Explicação dos parâmetros:**
- `--member`: identidade que receberá a permissão; o prefixo `user:` indica uma conta Google pessoal (use `serviceAccount:` para service accounts ou `group:` para grupos).
- `--role`: papel IAM a ser concedido; `roles/editor` permite criar, modificar e excluir a maioria dos recursos do projeto, mas não gerenciar permissões de IAM.

### Passo 3: Validar atribuição

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.role:roles/editor" \
  --format="table(bindings.members)"
```

**Explicação dos parâmetros:**
- `--flatten`: "achata" a estrutura hierárquica da política IAM, expandindo a lista `bindings[].members` para que cada membro seja uma linha separada — necessário para filtrar por membro individualmente.
- `--filter`: aplica um filtro de busca no campo indicado; `bindings.role:roles/editor` retorna apenas as entradas com esse papel.
- `--format`: define o formato de saída; `table(bindings.members)` exibe apenas a coluna de membros em formato tabular.

**Resultado esperado:** Novo usuário listado com role "roles/editor".

---

## Validação e Testes

### Verificar todas as VPCs

```bash
gcloud compute networks list
gcloud compute networks subnets list
```

### Verificar bastion com múltiplas NICs

```bash
gcloud compute instances describe "$BASTION_HOST" --zone="$ZONE"
```

### Verificar Cloud SQL

```bash
gcloud sql instances describe "$DEV_SQL_INSTANCE"
gcloud sql databases list --instance="$DEV_SQL_INSTANCE"
```

### Verificar cluster Kubernetes

```bash
kubectl get nodes
kubectl get pods -A
kubectl get svc
```

### Testar conectividade ao WordPress

```bash
EXTERNAL_IP=$(kubectl get svc wordpress \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -I http://$EXTERNAL_IP
```

---

## Troubleshooting

### Cluster não cria pods

```bash
# Verificar recursos disponíveis
kubectl top nodes
kubectl describe nodes

# Ver logs de eventos
kubectl get events -A --sort-by='.lastTimestamp'
```

### WordPress não conecta ao banco

```bash
# Verificar secrets
kubectl get secrets
kubectl describe secret database

# Verificar logs do deployment
kubectl logs -l app=wordpress
```

### Cloud SQL não acessível

```bash
# Verificar instância
gcloud sql instances describe "$DEV_SQL_INSTANCE"

# Verificar firewall
gcloud compute firewall-rules list --filter="network:$DEV_VPC"
```

### LoadBalancer não obtém IP externo

```bash
kubectl describe svc wordpress
kubectl get events
```

---

## Limpeza (Opcional)

```bash
# Deletar Kubernetes resources
kubectl delete deployment wordpress
kubectl delete svc wordpress
kubectl delete secret database cloudsql-instance-credentials

# Deletar cluster
gcloud container clusters delete "$K8S_CLUSTER" --zone="$ZONE" --quiet

# Deletar Cloud SQL
gcloud sql instances delete "$DEV_SQL_INSTANCE" --quiet

# Deletar instâncias
gcloud compute instances delete "$BASTION_HOST" --zone="$ZONE" --quiet

# Deletar firewall rules
gcloud compute firewall-rules delete "allow-ssh-from-iap" --quiet
gcloud compute firewall-rules delete "allow-ssh-from-iap-prod" --quiet

# Deletar VPCs (deleta também subnets)
gcloud compute networks delete "$DEV_VPC" --quiet
gcloud compute networks delete "$PROD_VPC" --quiet

# Limpar arquivos locais
rm -rf wp-k8s/ key.json
```

---

## Conceitos-Chave para Relembrar

| Conceito | O que é | Quando usar |
|---|---|---|
| VPC Custom | Rede virtual com subnets manualmente definidas | Segmentação de ambientes (dev/prod) |
| Bastion Host | VM com acesso administrativo via IAP | Gerenciamento seguro de múltiplas redes |
| Cloud SQL | Banco de dados gerenciado MySQL | Hospedagem de aplicações que exigem banco relacional |
| GKE Cluster | Kubernetes gerenciado no Google Cloud | Containers, microserviços e workloads escaláveis |
| Service Account | Identidade para aplicações | Autenticação segura entre componentes |
| Uptime Check | Monitoramento de disponibilidade | Alertas automáticos de indisponibilidade |
| IAM Role | Controle de acesso granular | Segurança baseada em princípio do menor privilégio |

---

## Fluxo Final da Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                    Google Cloud Project                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────── VPC DEV ───────────────┐                   │
│  │ 192.168.0.0/16                       │                   │
│  │                                      │                   │
│  │  ┌──────────────────────────────┐    │  ┌─────────────┐  │
│  │  │  griffin-dev-wp (WordPress)  │    │  │   Bastion   │  │
│  │  │  192.168.16.0/20             │    │  │  (2 NICs)   │  │
│  │  │  - GKE Cluster (2 nodes)     │    │  │ 192.168.32  │  │
│  │  │  - WordPress Deployment      │    │  │    .10      │  │
│  │  └──────────────────────────────┘    │  └─────────────┘  │
│  │                                      │                   │
│  │  ┌──────────────────────────────┐    │                   │
│  │  │  griffin-dev-mgmt            │    │                   │
│  │  │  192.168.32.0/20             │    │                   │
│  │  │  - Cloud SQL Instance        │    │                   │
│  │  └──────────────────────────────┘    │                   │
│  └──────────────────────────────────────┘                   │
│                                                             │
│  ┌────────────── VPC PROD ──────────────┐                   │
│  │ 192.168.0.0/16                       │                   │
│  │                                      │                   │
│  │  ┌──────────────────────────────┐    │ ┌──────────────┐  │
│  │  │  griffin-prod-wp             │    │ │  Bastion     │  │
│  │  │  192.168.48.0/20             │    │ │   NIC 2      │  │
│  │  └──────────────────────────────┘    │ │ 192.168.64   │  │
│  │                                      │ │    .10       │  │
│  │  ┌──────────────────────────────┐    │ └──────────────┘  │
│  │  │  griffin-prod-mgmt           │    │                   │
│  │  │  192.168.64.0/20             │    │                   │
│  │  └──────────────────────────────┘    │                   │
│  └──────────────────────────────────────┘                   │
│                                                             │
│  Monitoramento: Uptime Check do WordPress Dev               │
│  Acesso: IAM Editor para engenheiro adicional               │
└─────────────────────────────────────────────────────────────┘
```

---

## Resumo de Execução

✅ 1. Definir variáveis de ambiente e configuração padrão
✅ 2. Criar VPC de desenvolvimento com subnets (wordpress + mgmt)
✅ 3. Criar VPC de produção com subnets (wordpress + mgmt)
✅ 4. Criar bastion com 2 NICs conectadas às VPCs
✅ 5. Provisionar Cloud SQL e configurar banco WordPress
✅ 6. Criar cluster GKE na subnet de development WordPress
✅ 7. Preparar secrets e configurações do Kubernetes
✅ 8. Deployar WordPress e criar LoadBalancer
✅ 9. Ativar uptime checks para monitoramento
✅ 10. Conceder acesso IAM ao engenheiro adicional

Todos os 9 tasks foram convertidos em comandos CLI executáveis com explicações de parâmetros e validações de resultado esperado.
