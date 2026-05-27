# GCP - Guia Completo de Serviços e Configurações

## Introdução

Este documento fornece um guia prático e estruturado para trabalhar com os principais serviços do Google Cloud Platform (GCP). Cada seção apresenta conceitos, passos de configuração, exemplos práticos e explicações do "por quê" de cada ação.

### Serviços Cobertos

- **Cloud Storage**: Armazenamento de objetos em buckets
- **IAM**: Gerenciamento de identidades e acessos
- **Cloud Monitoring**: Monitoramento de recursos e coleta de métricas
- **Cloud Run**: Execução de funções serverless
- **Cloud Pub/Sub**: Sistema de mensageria publish-subscribe

---

## SERVIÇO 1: Cloud Storage (Armazenamento em Nuvem)

### Conceito
Cloud Storage permite armazenar dados estruturados e não-estruturados em buckets (contêineres). Os objetos são acessíveis via CLI, SDK ou Console do GCP. Ideal para backups, arquivos estáticos e data lakes.

### Passo 1: Criar um Bucket

```bash
gcloud storage buckets create gs://qwiklabs-gcp-03-42477ee9628d
```

**Por quê?**
- Bucket é a unidade de armazenamento no Cloud Storage
- Precisa de um nome globalmente único (similar a domínios na internet)
- Todos os objetos são armazenados dentro de buckets

**Resultado esperado:**
```
Creating gs://qwiklabs-gcp-03-42477ee9628d/
Bucket [gs://qwiklabs-gcp-03-42477ee9628d] created.
```

### Passo 2: Fazer Upload de um Arquivo Local

```bash
gcloud storage cp ada.jpg gs://qwiklabs-gcp-03-42477ee9628d
```

**Por quê?**
- `cp` é o comando de cópia (similar a `cp` do Linux)
- Arquivo local `ada.jpg` é enviado para o bucket
- Especificar caminho completo: `gs://nome-bucket/caminho`

**Resultado esperado:**
```
Copying file://./ada.jpg to gs://qwiklabs-gcp-03-42477ee9628d/ada.jpg
  Completed 1/1
```

### Passo 3: Listar Arquivos no Bucket

```bash
gcloud storage ls -l gs://qwiklabs-gcp-03-42477ee9628d/ada.jpg
```

**Por quê?**
- Verifica se o arquivo foi enviado com sucesso
- `-l` mostra informações detalhadas (tamanho, data, etc.)

**Resultado esperado:**
```
   1024  2026-05-13T10:15:30Z  gs://qwiklabs-gcp-03-42477ee9628d/ada.jpg
```

### Passo 4: Fazer Download de Arquivo do Bucket

```bash
gcloud storage cp gs://qwiklabs-gcp-03-42477ee9628d/ada.jpg ./local-ada.jpg
```

**Por quê?**
- Baixa arquivo do bucket para o diretório local
- Salva com nome diferente se desejado

### Passo 5: Copiar Arquivo dentro do Bucket (em Subdiretório)

```bash
gcloud storage cp gs://qwiklabs-gcp-03-42477ee9628d/ada.jpg gs://qwiklabs-gcp-03-42477ee9628d/image-folder/
```

**Por quê?**
- Reorganiza arquivos dentro do bucket
- Cria hierarquia de pastas (embora Cloud Storage não tenha diretórios reais, simula com prefixos)

### Passo 6: Copiar Recursivamente (Diretório Inteiro)

```bash
gcloud storage cp -r gs://qwiklabs-gcp-03-42477ee9628d ./local-backup
```

**Por quê?**
- `-r` copia recursivamente (inclui subdiretórios)
- Útil para backups de múltiplos arquivos

### Passo 7: Gerenciar Permissões - Fazer Público

```bash
gcloud storage objects update gs://qwiklabs-gcp-03-42477ee9628d/ada.jpg \
  --add-acl-grant=entity=allUsers,role=READER
```

**Por quê?**
- Permite qualquer pessoa na internet acessar o arquivo
- `allUsers` = qualquer usuário autenticado
- `role=READER` = apenas leitura
- Sem isso, só conta GCP proprietária acessa

**Resultado esperado:**
```
ada.jpg is now readable by allUsers
```

### Passo 8: Remover Permissões Públicas

