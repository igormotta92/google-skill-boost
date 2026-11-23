# Configure Secure RDP using a Windows Bastion Host: Challenge Lab

## Visao Geral

Em um challenge lab voce recebe um cenario e um conjunto de tarefas, sem passo a passo. Voce usa o que aprendeu no curso para resolver cada tarefa, e um sistema de avaliacao automatica indica se cada uma foi concluida corretamente. Para fazer 100%, todas as tarefas precisam ser concluidas dentro do tempo.

Cenario: a empresa vai rodar servicos Windows na nuvem e voce precisa de um ambiente seguro para gerencia-los:

- os servidores de producao ficam **isolados** da internet (sem acesso de/para fora);
- um **bastion host** (jump box) e acessivel da internet **apenas via RDP** e so fala RDP com as VMs internas;
- todas as VMs tem uma **segunda interface** interna na VPC `default`, usada pelo sistema de monitoramento.

Ao final, o Microsoft IIS deve estar instalado no servidor seguro.

## Introducao

Este guia converte o challenge lab (GSP303) para execucao via CLI com `gcloud`. A configuracao do IIS e feita via RDP (passo do lab), com uma alternativa por startup script PowerShell.

Recursos com nome exigido:
- VPC `securenetwork` (com uma subnet na `REGION` do lab);
- VM segura `vm-securehost` (sem IP externo);
- bastion `vm-bastionhost` (com IP externo efemero);
- usuario Windows `app_admin` em cada VM.

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export REGION="<REGION_DO_LAB>"   # ajuste conforme o lab
export ZONE="<ZONE_DO_LAB>"       # ajuste conforme o lab

# Rede
export VPC_NAME="securenetwork"
export SUBNET_NAME="securenetwork-subnet"
export SUBNET_RANGE="192.168.16.0/20"

# VMs
export SECURE_VM="vm-securehost"
export BASTION_VM="vm-bastionhost"
export BASTION_TAG="bastion"
export SECURE_TAG="securehost"
export WIN_IMAGE_FAMILY="windows-2016"   # Server with Desktop Experience
export WIN_USER="app_admin"

# Firewall
export FW_RDP_EXTERNAL="securenetwork-allow-rdp-bastion"
export FW_RDP_INTERNAL="securenetwork-allow-rdp-bastion-to-secure"

gcloud config set project "$PROJECT_ID"
gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"
```

### Explicacao dos parametros
- `SUBNET_RANGE`: faixa privada para a subnet (qualquer faixa que nao conflite com a `default`).
- `windows-2016`: familia de imagem do Windows Server 2016 **com** Desktop Experience (`windows-2016-core` e a versao sem GUI).
- `BASTION_TAG` / `SECURE_TAG`: network tags usadas nas regras de firewall.

---

## TAREFA 1: Criar a rede VPC

### Conceito
Uma VPC em modo **custom** com uma unica subnet regional. Redes custom nao tem regras de firewall de entrada, entao tudo e negado ate ser liberado explicitamente.

### Passo 1: Criar a VPC

```bash
gcloud compute networks create "$VPC_NAME" --subnet-mode=custom
```

> Click **Check my progress**: *Create the VPC network*.

### Passo 2: Criar a subnet

```bash
gcloud compute networks subnets create "$SUBNET_NAME" \
  --network="$VPC_NAME" \
  --region="$REGION" \
  --range="$SUBNET_RANGE"
```

> Click **Check my progress**: *Create the VPC subnet*.

### Passo 3: Liberar RDP da internet para o bastion

```bash
gcloud compute firewall-rules create "$FW_RDP_EXTERNAL" \
  --network="$VPC_NAME" \
  --direction=INGRESS \
  --allow=tcp:3389 \
  --source-ranges=0.0.0.0/0 \
  --target-tags="$BASTION_TAG"
```

### Passo 4: Liberar RDP do bastion para o servidor seguro (dentro da VPC)

```bash
gcloud compute firewall-rules create "$FW_RDP_INTERNAL" \
  --network="$VPC_NAME" \
  --direction=INGRESS \
  --allow=tcp:3389 \
  --source-tags="$BASTION_TAG" \
  --target-tags="$SECURE_TAG"
