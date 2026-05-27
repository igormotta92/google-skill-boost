# VPC Networks: Controlling Access no GCP

## Introducao

Este documento organiza o laboratorio de VPC Networks com foco em controle de acesso no GCP.
A ideia e praticar:

- criacao de VMs em uma VPC
- liberacao de trafego com firewall rules baseadas em tags
- validacao de conectividade interna e externa
- diferenca de permissoes entre papeis de rede e seguranca
- uso de service account para administrar regras de firewall

---

## TAREFA 1: Criar Servidores Web (blue e green)

### Conceito
Criamos duas instancias na mesma rede VPC. Uma tera tag de servidor web e outra nao, para observar como o firewall aplica regras de forma seletiva.

### Passo 1: Criar VM blue (com tag web-server)

```bash
gcloud compute instances create blue \
    --zone=us-central1-a \
    --network=default \
    --tags=web-server
```

**Por que?**
- A tag `web-server` sera usada para aplicar regra de firewall apenas nesta VM.

### Passo 2: Criar VM green (sem tag web-server)

```bash
gcloud compute instances create green \
    --zone=us-central1-a \
    --network=default
```

**Por que?**
- Serve para comparar comportamento com e sem tag quando a regra de firewall for criada.

### Passo 3: Instalar NGINX e personalizar pagina na VM blue

```bash
gcloud compute ssh blue --zone=us-central1-a

sudo apt-get install nginx-light -y
sudo sed -i 's/<h1>Welcome to nginx!<\/h1>/<h1>Welcome to the blue server!<\/h1>/' /var/www/html/index.nginx-debian.html
cat /var/www/html/index.nginx-debian.html
exit
```

**Por que?**
- Garante que cada servidor tenha uma resposta visual diferente, facilitando o teste.

### Passo 4: Instalar NGINX e personalizar pagina na VM green

```bash
gcloud compute ssh green --zone=us-central1-a

sudo apt-get install nginx-light -y
sudo sed -i 's/<h1>Welcome to nginx!<\/h1>/<h1>Welcome to the green server!<\/h1>/' /var/www/html/index.nginx-debian.html
cat /var/www/html/index.nginx-debian.html
exit
```

**Resultado esperado:**
- As duas VMs respondem HTTP na porta 80 com paginas diferentes.

### Resumo Visual - Tarefa 1

```
[VPC default - us-central1-a]
        |
   -----------------
   |               |
 [blue]          [green]
 tag:web-server  sem tag
 nginx           nginx
```

---

## TAREFA 2: Criar Firewall Rule com Target Tag

### Conceito
A regra de firewall vai liberar HTTP e ICMP apenas para instancias com a tag `web-server`.

### Passo 1: Criar regra de firewall

```bash
gcloud compute firewall-rules create allow-http-web-server \
    --network=default \
    --target-tags=web-server \
    --source-ranges=0.0.0.0/0 \
    --allow=tcp:80,icmp
```

**Por que?**
- `--target-tags=web-server`: aplica a regra so para VMs com essa tag.
- `--allow=tcp:80,icmp`: permite HTTP e ping.

### Passo 2: Criar VM de teste

```bash
gcloud compute instances create test-vm \
    --machine-type=e2-micro \
    --subnet=default \
    --zone=us-central1-a
```

**Por que?**
- Essa VM sera usada para validar conectividade como cliente interno.

### Passo 3: Testar conectividade HTTP

```bash
gcloud compute ssh test-vm --zone=us-central1-a

# IP interno
curl 10.128.0.2
curl 10.128.0.3

# IP externo
curl 35.193.212.252
curl 34.46.136.178
```

**Resultado esperado:**
- Acesso para `blue` deve funcionar (tem tag `web-server`).
- Acesso para `green` pode falhar por nao ter sido alvo da regra.

### Referencia de IPs usada no laboratorio

| Name | Zone | In use by | Internal IP | External IP |
|------|------|-----------|-------------|-------------|
| blue | us-central1-a | nic0 | 10.128.0.2 | 35.193.212.252 |
| green | us-central1-a | nic0 | 10.128.0.3 | 34.46.136.178 |
| test-vm | us-central1-a | nic0 | 10.128.0.4 | 34.67.99.86 |

### Resumo Visual - Tarefa 2

```
[allow-http-web-server]
source: 0.0.0.0/0
allow: tcp:80, icmp
target-tag: web-server
        |
     [blue]  <- recebe trafego liberado
     [green] <- fora da regra (sem tag)
```

---

## TAREFA 3: Explorar Permissoes (Network Admin vs Security Admin)

### Conceito
Nem toda conta pode listar/deletar firewall rules. Nesta tarefa, validamos limites de permissao com a conta padrao e depois com service account dedicada.

### Passo 1: Validar erro de permissao na test-vm

```bash
gcloud compute ssh test-vm --zone=us-central1-a
```

Comandos esperados para falhar:

```bash
gcloud compute firewall-rules list
gcloud compute firewall-rules delete allow-http-web-server
```

