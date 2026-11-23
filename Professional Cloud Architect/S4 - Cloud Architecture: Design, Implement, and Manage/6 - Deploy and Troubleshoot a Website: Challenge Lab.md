# Deploy and Troubleshoot a Website: Challenge Lab

## Visao Geral

Em um challenge lab voce recebe um cenario e um conjunto de tarefas, sem passo a passo. Voce usa o que aprendeu no curso para resolver cada tarefa, e um sistema de avaliacao automatica indica se cada uma foi concluida corretamente. Para fazer 100%, todas as tarefas precisam ser concluidas dentro do tempo.

Cenario: a empresa vai lancar um produto novo e o site esta pronto, mas quem o construiu saiu antes do deploy. Voce vai publica-lo na nuvem usando um servidor Apache simples como placeholder do site, em uma VM do Compute Engine.

## Introducao

Este guia converte o challenge lab (GSP101) para execucao via CLI com `gcloud` e `curl`. Como e um challenge lab, a sequencia de comandos foi inferida a partir dos criterios de cada tarefa.

Objetivo:
- criar uma VM Linux com o nome e a zona informados no lab;
- liberar HTTP para a VM (tag + regra de firewall);
- instalar o Apache e publicar uma pagina "Hello World!";
- testar pelo IP externo.

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export INSTANCE_NAME="<INSTANCE_NAME_DO_LAB>"   # "Instance name" do lab
export ZONE="<ZONE_DO_LAB>"                     # "Compute zone" do lab

export HTTP_TAG="http-server"
export FW_RULE="default-allow-http"

gcloud config set project "$PROJECT_ID"
gcloud config set compute/zone "$ZONE"
```

### Explicacao dos parametros
- `HTTP_TAG=http-server`: mesma tag que o Console aplica ao marcar **Allow HTTP traffic**.
- `FW_RULE=default-allow-http`: mesmo nome da regra criada pelo Console para essa tag.

---

## TAREFA 1: Criar a VM Linux

### Conceito
VM Debian com a tag `http-server`, para que a regra de firewall de HTTP se aplique a ela desde a criacao.

### Passo 1: Criar a VM

```bash
gcloud compute instances create "$INSTANCE_NAME" \
  --zone="$ZONE" \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags="$HTTP_TAG"
```

### Explicacao dos parametros
- `--image-family=debian-12 --image-project=debian-cloud`: Linux Debian 12.
- `--tags=http-server`: equivale a marcar **Allow HTTP traffic** no Console.

### Resultado esperado
- VM `INSTANCE_NAME` em `RUNNING` na `ZONE`, com IP externo.

---

## TAREFA 2: Liberar acesso publico a VM

### Conceito
A tag sozinha nao abre a porta: e preciso uma regra de firewall que libere `tcp:80` para essa tag.

### Passo 1: Criar a regra (se ainda nao existir)

```bash
gcloud compute firewall-rules describe "$FW_RULE" >/dev/null 2>&1 || \
gcloud compute firewall-rules create "$FW_RULE" \
  --network=default \
  --direction=INGRESS \
  --allow=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags="$HTTP_TAG"
```

### Explicacao dos parametros
- `describe ... || create ...`: so cria a regra se ela ainda nao existir.
- `--allow=tcp:80 --source-ranges=0.0.0.0/0`: HTTP de qualquer origem.
- `--target-tags=http-server`: aplica apenas nas VMs com a tag.

### Resultado esperado
- Regra `default-allow-http` liberando `tcp:80` para `http-server`.

> Click **Check my progress**: *Create a Compute Engine instance, add necessary firewall rules.*

---

## TAREFA 3: Executar um servidor Apache basico

### Conceito
A VM e administrada como qualquer servidor Linux: via SSH, instala-se o `apache2` e substitui-se a pagina padrao por um "Hello World!".

### Passo 1: Instalar o Apache e publicar a pagina

```bash
gcloud compute ssh "$INSTANCE_NAME" --zone="$ZONE" --command "
  sudo apt-get update &&
  sudo apt-get install -y apache2 &&
  echo '<!doctype html><html><body><h1>Hello World!</h1></body></html>' | sudo tee /var/www/html/index.html &&
  sudo systemctl enable --now apache2 &&
  systemctl is-active apache2
