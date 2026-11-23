# Migrate a MySQL Database to Google Cloud SQL: Challenge Lab

## Visao Geral

Em um challenge lab voce recebe um cenario e um conjunto de tarefas, sem passo a passo. Voce usa o que aprendeu no curso para resolver cada tarefa, e um sistema de avaliacao automatica indica se cada uma foi concluida corretamente. Para fazer 100%, todas as tarefas precisam ser concluidas dentro do tempo.

Cenario: um blog WordPress roda em um servidor que nao atende mais. Como primeira etapa da migracao, o banco MySQL local do blog sera migrado para o Cloud SQL e o WordPress sera reconfigurado para usar o banco novo.

Dados do ambiente:
- VM `blog` ja rodando, com WordPress em `/var/www/html/wordpress`;
- MySQL local com o banco `wordpress`, usuario `blogadmin` e senha `Password1*`;
- arquivo de configuracao: `/var/www/html/wordpress/wp-config.php`.

## Introducao

Este guia converte o challenge lab (GSP306) para execucao via CLI com `gcloud`, `mysql`/`mysqldump` (na VM `blog`) e `curl`. Como e um challenge lab, a sequencia de comandos foi inferida a partir dos criterios de cada tarefa.

Pontos de atencao:
- o score comeca com 20 pontos porque o blog esta no ar; se a migracao quebrar o blog, esses pontos sao perdidos;
- criar o Cloud SQL na `REGION`/`ZONE` informadas no lab;
- o check da Tarefa 3 verifica se a VM `blog` esta **autorizada** a acessar o Cloud SQL (authorized networks).

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao. Os comandos na VM `blog` sao enviados com `gcloud compute ssh --command`, com as variaveis expandidas no Cloud Shell.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export REGION="<REGION_DO_LAB>"   # ajuste conforme o lab
export ZONE="<ZONE_DO_LAB>"       # ajuste conforme o lab

# Ambiente existente
export BLOG_VM="blog"
export WP_DIR="/var/www/html/wordpress"
export WP_CONFIG="${WP_DIR}/wp-config.php"
export DB_NAME="wordpress"
export DB_USER="blogadmin"
export DB_PASS='Password1*'

# Cloud SQL (nome livre)
export SQL_INSTANCE="wordpress"
export SQL_ROOT_PASS='Password1*'
export DUMP_FILE="wordpress_backup.sql"

gcloud config set project "$PROJECT_ID"
```

### Passo 2: Capturar zona e IP externo da VM `blog`

```bash
export BLOG_ZONE="$(gcloud compute instances list --filter="name=${BLOG_VM}" --format='value(zone.basename())')"
export BLOG_IP="$(gcloud compute instances describe "$BLOG_VM" --zone="$BLOG_ZONE" \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)')"
echo "blog: zona=${BLOG_ZONE} ip=${BLOG_IP}"

curl -s -o /dev/null -w "%{http_code}\n" "http://${BLOG_IP}"
```

### Explicacao dos parametros
- `DB_PASS` / `SQL_ROOT_PASS` entre aspas simples: o `*` nao e expandido pelo shell.
- `BLOG_IP`: usado no authorized network do Cloud SQL e no teste final.

**Resultado esperado:** blog respondendo `200` (ou `30x`) antes da migracao.

---

## TAREFA 1: Criar a instancia Cloud SQL

### Conceito
Instancia Cloud SQL for MySQL na `REGION`/`ZONE` do lab. Ja na criacao, a VM `blog` e autorizada a conectar pelo IP publico (`BLOG_IP/32`).

### Passo 1: Criar a instancia

```bash
gcloud sql instances create "$SQL_INSTANCE" \
  --database-version=MYSQL_8_0 \
  --edition=enterprise \
  --tier=db-n1-standard-1 \
  --region="$REGION" \
  --zone="$ZONE" \
  --root-password="$SQL_ROOT_PASS" \
  --authorized-networks="${BLOG_IP}/32"
```

A criacao leva alguns minutos.

### Passo 2: Capturar o IP do Cloud SQL

```bash
export SQL_IP="$(gcloud sql instances describe "$SQL_INSTANCE" \
  --format='value(ipAddresses[0].ipAddress)')"
echo "$SQL_IP"
```

### Explicacao dos parametros
- `--database-version=MYSQL_8_0`: MySQL compativel com o WordPress.
- `--edition=enterprise --tier=db-n1-standard-1`: maquina pequena suficiente para o blog.
- `--region` / `--zone`: localizacao exigida pelo lab.
- `--authorized-networks=BLOG_IP/32`: libera somente o IP da VM `blog`.

### Resultado esperado
- Instancia `wordpress` em `RUNNABLE` com IP publico.

> Click **Check my progress**: *Check that there is a Cloud SQL instance.*

---

## TAREFA 2: Configurar o novo banco

### Conceito
Criar o banco `wordpress` e o usuario `blogadmin` no Cloud SQL, com as mesmas credenciais do banco local. Assim o `wp-config.php` so precisa trocar o host.

### Passo 1: Criar banco e usuario

```bash
gcloud sql databases create "$DB_NAME" --instance="$SQL_INSTANCE"