**Por que?**
- A service account padrao da VM geralmente nao possui papeis administrativos de rede/seguranca.

### Passo 2: Descobrir service account da test-vm

```bash
SA=$(gcloud compute instances describe test-vm --zone=us-central1-a --format='value(serviceAccounts[0].email)')
echo $SA
```

### Passo 3: Listar roles da service account

```bash
gcloud projects get-iam-policy qwiklabs-gcp-00-f2f77b44739d \
    --flatten="bindings[].members" \
    --filter="bindings.members:serviceAccount:449307102504-compute@developer.gserviceaccount.com" \
    --format="table(bindings.role)"
```

### Passo 4: Checar especificamente roles de rede/seguranca

```bash
gcloud projects get-iam-policy qwiklabs-gcp-00-f2f77b44739d \
    --flatten="bindings[].members" \
    --filter="bindings.members:serviceAccount:449307102504-compute@developer.gserviceaccount.com AND bindings.role:(roles/compute.networkAdmin OR roles/compute.securityAdmin)" \
    --format="table(bindings.role)"
```

**Resultado esperado:**
- Confirmar ausencia (ou insuficiencia) de papeis necessarios para administrar firewall.

---

## TAREFA 4: Criar e Usar Service Account Administrativa

### Conceito
Criamos uma service account dedicada para administrar rede. Primeiro adicionamos `Network Admin`, depois `Security Admin` para liberar acao de delete em firewall rule.

### Passo 1: Criar service account

```bash
gcloud iam service-accounts create Network-admin \
    --display-name="Network Admin Service Account"
```

### Passo 2: Criar chave da service account

```bash
gcloud iam service-accounts keys create credentials.json \
    --iam-account=Network-admin@qwiklabs-gcp-00-f2f77b44739d.iam.gserviceaccount.com
```

### Passo 3: Copiar chave para test-vm e autenticar

```bash
gcloud compute scp credentials.json test-vm:/tmp/ --zone=us-central1-a

gcloud compute ssh test-vm --zone=us-central1-a
gcloud auth activate-service-account --key-file /tmp/credentials.json
```

### Passo 4: Adicionar role Network Admin

```bash
gcloud projects add-iam-policy-binding qwiklabs-gcp-00-f2f77b44739d \
    --member=serviceAccount:Network-admin@qwiklabs-gcp-00-f2f77b44739d.iam.gserviceaccount.com \
    --role=roles/compute.networkAdmin
```

Teste esperado:

```bash
gcloud compute firewall-rules list
```

Comportamento esperado apos `Network Admin`:
- listar regras deve funcionar
- deletar regra ainda pode falhar

### Passo 5: Adicionar role Compute Security Admin

```bash
gcloud projects add-iam-policy-binding qwiklabs-gcp-00-f2f77b44739d \
    --member=serviceAccount:Network-admin@qwiklabs-gcp-00-f2f77b44739d.iam.gserviceaccount.com \
    --role=roles/compute.securityAdmin
```

Teste esperado:

```bash
gcloud compute firewall-rules delete allow-http-web-server
# [Y]
```

### Passo 6: Verificar efeito da remocao da regra

```bash
curl -c 3 35.193.212.252
```

**Resultado esperado:**
- Apos remover a regra, o acesso HTTP antes permitido deve parar de funcionar conforme politica de firewall.

---

## Comandos Uteis para Troubleshooting

### Listar regras e instancias

```bash
gcloud compute firewall-rules list
gcloud compute instances list
```

### Ver detalhes de uma instancia

```bash
gcloud compute instances describe test-vm --zone=us-central1-a
```

### Ver autenticacao ativa

```bash
gcloud auth list
gcloud config list account
```

### Deletar service account (se necessario)

```bash
gcloud iam service-accounts delete Network-admin@qwiklabs-gcp-00-f2f77b44739d.iam.gserviceaccount.com
```

---

## Conceitos-Chave para Relembrar

| Conceito | O que e | Quando usar |
|----------|---------|-------------|
| **Firewall Rule** | Regra que permite/bloqueia trafego | Controle de acesso em rede |
| **Target Tag** | Filtro para aplicar regra em VMs especificas | Segmentar acesso sem usar IP fixo |
| **Source Ranges** | Origem permitida para trafego | Restringir quem pode acessar |
| **Service Account** | Identidade de maquina/aplicacao | Automacao e acesso autenticado ao GCP |
| **Network Admin** | Administra recursos de rede | Operacoes gerais de rede |
| **Security Admin** | Administra politicas de seguranca | Operacoes sensiveis como firewall |

---

## Diferenca: Network Admin vs Security Admin

| Aspecto | Network Admin | Security Admin |
|--------|----------------|----------------|
| **Foco** | Gerenciamento de rede | Gerenciamento de seguranca |
| **Listar firewall rules** | Geralmente sim | Sim |
| **Alterar/excluir firewall rules** | Pode ser limitado | Sim, com privilegios de seguranca |
| **Uso comum** | Operacao de rede | Governanca e controle de acesso |