```

### Explicacao dos parametros
- `--subnet-mode=custom`: subnets criadas manualmente.
- Regra externa: `tcp:3389` de `0.0.0.0/0` **somente** para VMs com a tag `bastion`.
- Regra interna: `tcp:3389` apenas com origem na tag `bastion` e destino na tag `securehost`.

### Resultado esperado
- VPC `securenetwork`, subnet na `REGION` e as duas regras de RDP.

> Click **Check my progress**: *Create the firewall rule*.

---

## TAREFA 2: Fazer deploy das VMs Windows e configurar as senhas

### Conceito
Cada VM tem **duas NICs**: `nic0` na `securenetwork` e `nic1` na VPC `default` (interna, para monitoramento). Cada NIC precisa estar em uma VPC diferente. So o bastion tem IP externo, e apenas na `nic0`.

### Passo 1: Criar o servidor seguro `vm-securehost` (sem IP externo)

```bash
gcloud compute instances create "$SECURE_VM" \
  --zone="$ZONE" \
  --machine-type=e2-medium \
  --image-family="$WIN_IMAGE_FAMILY" \
  --image-project=windows-cloud \
  --boot-disk-size=50GB \
  --tags="$SECURE_TAG" \
  --network-interface=subnet="$SUBNET_NAME",no-address \
  --network-interface=subnet=default,no-address
```

> Click **Check my progress**: *Create the vm-securehost instance*.

### Passo 2: Criar o bastion `vm-bastionhost` (IP externo efemero na nic0)

```bash
gcloud compute instances create "$BASTION_VM" \
  --zone="$ZONE" \
  --machine-type=e2-medium \
  --image-family="$WIN_IMAGE_FAMILY" \
  --image-project=windows-cloud \
  --boot-disk-size=50GB \
  --tags="$BASTION_TAG" \
  --network-interface=subnet="$SUBNET_NAME" \
  --network-interface=subnet=default,no-address
```

> Click **Check my progress**: *Create the vm-bastionhost instance*.

### Passo 3: Criar usuario e resetar as senhas

Aguarde 2-3 minutos apos a criacao (o Windows precisa terminar o primeiro boot).

```bash
gcloud compute reset-windows-password "$BASTION_VM" --user "$WIN_USER" --zone "$ZONE" --quiet
gcloud compute reset-windows-password "$SECURE_VM"  --user "$WIN_USER" --zone "$ZONE" --quiet
```

Anote **usuario e senha de cada VM**: as credenciais sao diferentes. Alternativa: Console > VM > **Set Windows password**.

### Passo 4: Capturar os IPs para o RDP

```bash
export BASTION_EXTERNAL_IP="$(gcloud compute instances describe "$BASTION_VM" --zone="$ZONE" \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)')"
export SECURE_INTERNAL_IP="$(gcloud compute instances describe "$SECURE_VM" --zone="$ZONE" \
  --format='get(networkInterfaces[0].networkIP)')"
echo "Bastion (RDP externo): $BASTION_EXTERNAL_IP"
echo "Securehost (RDP interno a partir do bastion): $SECURE_INTERNAL_IP"
```

### Explicacao dos parametros
- `--network-interface=subnet=...,no-address`: NIC sem IP externo.
- `--network-interface=subnet=...` (sem `no-address`): NIC com IP externo efemero.
- `--image-project=windows-cloud`: projeto publico das imagens Windows.
- `reset-windows-password --user`: cria o usuario (se nao existir) e gera uma senha nova.

### Resultado esperado
- `vm-securehost`: 2 NICs, nenhuma com IP externo.
- `vm-bastionhost`: 2 NICs, IP externo so na `nic0`.

---

## TAREFA 3: Conectar no servidor seguro e configurar o IIS

### Conceito
O acesso ao `vm-securehost` e em dois saltos: internet -> RDP no bastion -> RDP interno no securehost. Dentro dele, instala-se o papel **Web Server (IIS)**.

### Opcao A: RDP manual (passo do lab)

1. Conecte via RDP em `BASTION_EXTERNAL_IP` com `app_admin` e a senha do bastion (Console > **RDP**, ou cliente RDP local).
2. No bastion, abra `mstsc.exe` e conecte em `SECURE_INTERNAL_IP` com `app_admin` e a senha do securehost.
3. No `vm-securehost`: **Server Manager > Add roles and features > Role-based or feature-based installation > Web Server (IIS)** > instalar.

Ou, no PowerShell (como administrador) dentro do `vm-securehost`:

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
Get-WindowsFeature -Name Web-Server
```

### Opcao B: Startup script PowerShell (sem RDP)

O IIS e instalado a partir dos arquivos locais do Windows, sem precisar de internet:

```bash
gcloud compute instances add-metadata "$SECURE_VM" --zone="$ZONE" \
  --metadata=windows-startup-script-ps1='Install-WindowsFeature -Name Web-Server -IncludeManagementTools'

gcloud compute instances reset "$SECURE_VM" --zone="$ZONE"
```