gcloud sql users create "$DB_USER" \
  --instance="$SQL_INSTANCE" \
  --host=% \
  --password="$DB_PASS"
```

### Passo 2: Validar

```bash
gcloud sql databases list --instance="$SQL_INSTANCE"
gcloud sql users list --instance="$SQL_INSTANCE"
```

### Explicacao dos parametros
- `databases create`: banco de usuario `wordpress`.
- `users create --host=%`: usuario `blogadmin` aceito de qualquer host (o acesso de rede continua restrito pelo authorized network).

### Resultado esperado
- Banco `wordpress` e usuario `blogadmin` listados.

> Click **Check my progress**: *Check that there is a user database on the Cloud SQL instance.*

---

## TAREFA 3: Fazer o dump e importar os dados

### Conceito
Na VM `blog`, `mysqldump` exporta o banco local; em seguida o cliente `mysql` (ja instalado na VM) importa o arquivo direto no Cloud SQL pelo IP publico autorizado.

### Passo 1: Conferir a autorizacao da VM no Cloud SQL

```bash
gcloud sql instances describe "$SQL_INSTANCE" \
  --format="value(settings.ipConfiguration.authorizedNetworks[].value)"
```

Se o IP do `blog` nao aparecer, adicione:

```bash
gcloud sql instances patch "$SQL_INSTANCE" --authorized-networks="${BLOG_IP}/32" --quiet
```

### Passo 2: Gerar o dump na VM `blog`

```bash
gcloud compute ssh "$BLOG_VM" --zone="$BLOG_ZONE" --command "
  mysqldump --databases ${DB_NAME} -h localhost -u ${DB_USER} -p'${DB_PASS}' \
    --hex-blob --single-transaction --default-character-set=utf8mb4 --no-tablespaces \
    > ~/${DUMP_FILE} && ls -lh ~/${DUMP_FILE}
"
```

### Passo 3: Importar no Cloud SQL a partir da VM `blog`

```bash
gcloud compute ssh "$BLOG_VM" --zone="$BLOG_ZONE" --command "
  mysql --host=${SQL_IP} --user=root -p'${SQL_ROOT_PASS}' < ~/${DUMP_FILE} &&
  mysql --host=${SQL_IP} --user=${DB_USER} -p'${DB_PASS}' -e 'SHOW TABLES FROM ${DB_NAME};'
"
```

### Explicacao dos parametros
- `--databases wordpress`: inclui `CREATE DATABASE IF NOT EXISTS` / `USE` no dump.
- `--single-transaction`: dump consistente sem travar as tabelas.
- `--no-tablespaces`: evita exigir o privilegio `PROCESS` no MySQL local.
- `mysql --host=SQL_IP < dump`: importa pela rede usando o authorized network.

### Resultado esperado
- Tabelas `wp_*` listadas no banco `wordpress` do Cloud SQL.

> Click **Check my progress**: *Check that the blog instance is authorized to access Cloud SQL.*

### Resumo visual

```text
[VM blog] (BLOG_ZONE, BLOG_IP)
  [MySQL local: wordpress]
        |
        | mysqldump --databases wordpress
        v
  [~/wordpress_backup.sql]
        |
        | mysql --host=SQL_IP  (authorized network BLOG_IP/32)
        v
[Cloud SQL wordpress] (REGION/ZONE, MySQL 8.0)
  [db wordpress] + [user blogadmin]
```

---

## TAREFA 4: Reconfigurar o WordPress

### Conceito
O `wp-config.php` define `DB_HOST`. Como banco, usuario e senha foram mantidos, basta trocar `localhost` pelo IP do Cloud SQL.

### Passo 1: Ver a configuracao atual

```bash
gcloud compute ssh "$BLOG_VM" --zone="$BLOG_ZONE" --command \
  "grep -E \"DB_(NAME|USER|HOST)\" ${WP_CONFIG}"
```

### Passo 2: Apontar o `DB_HOST` para o Cloud SQL

```bash
gcloud compute ssh "$BLOG_VM" --zone="$BLOG_ZONE" --command "
  sudo cp ${WP_CONFIG} ${WP_CONFIG}.bak &&
  sudo sed -i \"/DB_HOST/s/localhost/${SQL_IP}/\" ${WP_CONFIG} &&
  grep DB_HOST ${WP_CONFIG}
"
```

### Explicacao dos parametros
- `cp ... .bak`: backup para rollback rapido.
- `sed "/DB_HOST/s/localhost/SQL_IP/"`: troca `localhost` apenas na linha do `DB_HOST`.

### Resultado esperado
- `define( 'DB_HOST', '<SQL_IP>' );` no `wp-config.php`.

> Click **Check my progress**: *Check that wp-config.php points to the Cloud SQL instance.*

### Resumo visual

```text
Antes:  [WordPress] --DB_HOST=localhost--> [MySQL local na VM blog]

        sed DB_HOST localhost -> SQL_IP