```bash
gcloud storage objects update gs://qwiklabs-gcp-03-42477ee9628d/ada.jpg \
  --remove-acl-grant=allUsers
```

**Por quê?**
- Desfaz permissões públicas
- Volta à privacidade padrão

### Passo 9: Deletar um Arquivo

```bash
gcloud storage rm gs://qwiklabs-gcp-03-42477ee9628d/ada.jpg
```

**Por quê?**
- Remove arquivo do bucket
- `rm` = remove (similar a `rm` do Linux)

### Resumo Visual - Cloud Storage

```
[Máquina Local]
       ↓
[gcloud storage cp]
       ↓
[Google Cloud Storage (Bucket)]
       ↓
[gs://bucket-name/objeto]
       ↓
[Objetos com ACLs (permissões)]
```

**Fluxo Típico:**
1. Criar bucket → 2. Upload de arquivos → 3. Definir permissões → 4. Compartilhar ou usar

---

## SERVIÇO 2: IAM (Gerenciamento de Identidades e Acessos)

### Conceito
IAM controla quem pode acessar quais recursos do GCP. Define roles (papéis) e bindings (atribuições) para usuários, grupos de serviço e service accounts.

### Passo 1: Listar Buckets (Verificar Acesso)

```bash
gsutil ls gs://qwiklabs-gcp-02-816e2a178921
```

**Por quê?**
- `gsutil` é a ferramenta legada de Storage (ainda funciona)
- Listar buckets verifica se sua conta tem permissão de acesso
- Se não tiver permissão, retorna erro de acesso negado

**Ou com gcloud (novo):**
```bash
gcloud storage ls gs://qwiklabs-gcp-02-816e2a178921
```

### Conceitos-Chave de IAM

| Conceito | O quê é | Exemplo |
|----------|---------|---------|
| **Principal** | Quem está tentando acessar | Usuário, Service Account, Grupo |
| **Role (Papel)** | Conjunto de permissões | `roles/viewer`, `roles/editor`, `roles/owner` |
| **Binding** | Atribuição: Principal + Role | "Igor tem role Editor em projeto X" |
| **Service Account** | Identidade para aplicações | Para que apps façam requisições autenticadas |

### Passo 2: Visualizar Política IAM de um Projeto

```bash
gcloud projects get-iam-policy PROJECT_ID
```

**Por quê?**
- Mostra todos os bindings (quem tem qual role)
- Ajuda a auditar permissões

### Passo 3: Conceder Role a um Usuário

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:usuario@example.com \
  --role=roles/compute.admin
```

**Por quê?**
- Usuário agora tem permissão de administrador de Compute Engine
- `--member`: Define o principal (pode ser user:, serviceAccount:, group:)
- `--role`: Define qual role

### Passo 4: Revogar Role de um Usuário

```bash
gcloud projects remove-iam-policy-binding PROJECT_ID \
  --member=user:usuario@example.com \
  --role=roles/compute.admin
```

**Por quê?**
- Remove acesso do usuário

### Roles Mais Comuns

| Role | Permissões | Use Case |
|------|-----------|----------|
| **roles/viewer** | Apenas leitura | Visualizar recursos |
| **roles/editor** | Criar, editar, deletar | Desenvolvimento |
| **roles/owner** | Tudo + gerenciar IAM | Administração |
| **roles/compute.admin** | Gerenciar VMs | DevOps |
| **roles/storage.admin** | Gerenciar buckets | Data engineer |

---

## SERVIÇO 3: Cloud Monitoring (Monitoramento e Métricas)

### Conceito
Cloud Monitoring coleta métricas de recursos (VMs, bancos de dados, etc.) e permite criar dashboards, alertas e monitorar performance em tempo real.

### Passo 1: Configurar Região e Zona Padrão

```bash
gcloud config set compute/zone "us-east4-c"
gcloud config set compute/region "us-east4"

export ZONE=$(gcloud config get compute/zone)
export REGION=$(gcloud config get compute/region)
```

**Por quê?**
- Define localização padrão para evitar repetir `--zone` e `--region`
- Exportar para variáveis torna scripts mais reutilizáveis

### Passo 2: Criar Instância de VM para Monitorar

```bash
gcloud compute instances create lamp-1-vm \
    --zone=$ZONE \
    --machine-type=e2-medium \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --tags=http-server \
    --metadata=startup-script='#!/bin/bash
        sudo apt-get update
        sudo apt-get install -y apache2 php7.0
        sudo service apache2 restart'
