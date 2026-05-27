---

# Protocolos

## Hierarquia de Protocolos

```
┌─────────────────────────────────────┐
│ HTTP / HTTPS  (Layer 7 - Aplicação) │
├─────────────────────────────────────┤
│ TCP / UDP     (Layer 4 - Transporte)│
├─────────────────────────────────────┤
│ IP            (Layer 3 - Rede)      │
├─────────────────────────────────────┤
│ Ethernet      (Layer 2 - Link)      │
└─────────────────────────────────────┘
```

## TCP (Layer 4 - Transporte)

**O quê é:**
- Protocolo de **transporte** que garante entrega confiável de dados
- Define **como** os dados viajam pela rede
- Estabelece conexão, envia dados, fecha conexão

**Exemplo:**
```
Seu navegador ←→ Internet ←→ Servidor
(TCP cria o "tubo" confiável entre os dois)
```

**Analogia:** TCP é como a **estrada e as regras de trânsito** que garantem você chegar ao seu destino com segurança.

---

## HTTP (Layer 7 - Aplicação)

**O quê é:**
- Protocolo de **aplicação** que define o formato das mensagens
- Define **o quê** e **como** se comunica
- Usa TCP por baixo para transportar os dados

**Exemplo:**
```
GET /index.html HTTP/1.1
Host: google.com

(Essa mensagem é HTTP, mas viaja por TCP)
```

**Analogia:** HTTP é como a **linguagem e as palavras** que você usa para pedir algo (GET, POST, etc).

---

## Analogia: Carta pelo Correio

```
HTTP = O que você escreve na carta
       ↓
TCP = O envelope que protege a carta
      ↓
IP = O endereço de entrega
     ↓
Ethernet = O camião que entrega
```

---

## Fluxo Real - Quando você acessa google.com

```
1. Você digita: google.com
   ↓
2. HTTP cria a requisição:
   GET / HTTP/1.1
   Host: google.com
   ↓
3. TCP encapsula essa requisição:
   - Estabelece conexão (handshake)
   - Envia os dados
   - Garante entrega
   ↓
4. IP rota para o IP correto
   ↓
5. Servidor recebe
   ↓
6. Servidor responde com HTTP:
   HTTP/1.1 200 OK
   Content-Type: text/html
   <html>...</html>
   ↓
7. TCP traz a resposta de volta
   ↓
8. Navegador interpreta o HTML e mostra a página
```

---

## Diferença Prática

| Aspecto | TCP | HTTP |
|--------|-----|------|
| **Camada** | Layer 4 (Transporte) | Layer 7 (Aplicação) |
| **Responsabilidade** | Entregar dados com segurança | Formato da conversa web |
| **Exemplo** | Porta 80 ou 443 | GET, POST, PUT, DELETE |
| **Protocolo** | TCP/IP | Baseado em TCP |
| **Use** | Qualquer coisa que precisa confiabilidade | Apenas para web |

---

## Em termos de Load Balancer

```bash
# Network Load Balancer (NLB) trabalha com TCP
# → Só vê: "cliente conectando na porta 80"
# → Não entende HTTP

gcloud compute forwarding-rules create nlb \
    --ports 80 \
    --target-pool web-pool
# O NLB é "cego" para HTTP, só roteia TCP


# HTTP Load Balancer (ALB) trabalha com HTTP
# → Vê: "cliente pedindo GET /api/users"
# → Entende rotas HTTP

gcloud compute forwarding-rules create alb \
    --target-http-proxy http-proxy
# ALB consegue ler HTTP e rotear por URL
```

---

## Resumo Simples

- **TCP** = "Como os dados viajam" (garantia de entrega)
- **HTTP** = "O que os dados dizem" (formato das requisições web)
- **HTTP usa TCP** para viajar pela internet

Quando você acessa um site:
1. HTTP cria a mensagem ("Quero a página /index.html")
2. TCP encapsula e entrega com segurança
3. Você vê a página no navegador

---

## Protocolos que usam TCP

TCP é usado por **muitos protocolos** que precisam de entrega garantida. Aqui estão os principais:

| Protocolo | Porta | O quê é | Uso |
|-----------|-------|--------|-----|
| **SSH** | 22 | Acesso remoto seguro | Terminal remoto, SCP |
| **FTP** | 20, 21 | Transferência de arquivos | Upload/download de arquivos |
| **SMTP** | 25, 587 | Enviar emails | Servidor de emails |
| **POP3** | 110 | Receber emails | Baixar emails |
| **IMAP** | 143 | Receber emails | Sincronizar emails |
| **DNS** | 53* | Resolver domínios | Converter google.com → IP |
| **Telnet** | 23 | Terminal remoto (inseguro) | Acesso remoto antigo |
| **SMTP** | 25 | Enviar emails | Servidor de emails |
| **MySQL** | 3306 | Banco de dados | Conexão com MySQL |
| **PostgreSQL** | 5432 | Banco de dados | Conexão com PostgreSQL |
| **MongoDB** | 27017 | Banco NoSQL | Banco de dados |
| **Redis** | 6379 | Cache em memória | Armazenamento rápido |
| **HTTPS** | 443 | Web seguro | Versão segura do HTTP |