Depois: [WordPress] --DB_HOST=SQL_IP--> [Cloud SQL wordpress]
```

---

## TAREFA 5: Validar e fazer troubleshooting

### Conceito
O blog deve continuar respondendo, agora lendo os dados do Cloud SQL.

### Passo 1: Testar o blog

```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://${BLOG_IP}"
curl -s "http://${BLOG_IP}" | grep -i -o '<title>.*</title>'
```

### Passo 2 (opcional): Provar que o blog nao depende do MySQL local

```bash
gcloud compute ssh "$BLOG_VM" --zone="$BLOG_ZONE" --command "sudo systemctl stop mysql"
curl -s -o /dev/null -w "%{http_code}\n" "http://${BLOG_IP}"
```

Se algo falhar, religue com `sudo systemctl start mysql` e restaure o backup do `wp-config.php`.

### Resultado esperado
- Blog respondendo `200` com o titulo do site, mesmo com o MySQL local parado.

> Click **Check my progress**: *Check that the blog still responds to requests.*

---

## Validacao

```bash
gcloud sql instances describe "$SQL_INSTANCE" \
  --format="yaml(state,region,gceZone,databaseVersion,settings.ipConfiguration.authorizedNetworks)"

gcloud sql databases list --instance="$SQL_INSTANCE"
gcloud sql users list --instance="$SQL_INSTANCE"

gcloud compute ssh "$BLOG_VM" --zone="$BLOG_ZONE" --command "grep DB_HOST ${WP_CONFIG}"

curl -s -o /dev/null -w "%{http_code}\n" "http://${BLOG_IP}"
```

Checklist esperado:
- Cloud SQL `wordpress` em `RUNNABLE` na `REGION`/`ZONE`, com `BLOG_IP/32` autorizado;
- banco `wordpress` com as tabelas `wp_*` e usuario `blogadmin`;
- `DB_HOST` apontando para o IP do Cloud SQL;
- blog respondendo `200`.

---

## Resumo Visual da Arquitetura

Compilado: estado final ao termino do lab. Os detalhes da migracao estao no `### Resumo visual` das Tarefas 3 e 4.

```text
[Internet] -- http://BLOG_IP
        |
        v
[VM blog] (BLOG_ZONE)  Apache + WordPress
  wp-config.php: DB_HOST=SQL_IP
        |
        | MySQL 3306 (authorized network BLOG_IP/32)
        v
[Cloud SQL wordpress] (REGION/ZONE, MySQL 8.0)
  db wordpress (tabelas wp_*) | user blogadmin

[MySQL local] (nao usado mais pelo WordPress)
```

---

## Troubleshooting

- `ERROR 2003 Can't connect to MySQL server` na importacao:
  - o IP do `blog` nao esta autorizado; rode o `instances patch --authorized-networks` da Tarefa 3.
- `Access denied` no `mysqldump`:
  - confira usuario/senha locais (`blogadmin` / `Password1*`) e as aspas simples na senha.
- Erro de collation/charset na importacao (ex.: `utf8mb4_0900_ai_ci`):
  - gere o dump novamente com `--default-character-set=utf8mb4`; se persistir, crie o Cloud SQL com `--database-version=MYSQL_5_7` (se disponivel).
- Blog com "Error establishing a database connection":
  - confira `DB_HOST` no `wp-config.php`, o usuario `blogadmin` no Cloud SQL e o authorized network; para voltar, `sudo cp wp-config.php.bak wp-config.php`.
- `gcloud sql instances create` recusando `--tier`:
  - confirme `--edition=enterprise` junto com `db-n1-standard-1`.
- Score caiu 20 pontos:
  - o blog parou de responder; corrija a conexao com o banco ou restaure o backup.

---

## Limpeza (Opcional)

Nao remova a VM `blog`: ela e pre-criada pelo lab.

```bash
gcloud sql instances delete "$SQL_INSTANCE" --quiet
```

---

## Conceitos-Chave

- Cloud SQL: MySQL gerenciado (backups, patches, alta disponibilidade opcional).
- Authorized networks: lista de IPs publicos liberados para conectar na instancia.
- `mysqldump` + `mysql`: migracao logica simples, adequada a bancos pequenos.
- Reconfiguracao da aplicacao: manter nome do banco e credenciais reduz a mudanca ao `DB_HOST`.
- Validacao com dependencia removida: parar o MySQL local prova que a app usa o Cloud SQL.

---

## Fluxo Final

1. Definir variaveis e capturar zona/IP da VM `blog`.
2. Criar o Cloud SQL `wordpress` na `REGION`/`ZONE` com `BLOG_IP/32` autorizado.
3. Criar o banco `wordpress` e o usuario `blogadmin`.
4. `mysqldump` na VM `blog` e importar no Cloud SQL via `mysql --host=SQL_IP`.
5. Trocar o `DB_HOST` do `wp-config.php` para o IP do Cloud SQL.
6. Testar o blog (opcionalmente com o MySQL local parado).