```

**Por quê?**
- Cria VM com Apache2 (servidor web)
- Startup script instala dependências automaticamente
- VM será monitorada pelo Cloud Monitoring

**Resultado esperado:**
```
Created [https://www.googleapis.com/compute/v1/projects/.../zones/us-east4-c/instances/lamp-1-vm].
```

### Passo 3: Conectar via SSH

```bash
gcloud compute ssh lamp-1-vm --zone=$ZONE
```

**Por quê?**
- Acessa a VM via linha de comando de forma segura
- Permite executar comandos remotamente

### Passo 4: Instalar Agentes de Monitoramento

```bash
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
```

**Por quê?**
- Google Cloud Ops Agent coleta métricas e logs
- Envia dados para Cloud Monitoring e Cloud Logging
- Sem esse agente, GCP só coleta métricas básicas

**Resultado esperado:**
```
Successfully installed and started Google Cloud Ops Agent
```

### Passo 5: Verificar Status do Agente

```bash
sudo systemctl status google-cloud-ops-agent"*"
```

**Por quê?**
- Confirma que o agente está rodando
- Se não estiver, investigar logs

### Passo 6: Atualizar Pacotes

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

**Por quê?**
- Garante que o agente está com patches de segurança

### Resumo Visual - Cloud Monitoring

```
[VM (lamp-1-vm)]
    ↓
[Google Cloud Ops Agent]
    ↓
[Coleta métricas: CPU, memória, disco, rede]
    ↓
[Cloud Monitoring (backend)]
    ↓
[Dashboards + Alertas + Gráficos]
```

**Fluxo:**
1. VM com agente instalado → 2. Agente envia métricas → 3. Visualizar em dashboards → 4. Configurar alertas

---

## SERVIÇO 4: Cloud Run Function (Funções Serverless)

### Conceito
Cloud Run executa código (geralmente em containers) sem gerenciar infraestrutura. Ideal para APIs serverless, webhooks e processamento de eventos. Cobra apenas pelo tempo de execução.

### Passo 1: Configurar Região Padrão

```bash
gcloud config set run/region europe-west3
```

**Por quê?**
- Define região padrão para Cloud Run
- Evita especificar `--region` em cada comando

### Passo 2: Criar Diretório do Projeto

```bash
mkdir gcf_hello_world && cd $_
```

**Por quê?**
- Organiza arquivos do projeto
- `cd $_` entra no diretório recém-criado

### Passo 3: Criar Arquivo JavaScript (index.js)

```bash
nano index.js
```

**Conteúdo:**
```javascript
const functions = require('@google-cloud/functions-framework');

// Register a CloudEvent callback com Pub/Sub trigger
functions.cloudEvent('helloPubSub', cloudEvent => {
  // Mensagem Pub/Sub é passada como payload do CloudEvent
  const base64name = cloudEvent.data.message.data;

  const name = base64name
    ? Buffer.from(base64name, 'base64').toString()
    : 'World';

  console.log(`Hello, ${name}!`);
});
```

**Por quê?**
- Define função que responde a eventos Pub/Sub
- Mensagens chegam em base64, decodificar para string
- Framework Google Cloud Functions automatiza setup

### Passo 4: Criar package.json (Dependências)

```bash
nano package.json
```

**Conteúdo:**
```json
{
  "name": "gcf_hello_world",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    "@google-cloud/functions-framework": "^3.0.0"
  }
}
```

**Por quê?**
- Define dependências do projeto
- Framework Google Cloud Functions fornece trigger handling

### Passo 5: Instalar Dependências

```bash
npm install
```

**Por quê?**
- Baixa pacotes listados no package.json
- Cria `node_modules/` com toda dependência

### Passo 6: Fazer Deploy da Função

```bash
gcloud functions deploy nodejs-pubsub-function \
  --gen2 \
  --runtime=nodejs22 \
  --region=europe-west3 \
  --source=. \
  --entry-point=helloPubSub \
  --trigger-topic cf-demo \
  --stage-bucket qwiklabs-gcp-02-891f6e0ec0e5-bucket \
  --service-account cloudfunctionsa@qwiklabs-gcp-02-891f6e0ec0e5.iam.gserviceaccount.com \
  --allow-unauthenticated
