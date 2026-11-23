# Deploy a Compute Instance with a Remote Startup Script: Challenge Lab

## Visao Geral

Em um challenge lab voce recebe um cenario e um conjunto de tarefas, sem passo a passo. Voce usa o que aprendeu no curso para resolver cada tarefa, e um sistema de avaliacao automatica indica se cada uma foi concluida corretamente. Para fazer 100%, todas as tarefas precisam ser concluidas dentro do tempo.

Cenario: voce gerencia a configuracao das VMs da organizacao e decidiu facilitar a alteracao dos startup scripts. Em vez de guardar o script direto na metadata das instancias, ele fica em um bucket do Cloud Storage e as VMs apontam para o arquivo no bucket.

O script de exemplo `install-web.sh` (instala o Apache) esta em `gs://spls/gsp301/install-web.sh`.

## Introducao

Este guia converte o challenge lab (GSP301) para execucao via CLI com `gcloud` e `curl`. Como e um challenge lab, a sequencia de comandos foi inferida a partir dos criterios de cada tarefa.

Objetivo:
- criar um bucket e copiar o `install-web.sh` para ele;
- criar uma VM Linux com `startup-script-url` apontando para o script no bucket;
- liberar HTTP (`80/tcp`) da internet para a VM;
- validar que o Apache esta servindo conteudo.

Regras do lab:
- criar a VM na zona `ZONE` informada no lab;
- **nao alterar** os recursos pre-criados, em especial a VM `lab-monitor`.

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export ZONE="<ZONE_DO_LAB>"          # ajuste conforme o lab
export REGION="${ZONE%-*}"           # ex: us-east1-b -> us-east1

# Recursos criados neste guia (nomes livres)
export BUCKET_NAME="${PROJECT_ID}-startup-scripts"
export SCRIPT_NAME="install-web.sh"
export VM_NAME="apache-web-vm"
export HTTP_TAG="http-server"
export FW_RULE="allow-http-80"

gcloud config set project "$PROJECT_ID"
gcloud config set compute/zone "$ZONE"
echo "Zona: $ZONE | Regiao: $REGION"
```

### Explicacao dos parametros
- `${ZONE%-*}`: remove o sufixo da zona para obter a regiao.
- `HTTP_TAG`: network tag que liga a regra de firewall a VM.

---

## TAREFA 1: Criar o bucket e copiar o startup script

### Conceito
O startup script fica no Cloud Storage. Assim, alterar o script nao exige mexer na metadata de cada VM.

### Passo 1: Criar o bucket

```bash
gcloud storage buckets create "gs://${BUCKET_NAME}" --location="$REGION"
```

### Passo 2: Copiar o script

```bash
gcloud storage cp "gs://spls/gsp301/${SCRIPT_NAME}" "gs://${BUCKET_NAME}/"
gcloud storage ls "gs://${BUCKET_NAME}/"
```

### Explicacao dos parametros
- `buckets create --location`: bucket regional na mesma regiao da VM.
- `storage cp gs://... gs://...`: copia direto entre buckets, sem baixar localmente.

### Resultado esperado
- `gs://<BUCKET_NAME>/install-web.sh` listado.

> Click **Check my progress**: *Create a storage bucket and copy the startup script*.

---

## TAREFA 2: Criar a VM com startup script remoto

### Conceito
A chave de metadata `startup-script-url` faz a VM baixar e executar o script do bucket no boot. A service account da VM precisa conseguir **ler** o objeto (escopo padrao inclui `devstorage.read_only`).

### Passo 1: Criar a VM

```bash
gcloud compute instances create "$VM_NAME" \
  --zone="$ZONE" \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags="$HTTP_TAG" \
  --metadata=startup-script-url="gs://${BUCKET_NAME}/${SCRIPT_NAME}"
```

### Passo 2: Conferir a metadata

```bash
gcloud compute instances describe "$VM_NAME" --zone="$ZONE" \
  --format="value(metadata.items)"
```

### Explicacao dos parametros
- `--metadata=startup-script-url=gs://...`: script remoto executado a cada boot.
- `--tags`: aplica a tag usada pela regra de firewall da Tarefa 3.
- Sem `--scopes`: usa os escopos padrao da Compute default SA (inclui leitura de Storage).

### Resultado esperado
- VM `RUNNING` com `startup-script-url` na metadata.

> Click **Check my progress**: *Create a VM instance with a remote startup script*.

---

## TAREFA 3: Criar regra de firewall para HTTP (80/tcp)

### Conceito
Sem regra de entrada na porta 80, o Apache instala mas nao fica acessivel pela internet.

### Passo 1: Criar a regra

```bash
gcloud compute firewall-rules create "$FW_RULE" \
  --network=default \
  --direction=INGRESS \
  --allow=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags="$HTTP_TAG"
```