*DNS usa tanto TCP quanto UDP

---

## Exemplos Práticos

### SSH (Secure Shell)

```bash
# Conectar a um servidor remoto
ssh user@34.47.249.144 -p 22
# → TCP estabelece conexão segura
# → Você pode executar comandos remotamente
```

### FTP (File Transfer Protocol)

```bash
# Transferir arquivos
ftp ftp.exemplo.com
put arquivo.txt
get documento.pdf
# → TCP garante que os arquivos chegam intactos
```

### Email (SMTP)

```bash
# Seu email usa SMTP para enviar
# Servidor: smtp.gmail.com:587 (TCP)
# Seu cliente conecta → envia email → servidor envia para destinatário
```

### Banco de Dados (MySQL)

```bash
# Conectar a banco de dados
mysql -h 192.168.1.100 -u user -p
# → TCP conecta ao servidor MySQL na porta 3306
# → Queries são enviadas e respostas recebidas com garantia
```

### Redis (Cache)

```bash
# Conectar a Redis para cache rápido
redis-cli -h 192.168.1.100 -p 6379
SET chave valor
GET chave
# → TCP mantém conexão aberta para muitas operações rápidas
```

---

## Padrão: Quando TCP vs UDP?

### TCP (com garantia)
- **SSH** - precisa de todas as teclas que você digita
- **Email** - não pode perder mensagens
- **Banco de dados** - dados críticos
- **FTP** - arquivos precisam chegar completos
- **HTTP/HTTPS** - web precisa de integridade

### UDP (sem garantia, mais rápido)
- **DNS** - pode fazer nova consulta se perder
- **Streaming de vídeo** - alguns frames perdidos são ok
- **VoIP** - latência baixa importa mais
- **Jogos** - velocidade importa mais que perfeição
- **IoT** - muitos sensores, algumas perdas aceitáveis

---

## Em Load Balancer - Casos de Uso

```bash
# Network Load Balancer pode balancear:

# 1. SSH (porta 22) - múltiplos servidores SSH
gcloud compute forwarding-rules create ssh-lb \
    --region asia-south1 \
    --ports 22 \
    --target-pool ssh-pool

# 2. MySQL (porta 3306) - múltiplos bancos
gcloud compute forwarding-rules create mysql-lb \
    --region asia-south1 \
    --ports 3306 \
    --target-pool database-pool

# 3. Redis (porta 6379) - múltiplos caches
gcloud compute forwarding-rules create redis-lb \
    --region asia-south1 \
    --ports 6379 \
    --target-pool cache-pool

# 4. FTP (porta 20,21) - múltiplos servidores FTP
gcloud compute forwarding-rules create ftp-lb \
    --region asia-south1 \
    --ports 20,21 \
    --target-pool ftp-pool
```

---

## Resumo - TCP é Fundamental

TCP é **a base** para quase toda comunicação que precisa ser confiável na internet:

```
Internet moderna usa TCP para:
├── Web (HTTP/HTTPS)
├── Email (SMTP, POP3, IMAP)
├── Acesso remoto (SSH, Telnet)
├── Transferência de arquivos (FTP, SFTP)
├── Bancos de dados (MySQL, PostgreSQL, MongoDB)
├── Cache (Redis, Memcached)
├── Mensageria (RabbitMQ, Kafka)
└── APIs (REST, gRPC, etc)
```

**A regra de ouro:** Se os dados são importantes e não pode perder nenhum byte, **use TCP**!

---

## Protocolos e Seus Transportes

## Tabelão Didático de Protocolos