```

**Explicação dos parâmetros:**
- `--gen2`: Usa segunda geração do Cloud Functions (mais flexível)
- `--runtime=nodejs22`: Linguagem e versão
- `--region`: Região de deployment
- `--source=.`: Código-fonte no diretório atual
- `--entry-point=helloPubSub`: Nome da função a executar
- `--trigger-topic cf-demo`: Trigger via Pub/Sub (se mensagem publicada em `cf-demo`, executa)
- `--stage-bucket`: Bucket temporário para staging
- `--allow-unauthenticated`: Qualquer pessoa pode chamar (via HTTP GET/POST)

**Resultado esperado:**
```
Deploying function...
Creating deployment...
Waiting for deployment...
Created successfully.
```

### Passo 7: Verificar Detalhes da Função

```bash
gcloud functions describe nodejs-pubsub-function \
  --region=europe-west3
```

**Por quê?**
- Mostra status, triggers, URL de invocação, etc.

### Passo 8: Testar Publicando Mensagem Pub/Sub

```bash
gcloud pubsub topics publish cf-demo --message="Cloud Function Gen2"
```

**Por quê?**
- Publica mensagem no tópico `cf-demo`
- Cloud Function foi subscrita a este tópico, então executa

### Passo 9: Ver Logs da Função

```bash
gcloud functions logs read nodejs-pubsub-function \
  --region=europe-west3 --limit=50
```

**Por quê?**
- Mostra últimos 50 logs (saída de `console.log`)
- Ajuda a debugar se algo deu errado

**Ou com logging mais detalhado:**
```bash
gcloud logging tail 'resource.type="cloud_function" AND resource.labels.function_name="nodejs-pubsub-function" AND resource.labels.region="europe-west3"'
```

### Resumo Visual - Cloud Run

```
[Código Local (Node.js)]
       ↓
[gcloud functions deploy]
       ↓
[Cloud Run (Serverless)]
       ↓
[Pub/Sub Topic (cf-demo)]
       ↓
[Mensagem Publicada]
       ↓
[Cloud Function Executada]
       ↓