"
```

Na primeira execucao, responda `Y` e de `ENTER` nas perguntas de chave SSH.

### Explicacao dos parametros
- `gcloud compute ssh --command`: executa os comandos na VM sem abrir shell interativo.
- `apt-get install -y apache2`: instala o servidor web.
- `tee /var/www/html/index.html`: substitui a pagina padrao pelo "Hello World!".
- `systemctl enable --now`: inicia o Apache e o mantem ativo apos reboot.

### Resultado esperado
- `systemctl is-active apache2` retornando `active`.

> Click **Check my progress**: *Add Apache2 HTTP Server to your instance*.

---

## TAREFA 4: Testar o servidor

### Conceito
O teste e um `GET` HTTP no IP **externo** da VM (nao HTTPS).

### Passo 1: Capturar o IP e testar

```bash
export EXTERNAL_IP="$(gcloud compute instances describe "$INSTANCE_NAME" --zone="$ZONE" \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)')"
echo "http://${EXTERNAL_IP}"

curl -s "http://${EXTERNAL_IP}"
```

### Resultado esperado
- Pagina com `Hello World!`.

> Click **Check my progress**: *Test your server*.

---

## Validacao

```bash
gcloud compute instances describe "$INSTANCE_NAME" --zone="$ZONE" \
  --format="yaml(status,tags.items,networkInterfaces[0].accessConfigs[0].natIP)"

gcloud compute firewall-rules describe "$FW_RULE" \
  --format="yaml(allowed,sourceRanges,targetTags)"

curl -s -o /dev/null -w "%{http_code}\n" "http://${EXTERNAL_IP}"
```

Checklist esperado:
- VM `INSTANCE_NAME` na `ZONE`, com tag `http-server` e IP externo;
- regra `tcp:80` de `0.0.0.0/0` para a tag;
- Apache ativo e `curl` retornando `200` com "Hello World!".

---

## Resumo Visual da Arquitetura

```text
[Internet] -- curl http://EXTERNAL_IP
        |
        | firewall default-allow-http (tcp:80, 0.0.0.0/0 -> http-server)
        v
[VM INSTANCE_NAME] (ZONE, debian-12, tag http-server)
  [Apache2 :80] -> /var/www/html/index.html ("Hello World!")
```

---

## Troubleshooting

- `Connection refused` / sem resposta:
  - a VM nao tem a tag `http-server` ou nao existe regra liberando `tcp:80`. Corrija a tag:
    ```bash
    gcloud compute instances add-tags "$INSTANCE_NAME" --zone="$ZONE" --tags="$HTTP_TAG"
    ```
- Acesso por `https://`:
  - use `http://EXTERNAL_IP`; o Apache nao foi configurado com TLS.
- Pagina padrao do Apache em vez de "Hello World!":
  - o `index.html` nao foi substituido; rode de novo o passo da Tarefa 3.
- VM criada com nome ou zona diferentes do lab:
  - o check procura `INSTANCE_NAME` na `ZONE` do lab; recrie com os valores corretos.

---

## Limpeza (Opcional)

```bash
gcloud compute instances delete "$INSTANCE_NAME" --zone="$ZONE" --quiet
gcloud compute firewall-rules delete "$FW_RULE" --quiet
```

---

## Conceitos-Chave

- VM do Compute Engine: administrada como qualquer servidor Linux (SSH, pacotes, servicos).
- Network tags + firewall: a tag `http-server` so tem efeito com uma regra que a referencie.
- Acesso publico: exige IP externo, regra de firewall e servico escutando na porta.
- Troubleshooting web: checar tag, regra, servico ativo e protocolo (`http` x `https`).

---

## Fluxo Final

1. Definir variaveis (`INSTANCE_NAME` e `ZONE` do lab).
2. Criar a VM Debian com a tag `http-server`.
3. Criar a regra `default-allow-http` (`tcp:80`) se nao existir.
4. Instalar o Apache via SSH e publicar o "Hello World!".
5. Testar `curl http://EXTERNAL_IP`.