### Explicacao dos parametros
- `--allow=tcp:80`: libera HTTP.
- `--source-ranges=0.0.0.0/0`: qualquer origem na internet.
- `--target-tags`: aplica so nas VMs com a tag `http-server`.

### Resultado esperado
- Regra `allow-http-80` listada em `gcloud compute firewall-rules list`.

> Click **Check my progress**: *Create a firewall rule to allow traffic (80/tcp)*.

---

## TAREFA 4: Testar se a VM esta servindo conteudo web

### Conceito
O teste final e um `GET` HTTP no IP **externo** da VM (nao HTTPS, nao IP interno).

### Passo 1: Conferir a execucao do startup script (Serial Console)

```bash
gcloud compute instances get-serial-port-output "$VM_NAME" --zone="$ZONE" \
  | grep -i startup-script | tail -n 20
```

### Passo 2: Capturar o IP externo e testar

```bash
export VM_EXTERNAL_IP="$(gcloud compute instances describe "$VM_NAME" --zone="$ZONE" \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)')"
echo "http://${VM_EXTERNAL_IP}"

curl -s -o /dev/null -w "%{http_code}\n" "http://${VM_EXTERNAL_IP}"
curl -s "http://${VM_EXTERNAL_IP}" | head -n 5
```

### Resultado esperado
- Log `startup-script-url: ... Finished running startup scripts` na serial.
- `curl` retornando `200` com a pagina padrao do Apache.

> Click **Check my progress**: *Test that the VM is serving web content*.

---

## Validacao

```bash
gcloud storage ls "gs://${BUCKET_NAME}/${SCRIPT_NAME}"

gcloud compute instances describe "$VM_NAME" --zone="$ZONE" \
  --format="yaml(status,tags.items,metadata.items)"

gcloud compute firewall-rules describe "$FW_RULE" \
  --format="yaml(allowed,sourceRanges,targetTags)"

curl -s -o /dev/null -w "%{http_code}\n" "http://${VM_EXTERNAL_IP}"
```

Checklist esperado:
- script no bucket;
- VM com `startup-script-url` e tag `http-server`;
- firewall `tcp:80` de `0.0.0.0/0` para a tag;
- HTTP `200` no IP externo.

---

## Resumo Visual da Arquitetura

```text
[gs://spls/gsp301/install-web.sh]
        |
        | gcloud storage cp
        v
[gs://BUCKET_NAME/install-web.sh]
        |
        | startup-script-url (leitura no boot)
        v
[VM apache-web-vm] (ZONE, debian-12, tag http-server) -- Apache :80
        ^
        | firewall allow-http-80 (tcp:80, 0.0.0.0/0 -> http-server)
        |
[Internet] -- curl http://VM_EXTERNAL_IP

[lab-monitor] (pre-criada, nao alterar)
```

---

## Troubleshooting

- Apache nao instalou:
  - veja a serial (`get-serial-port-output`) e procure erros de `startup-script-url`.
- Erro `403` ao baixar o script no boot:
  - a SA da VM nao le o bucket. Conceda leitura e reinicie a VM:
    ```bash
    export VM_SA="$(gcloud compute instances describe "$VM_NAME" --zone="$ZONE" --format='value(serviceAccounts[0].email)')"
    gcloud storage buckets add-iam-policy-binding "gs://${BUCKET_NAME}" \
      --member="serviceAccount:${VM_SA}" --role=roles/storage.objectViewer
    gcloud compute instances reset "$VM_NAME" --zone="$ZONE"
    ```
- `curl` sem resposta / `Connection refused`:
  - confira a regra de firewall e a tag da VM; use `http://` e o IP externo.
- VM criada na zona errada:
  - recrie na `ZONE` do lab; o check procura a VM na zona indicada.

---

## Limpeza (Opcional)

Nao remova a VM `lab-monitor`.

```bash
gcloud compute instances delete "$VM_NAME" --zone="$ZONE" --quiet
gcloud compute firewall-rules delete "$FW_RULE" --quiet
gcloud storage rm -r "gs://${BUCKET_NAME}"
```

---

## Conceitos-Chave

- `startup-script-url`: metadata que aponta para um script no Cloud Storage, executado a cada boot.
- Escopos e IAM da VM: a SA precisa de leitura no bucket para baixar o script.
- Network tags + firewall: controlam quais VMs recebem cada regra de entrada.
- Serial port output: principal fonte para depurar startup scripts.

---

## Fluxo Final

1. Definir variaveis (`ZONE` do lab, bucket, VM, tag).
2. Criar o bucket e copiar `install-web.sh`.
3. Criar a VM com `startup-script-url` e tag `http-server`.
4. Criar a regra de firewall `tcp:80` para a tag.
5. Conferir a serial e testar `curl http://VM_EXTERNAL_IP`.