[Log Output → Cloud Logging]
```

**Fluxo:**
1. Escrever código → 2. Deploy → 3. Função registra em Pub/Sub → 4. Publica mensagem → 5. Função executa

---

## SERVIÇO 5: Cloud Pub/Sub (Mensageria)

### Conceito
Pub/Sub implementa padrão publish-subscribe desacoplado: publishers enviam mensagens para tópicos, subscribers recebem de subscrições. Ideal para arquiteturas event-driven.

### Passo 1: Criar um Tópico

```bash
gcloud pubsub topics create myTopic
```

**Por quê?**
- Tópico é o "canal" para mensagens
- Publishers publicam neste tópico
- Subscribers se inscrevem em subscrições vinculadas a este tópico

**Resultado esperado:**
```
Created topic [projects/qwiklabs-gcp-xxxxx/topics/myTopic].
```

### Passo 2: Criar Múltiplos Tópicos

```bash
gcloud pubsub topics create Test1
gcloud pubsub topics create Test2
```

**Por quê?**
- Permite organizar mensagens por categoria/tipo de evento

### Passo 3: Listar Todos os Tópicos

```bash
gcloud pubsub topics list
```

**Resultado esperado:**
```
NAME: projects/qwiklabs-gcp-xxxxx/topics/Test1
NAME: projects/qwiklabs-gcp-xxxxx/topics/Test2
NAME: projects/qwiklabs-gcp-xxxxx/topics/myTopic
```

### Passo 4: Deletar Tópicos Desnecessários

```bash
gcloud pubsub topics delete Test1
gcloud pubsub topics delete Test2
```

**Por quê?**
- Remove tópicos não usados
- Reduz clutter(bagunça) e potenciais custos

### Passo 5: Criar Subscrições

```bash
gcloud pubsub subscriptions create mySubscription --topic myTopic
gcloud pubsub subscriptions create Test1 --topic myTopic
gcloud pubsub subscriptions create Test2 --topic myTopic
```

**Por quê?**
- Subscrição permite que um subscriber receba mensagens de um tópico
- Um tópico pode ter múltiplas subscrições (cada uma recebe cópia das mensagens)
- Múltiplas subscrições = múltiplos "observadores" do mesmo tópico

### Passo 6: Listar Subscrições de um Tópico

```bash
gcloud pubsub topics list-subscriptions myTopic
```

**Resultado esperado:**
```
mySubscription
Test1
Test2
```

### Passo 7: Deletar Subscrições

```bash
gcloud pubsub subscriptions delete Test1
gcloud pubsub subscriptions delete Test2
```

**Por quê?**
- Remove subscrições não usadas

### Passo 8: Publicar Mensagens

```bash
gcloud pubsub topics publish myTopic --message "Hello"
gcloud pubsub topics publish myTopic --message "Publisher's name is Igor"
gcloud pubsub topics publish myTopic --message "Publisher likes to eat hamburguer"
gcloud pubsub topics publish myTopic --message "Publisher thinks Pub/Sub is awesome"
```

**Por quê?**
- Envia mensagens para o tópico
- Todas as subscrições recebem cópia de cada mensagem
- Mensagens ficam no buffer até serem consumidas

### Passo 9: Puxar Mensagens (Pull)

```bash
gcloud pubsub subscriptions pull mySubscription --auto-ack
```

**Por quê?**
- Subscriber puxa mensagens manualmente
- `--auto-ack` reconhece (acknowledges) automaticamente (marca como consumida)
- Sem `--auto-ack`, mensagem fica pendente

**Resultado esperado:**
```
┌─────────────────────────────────────┬──────────────┐
│ DATA                                │ MESSAGE_ID   │
├─────────────────────────────────────┼──────────────┤
│ Hello                               │ 123456789    │
│ Publisher's name is Igor            │ 123456790    │
└─────────────────────────────────────┴──────────────┘
```

### Passo 10: Publicar Mais Mensagens

```bash
gcloud pubsub topics publish myTopic --message "Publisher is starting to get the hang of Pub/Sub"
gcloud pubsub topics publish myTopic --message "Publisher wonders if all messages will be pulled"
gcloud pubsub topics publish myTopic --message "Publisher will have to test to find out"
```

### Passo 11: Puxar com Limite

```bash
gcloud pubsub subscriptions pull mySubscription --limit=3
gcloud pubsub subscriptions pull mySubscription --auto-ack --limit=3
```

**Por quê?**
- `--limit=3`: Puxa apenas 3 mensagens por vez
- Útil para processar em batches

### Passo 12: Usar com Python (Cloud Sheel)

```bash
sudo apt-get install -y virtualenv
python3 -m venv venv
source venv/bin/activate

pip install --upgrade google-cloud-pubsub
```

**Por quê?**
- Cria ambiente virtual isolado
- Instala SDK Python do Pub/Sub

### Passo 13: Clonar Exemplos e Testar

```bash
git clone https://github.com/googleapis/python-pubsub.git
cd python-pubsub/samples/snippets

echo $GOOGLE_CLOUD_PROJECT

# Criar tópico
python publisher.py $GOOGLE_CLOUD_PROJECT create MyTopic

# Listar tópicos
python publisher.py $GOOGLE_CLOUD_PROJECT list

# Criar subscrição
python subscriber.py $GOOGLE_CLOUD_PROJECT create MyTopic MySub

# Listar subscrições
python subscriber.py $GOOGLE_CLOUD_PROJECT list-in-project

# Ajuda
python subscriber.py -h
```

**Por quê?**
- Scripts Python automatizam operações Pub/Sub
- Exemplos do repo oficial mostram melhores práticas

### Passo 14: Publicar via CLI e Receber via Python

```bash
# Em um terminal: iniciar subscriber
python subscriber.py $GOOGLE_CLOUD_PROJECT receive MySub

# Em outro terminal: publicar mensagens
gcloud pubsub topics publish MyTopic --message "Hello from CLI"
gcloud pubsub topics publish MyTopic --message "Another message"
```

**Por quê?**
- Subscriber fica escutando (blocking)
- Cada vez que mensagem é publicada, subscriber recebe em tempo real (push)

### Resumo Visual - Pub/Sub

```
[Publisher]          [Tópico myTopic]          [Subscriber]
    ↓                       ↑                         ↓