Acompanhar pela serial:

```bash
gcloud compute instances get-serial-port-output "$SECURE_VM" --zone="$ZONE" \
  | grep -i -E 'windows-startup-script|Web-Server' | tail -n 20
```

### Explicacao dos parametros
- `mstsc.exe`: cliente Remote Desktop do Windows.
- `Install-WindowsFeature -Name Web-Server`: instala o papel IIS.
- `windows-startup-script-ps1`: metadata executada no boot do Windows.

### Resultado esperado
- Papel **Web Server (IIS)** instalado no `vm-securehost`.

> Click **Check my progress**: *Configure the IIS web server software*.

---

## Validacao

```bash
gcloud compute networks subnets list --network="$VPC_NAME" \
  --format="table(name,region,ipCidrRange)"

gcloud compute firewall-rules list --filter="network:${VPC_NAME}" \
  --format="table(name,sourceRanges.list(),sourceTags.list(),targetTags.list(),allowed[].map().firewall_rule().list())"

gcloud compute instances list --filter="name~'vm-(securehost|bastionhost)'" \
  --format="table(name,zone.basename(),networkInterfaces[].network.basename().list(),networkInterfaces[].accessConfigs[0].natIP.list())"
```

Checklist esperado:
- `securenetwork` com uma subnet na `REGION`;
- RDP externo so para a tag `bastion`; RDP interno do bastion para `securehost`;
- `vm-securehost` sem IP externo; `vm-bastionhost` com IP externo so na `nic0`;
- ambas com segunda NIC na `default`;
- IIS instalado no `vm-securehost`.

---

## Resumo Visual da Arquitetura

```text
[Internet]
    |
    | RDP tcp:3389 (fw: 0.0.0.0/0 -> tag bastion)
    v
+------------------------- VPC securenetwork (REGION) -----------------+
|  [vm-bastionhost] nic0 + IP externo --RDP 3389--> [vm-securehost]    |
|   tag bastion                        (fw: tag      nic0 sem IP ext.  |
|                                       bastion ->   tag securehost    |
|                                       securehost)  IIS instalado     |
+----------------------------------------------------------------------+
        | nic1 (interna)                         | nic1 (interna)
        v                                        v
+------------------------- VPC default (monitoramento) ----------------+
+----------------------------------------------------------------------+
```

---

## Troubleshooting

- Nao conecta no bastion via RDP:
  - use o IP **externo** da `nic0`; confira a regra `tcp:3389` para a tag `bastion`; algumas redes locais bloqueiam a porta 3389.
- Bastion conecta, mas nao chega no securehost:
  - use o IP interno da `nic0` do securehost (`securenetwork`) e confira a regra interna `bastion -> securehost`.
- `reset-windows-password` falha logo apos criar a VM:
  - o Windows ainda esta no primeiro boot; aguarde alguns minutos e repita.
- Erro ao criar VM com duas NICs na mesma VPC:
  - cada NIC precisa estar em uma VPC diferente (`securenetwork` e `default`).
- IIS nao instalou pela Opcao B:
  - confira a serial; se necessario, instale pela Opcao A (RDP + PowerShell).

---

## Limpeza (Opcional)

```bash
gcloud compute instances delete "$SECURE_VM" "$BASTION_VM" --zone="$ZONE" --quiet
gcloud compute firewall-rules delete "$FW_RDP_EXTERNAL" "$FW_RDP_INTERNAL" --quiet
gcloud compute networks subnets delete "$SUBNET_NAME" --region="$REGION" --quiet
gcloud compute networks delete "$VPC_NAME" --quiet
```

---

## Conceitos-Chave

- Bastion host: unico ponto de entrada exposto; reduz a superficie de ataque das VMs internas.
- VPC custom: sem regras de entrada por padrao; tudo e liberado explicitamente.
- Multiplas NICs: cada interface em uma VPC distinta (producao x monitoramento).
- Network tags: segmentam regras de firewall por funcao (bastion, securehost).
- `reset-windows-password`: cria usuario e senha em VMs Windows via API.

---

## Fluxo Final

1. Definir variaveis (`REGION`/`ZONE` do lab).
2. Criar a VPC `securenetwork` e a subnet.
3. Criar as regras de RDP (internet -> bastion, bastion -> securehost).
4. Criar `vm-securehost` (2 NICs, sem IP externo) e `vm-bastionhost` (2 NICs, IP externo na nic0).
5. Resetar as senhas com o usuario `app_admin`.
6. RDP no bastion -> RDP no securehost -> instalar IIS (ou startup script PowerShell).