| Protocolo | Camada | Transporte | Porta(s) Padrão | O que faz | Quando usar | Observações |
|-----------|--------|------------|------------------|-----------|-------------|-------------|
| TCP | Transporte | TCP | N/A | Garante entrega confiável, ordem e retransmissão | Web, banco, e-mail, SSH, APIs | Mais confiável, porém com mais overhead |
| UDP | Transporte | UDP | N/A | Envia datagramas sem garantir entrega | DNS, VoIP, streaming, jogos | Mais rápido, menos confiável |
| QUIC | Transporte | UDP | 443 | Protocolo moderno de transporte para web e baixa latência | HTTP/3, conexões rápidas e resilientes | Combina ideia de transporte com segurança |
| HTTP | Aplicação | TCP | 80 | Protocolo da web para troca de páginas e APIs | Sites, APIs REST, sistemas web | Não é criptografado |
| HTTPS | Aplicação | TCP | 443 | HTTP com criptografia TLS/SSL | Sites seguros, APIs seguras, login, pagamentos | É o padrão da web moderna |
| HTTP/2 | Aplicação | TCP | 443 | Evolução do HTTP com multiplexação e melhor performance | Sites e APIs modernas | Normalmente usado com HTTPS |
| HTTP/3 | Aplicação | UDP via QUIC | 443 | Evolução do HTTP usando QUIC em vez de TCP | Web moderna com baixa latência | Importante notar que não usa TCP diretamente |
| WebSocket | Aplicação | TCP | 80, 443 | Comunicação bidirecional persistente | Chat, dashboards em tempo real, notificações | Começa via HTTP e depois mantém canal aberto |
| gRPC | Aplicação | TCP | 443 ou custom | RPC moderno e eficiente | Comunicação entre microsserviços | Geralmente roda sobre HTTP/2 |
| SSH | Aplicação | TCP | 22 | Acesso remoto seguro a servidores | Administração remota, SCP, SFTP, automação | Muito comum em cloud |
| DNS | Aplicação | UDP / TCP | 53 | Resolve nomes para IPs | Navegação, descoberta de serviços | UDP em consultas comuns; TCP em respostas grandes e transferência de zona |
| Telnet | Aplicação | TCP | 23 | Acesso remoto sem criptografia | Testes legados e ambientes antigos | Inseguro, evitar em produção |
| FTP | Aplicação | TCP | 20, 21 | Transferência de arquivos | Ambientes legados | Tem limitações com firewall/NAT |
| SFTP | Aplicação | TCP | 22 | Transferência segura de arquivos via SSH | Upload/download seguro | Não é “FTP com SSL”; é outro protocolo |
| FTPS | Aplicação | TCP | 990 ou 21 | FTP com TLS/SSL | Ambientes que exigem compatibilidade com FTP | Diferente de SFTP |
| SMTP | Aplicação | TCP | 25, 587, 465 | Envio de e-mails | Servidores e clientes de e-mail | 587 é submission; 465 costuma usar TLS implícito |
| POP3 | Aplicação | TCP | 110 | Baixa e-mails do servidor | Clientes de e-mail simples | Menos usado hoje |
| IMAP | Aplicação | TCP | 143 | Sincroniza e-mails no servidor | Clientes de e-mail modernos | Melhor que POP3 para múltiplos dispositivos |
| LDAP | Aplicação | TCP / UDP | 389 | Consulta diretórios de usuários e grupos | Autenticação corporativa, Active Directory | TCP é o mais comum |
| LDAPS | Aplicação | TCP | 636 | LDAP com criptografia | Diretórios seguros | Equivalente seguro do LDAP |
| DHCP | Aplicação | UDP | 67, 68 | Distribui IP automaticamente na rede | Redes locais, inicialização de hosts | Muito usado em redes internas |
| TFTP | Aplicação | UDP | 69 | Transferência simples de arquivos | Boot de dispositivos, rede legada | Simples e sem autenticação |
| NTP | Aplicação | UDP | 123 | Sincroniza relógios entre máquinas | Infraestrutura, servidores, clusters | Muito importante para logs e segurança |
| SNMP | Aplicação | UDP | 161, 162 | Monitoramento e gerenciamento de rede | Roteadores, switches, observabilidade | 161 consulta, 162 traps |
| SIP | Aplicação | UDP / TCP | 5060, 5061 | Sinalização de chamadas VoIP | Telefonia IP, videoconferência | Não carrega a mídia, só controla a chamada |
| RTP | Aplicação | UDP | Dinâmicas | Transporta áudio e vídeo em tempo real | VoIP, streaming ao vivo | Prioriza latência baixa |
| MySQL | Aplicação | TCP | 3306 | Comunicação com banco MySQL | Aplicações e APIs com banco relacional | Muito comum em backends |
| PostgreSQL | Aplicação | TCP | 5432 | Comunicação com banco PostgreSQL | Aplicações web, analytics, sistemas transacionais | Muito usado em sistemas modernos |
| SQL Server | Aplicação | TCP | 1433 | Comunicação com Microsoft SQL Server | Sistemas corporativos | Muito comum em ambientes Microsoft |
| Oracle Net | Aplicação | TCP | 1521 | Comunicação com banco Oracle | Grandes ambientes corporativos | Muito usado em legado enterprise |
| MongoDB | Aplicação | TCP | 27017 | Comunicação com banco NoSQL documental | Apps com dados semi-estruturados | Muito comum em aplicações flexíveis |
| Redis | Aplicação | TCP | 6379 | Cache, fila leve e armazenamento em memória | Cache, sessões, filas rápidas | Altamente performático |
| Memcached | Aplicação | TCP / UDP | 11211 | Cache em memória distribuído | Caches simples | TCP é mais comum |
| Kafka | Aplicação | TCP | 9092 | Streaming e mensageria distribuída | Eventos, pipelines, dados em tempo real | Muito usado em arquiteturas orientadas a eventos |
| RabbitMQ | Aplicação | TCP | 5672 | Mensageria e filas | Integração entre serviços | Muito usado em sistemas desacoplados |
| AMQP | Aplicação | TCP | 5672 | Protocolo de mensageria | Filas, integração assíncrona | RabbitMQ usa AMQP com frequência |
| MQTT | Aplicação | TCP | 1883 | Mensageria leve para dispositivos | IoT, sensores, dispositivos com pouca banda | Muito eficiente |
| MQTTS | Aplicação | TCP | 8883 | MQTT com TLS | IoT seguro | Versão segura do MQTT |