[msg: "Hello"]   ────→  [Buffer de Msgs]   ────→  [Recebe msg]
[msg: "World"]   ────→                      ────→  [Processa]
                        [Múltiplas Subs] ──────→  [Sub-2]
                        (cada recebe cópia)
```

**Fluxo Pub/Sub:**
1. Publisher → Publica mensagem no tópico
2. Mensagem fica buffered no tópico
3. Todas as subscrições recebem cópia
4. Subscriber processa e reconhece (ack)

### Comparação: Push vs Pull

| Aspecto | Push | Pull |
|--------|------|------|
| **Iniciativa** | GCP envia para subscriber | Subscriber pede mensagens |
| **Latência** | Imediata (quase real-time) | Depende de polling |
| **Webhook** | Requer endpoint HTTP | Não precisa |
| **Reliability** | GCP roteia automaticamente | Subscriber controla retry |
| **Use Case** | Cloud Functions, Apps | Scripts em batch |

Para um detalhamento de como o SDK escuta mensagens em produção (incluindo `streaming_pull_future`, gRPC em streaming e padrões para evitar travas), veja [Apendice A: Streaming Pull com SDK gRPC](#apendice-a-streaming-pull-com-sdk-grpc).

---

## Comandos Úteis para Troubleshooting

### Cloud Storage
```bash
# Listar buckets
gcloud storage buckets list

# Ver detalhes de um bucket
gcloud storage buckets describe gs://nome-bucket

# Deletar bucket vazio
gcloud storage buckets delete gs://nome-bucket

# Listar todos os arquivos (recursivo)
gcloud storage ls -r gs://nome-bucket

# Ver classe de armazenamento
gcloud storage ls -L gs://nome-bucket
```

### Cloud Monitoring
```bash
# Listar instâncias de VM
gcloud compute instances list

# Ver logs de startup script
gcloud compute instances get-serial-port-output INSTANCE_NAME --zone=ZONE

# Verificar firewall rules
gcloud compute firewall-rules list
```

### Cloud Run
```bash
# Listar todas as funções
gcloud functions list

# Ver logs em tempo real
gcloud functions logs read FUNCTION_NAME --limit=50 --follow

# Deletar uma função
gcloud functions delete FUNCTION_NAME --gen2
```

### Pub/Sub
```bash
# Listar todos os tópicos
gcloud pubsub topics list

# Listar todas as subscrições
gcloud pubsub subscriptions list

# Ver detalhes de uma subscrição
gcloud pubsub subscriptions describe SUBSCRIPTION_NAME

# Deletar tópico
gcloud pubsub topics delete TOPIC_NAME

# Purgar mensagens (esvaziar subscrição)
gcloud pubsub subscriptions seek SUBSCRIPTION_NAME --time=2026-01-01T00:00:00Z
```

---

## Conceitos-Chave para Relembrar

| Conceito | O quê é | Quando usar |
|----------|---------|-------------|
| **Bucket** | Unidade de armazenamento | Guardar arquivos |
| **ACL (Access Control List)** | Controle de acesso | Definir permissões públicas/privadas |
| **Service Account** | Identidade para apps | Apps fazendo requisições autenticadas |
| **Role** | Conjunto de permissões | Controle de acesso (IAM) |
| **Tópico** | Canal de mensagens | Pub/Sub - ponto de publicação |
| **Subscrição** | Receptor de mensagens | Pub/Sub - ponto de consumo |
| **Cloud Function** | Função serverless | Executar código sem infra |
| **Ops Agent** | Agente de monitoramento | Enviar métricas para Cloud Monitoring |

---

## Fluxos de Integração Comuns

### Fluxo 1: Upload → Cloud Storage → Pub/Sub → Cloud Run
```
[Local]
   ↓
[Upload via gcloud storage cp]
   ↓
[Cloud Storage Bucket]
   ↓
[Event Notification → Pub/Sub]
   ↓
[Cloud Function (disparada por Pub/Sub)]
   ↓
[Processa arquivo]
```

### Fluxo 2: Monitorar VM → Alertas → Cloud Run
```
[VM com Ops Agent]
   ↓
[Envia métricas para Cloud Monitoring]
   ↓
[Alert Policy (ex: CPU > 80%)]
   ↓
[Notificação via Pub/Sub]
   ↓
[Cloud Function responde (escala, notifica, etc.)]
```

### Fluxo 3: Pub/Sub com Múltiplos Subscribers
```
[Publisher]
   ↓
[Tópico]
   ├─→ [Subscrição A] → [Cloud Function A]
   ├─→ [Subscrição B] → [Cloud Function B]
   └─→ [Subscrição C] → [Python Script]
```

Cada subscrição recebe cópia independente, permitindo múltiplos processadores.

---

## Dicas de Performance e Custo

1. **Cloud Storage**: Use classe de armazenamento apropriada (Standard → Nearline → Coldline → Archive conforme acesso)
2. **Cloud Run**: Paga por tempo de execução; optimize memory/CPU conforme necessidade
3. **Pub/Sub**: Mensagens sem limite; preço por 1M de mensagens
4. **Cloud Monitoring**: Primeira 150MB de logs grátis; além disso, custos adicionais são aplicados com base no volume de dados armazenados e analisados.
5. **Pub/Sub + Cloud Run**: Combine triggers para pipelines serverless

---

## Apendice A: Streaming Pull com SDK gRPC

### Conceito
Quando usamos o SDK do Google Cloud Pub/Sub (ex.: Python) com `streaming_pull_future`, o cliente **nao faz polling simples de segundo em segundo** como padrao. Em vez disso, ele usa **gRPC StreamingPull**, mantendo uma conexao de streaming aberta com o servico para receber mensagens de forma continua e com menor latencia.

### Como funciona por baixo dos panos
1. O cliente abre um stream gRPC bidirecional (`StreamingPull`).
2. O servidor envia mensagens conforme ficam disponiveis.
3. O cliente envia `ack`/`nack` e extensao de lease no mesmo canal.
4. Se houver falha de rede ou encerramento do stream, o SDK tenta reconectar automaticamente (com backoff).

### Diferenca pratica: Polling vs StreamingPull
| Modo | Comportamento | Latencia | Observacao |
|------|----------------|----------|------------|
| **Pull (CLI/manual)** | Cliente faz requisicoes periodicas | Maior | Mais simples para testes e batch |
| **StreamingPull (SDK)** | Conexao gRPC aberta em stream continuo | Menor | Melhor para consumidores de longa duracao |

### Exemplo robusto (Python)
```python
from concurrent.futures import TimeoutError
from google.cloud import pubsub_v1
import logging

project_id = "SEU_PROJETO"
subscription_id = "SUA_SUBSCRICAO"

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path(project_id, subscription_id)

flow_control = pubsub_v1.types.FlowControl(
  max_messages=50,
  max_bytes=10 * 1024 * 1024,
)

def callback(message: pubsub_v1.subscriber.message.Message) -> None:
  try:
    # Mantenha o callback rapido; mova trabalho pesado para outro worker.
    logging.info("msg_id=%s size=%d", message.message_id, len(message.data))
    message.ack()
  except Exception:
    logging.exception("Falha ao processar mensagem")
    message.nack()

streaming_pull_future = subscriber.subscribe(
  subscription_path,
  callback=callback,
  flow_control=flow_control,
)

logging.info("Escutando em %s", subscription_path)

try:
  # Mantem o processo ativo; timeout ajuda em observabilidade e reciclagem controlada.
  streaming_pull_future.result(timeout=600)
except TimeoutError:
  logging.warning("Timeout de observacao; reiniciando stream de forma controlada")
  streaming_pull_future.cancel()
  streaming_pull_future.result()
except KeyboardInterrupt:
  streaming_pull_future.cancel()
  streaming_pull_future.result()
```

### Boas praticas para evitar travas
1. Deixe o callback curto e idempotente.
2. Sempre trate excecao e finalize com `ack()` ou `nack()`.
3. Ajuste `flow_control` para evitar saturacao.
4. Defina timeout/retry em chamadas externas feitas dentro do callback.
5. Implemente shutdown gracioso com `streaming_pull_future.cancel()`.
6. Monitore taxa de `nack`, backlog e idade das mensagens.
