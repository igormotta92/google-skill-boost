# Conceitos e Passo a Passo - Securing Virtual Machines using Chrome Enterprise Premium

## Introdução

Este documento organiza o laboratório de acesso seguro a máquinas virtuais usando o Identity-Aware Proxy (IAP) no Google Cloud. A proposta é permitir conexão com VMs Linux e Windows sem expor IP público, usando tunelamento TCP controlado por IAM.

### Objetivo do laboratório

- Criar VMs sem IP externo para acesso administrativo seguro.
- Liberar somente o intervalo de origem usado pelo IAP.
- Conceder as permissões necessárias para acesso por túnel.
- Validar acesso SSH e RDP via IAP.

---

## TAREFA 1: Habilitar IAP TCP Forwarding no projeto

### Conceito

O IAP TCP Forwarding permite acessar instâncias Compute Engine por SSH ou RDP mesmo quando elas não possuem IP público. Em vez de abrir a VM para a internet, o tráfego passa pelo serviço do IAP e é autorizado com base em políticas IAM.

### Pré-requisito: habilitar a API do IAP

Antes de iniciar, confirme que a API Cloud Identity-Aware Proxy está habilitada no projeto.

### Passo 1: Criar as instâncias do laboratório

#### 1.1 Criar a VM Linux sem IP público

```bash
gcloud compute instances create linux-iap \
    --zone=us-east4-c \
    --machine-type=e2-micro \
    --subnet=default \
    --no-address \
    --image-family=debian-12 \
    --image-project=debian-cloud
```

**Explicação dos parâmetros:**
- `--zone=us-east4-c`: define a zona onde a VM será criada.
- `--machine-type=e2-micro`: usa uma máquina pequena, suficiente para o lab.
- `--subnet=default`: conecta a VM à sub-rede padrão.
- `--no-address`: impede a criação de IP externo.
- `--image-family=debian-12`: usa a família de imagens Debian 12.
- `--image-project=debian-cloud`: informa o projeto que mantém a imagem.

#### 1.2 Criar a VM Windows sem IP público

```bash
gcloud compute instances create windows-iap \
    --zone=us-east4-c \
    --subnet=default \
    --no-address \
    --image-family=windows-2016 \
    --image-project=windows-cloud
```

**Por quê?**
- Essa instância será acessada por RDP usando o túnel do IAP.
- O uso de `--no-address` garante que o acesso continue privado.

#### 1.3 Criar a máquina de conectividade Windows

```bash
gcloud compute instances create windows-connectivity \
    --zone=us-east4-c \
    --subnet=default \
    --image=iap-desktop-v001 \
    --image-project=qwiklabs-resources \
    --scopes=https://www.googleapis.com/auth/cloud-platform
```

**Por quê?**
- Essa VM serve como estação de apoio para testar o acesso RDP com o IAP Desktop.
- O escopo `cloud-platform` permite que a instância interaja com os recursos necessários do projeto.

### Passo 2: Criar a firewall rule para o IAP

```bash
gcloud compute firewall-rules create allow-ingress-from-iap \
    --direction=INGRESS \
    --network=default \
    --action=ALLOW \
    --rules=tcp:22,tcp:3389 \
    --source-ranges=35.235.240.0/20
```

**Por quê?**
- O IAP usa o intervalo `35.235.240.0/20` para estabelecer os túneis.
- A regra libera apenas SSH (`22`) e RDP (`3389`).
- Sem essa regra, a conexão do IAP chegaria à VPC, mas seria bloqueada no firewall.

### Passo 3: Conceder acesso IAM às instâncias

```bash
ZONE="us-east4-c"
STUDENT_EMAIL="student-02-36ebb8243546@qwiklabs.net"

# 480314111654-compute@developer.gserviceaccount.com
SA_EMAIL="$(gcloud compute instances describe windows-connectivity \
  --zone="$ZONE" \
  --format='value(serviceAccounts[0].email)')"

for VM in linux-iap windows-iap; do
  gcloud compute instances add-iam-policy-binding "$VM" \
    --zone="$ZONE" \
    --member="serviceAccount:${SA_EMAIL}" \
    --role="roles/iap.tunnelResourceAccessor"

  gcloud compute instances add-iam-policy-binding "$VM" \
    --zone="$ZONE" \
    --member="user:${STUDENT_EMAIL}" \
    --role="roles/iap.tunnelResourceAccessor"
done
```

**O que esse bloco faz:**
- Descobre o service account anexado à VM `windows-connectivity`.
- Concede o papel `roles/iap.tunnelResourceAccessor` nas VMs `linux-iap` e `windows-iap`.
- Libera o acesso tanto para a service account quanto para o usuário do laboratório.

**Resultado esperado:**
- As entidades autorizadas passam a poder abrir túneis IAP diretamente para as instâncias.

### Passo 4: Validar acesso SSH à VM Linux

```bash
gcloud compute ssh linux-iap
```

**Por quê?**
- Esse comando confirma que a instância Linux pode ser acessada por SSH via IAP sem IP externo.

**Resultado esperado:**
- A sessão SSH é aberta com sucesso na VM `linux-iap`.

### Passo 5: Preparar o acesso RDP à VM Windows

#### 5.1 Credenciais do ambiente

```text
Usuário: student_02_36ebb8243
Senha: *SPt4A5vL>l&D=P
```

#### 5.2 Executar o instalador do IAP Desktop

```text
c:\Users\student_02_36ebb8243\Downloads>IapDesktopX64.msi
```

**Por quê?**
- O IAP Desktop facilita a conexão RDP com túneis IAP a partir de um ambiente Windows.

### Passo 6: Abrir túnel IAP para a VM Windows

#### 6.1 Observação para uso com PuTTY

Se você estiver usando o PuTTY como apoio para tunelamento ou encaminhamento de portas, ajuste a configuração abaixo antes de testar a conexão:

```text
PuTTY Window > clique com o botão direito > Change Settings > SSH > Tunnels > marcar "Local ports accept connections from other hosts"
```

**Por quê?**
- Essa opção permite que a porta local encaminhada aceite conexões que não fiquem restritas apenas ao host local.
- Ela só é relevante quando o fluxo do laboratório exigir PuTTY no caminho.
- Para o comando `gcloud compute start-iap-tunnel` executado localmente, o ponto principal continua sendo usar a porta retornada pelo próprio túnel.


```bash
gcloud compute start-iap-tunnel windows-iap 3389 \
    --local-host-port=localhost:0 \
    --zone=us-east4-c
```

**Explicação dos parâmetros:**
- `windows-iap`: nome da VM de destino.
- `3389`: porta do RDP.
- `--local-host-port=localhost:0`: escolhe automaticamente uma porta local disponível.
- `--zone=us-east4-c`: zona da instância.

**Saída esperada:**

```text
Listening on port [50545].
```

### Passo 7: Conectar via Remote Desktop

Use o endpoint local informado pelo túnel:

```text
Remote Desktop
localhost:50545
```

**Resultado esperado:**
- O cliente RDP se conecta à VM `windows-iap` usando a porta local publicada pelo IAP.

### Resumo Visual - Tarefa 1

```text
[Administrador / Estação Windows]
        ↓
[IAP TCP Forwarding]
        ↓
[Firewall: allow tcp:22,tcp:3389 from 35.235.240.0/20]
        ↓
[VPC default - us-east4-c]
        ↓
   linux-iap          windows-iap
 (sem IP externo)   (sem IP externo)
      SSH                 RDP
        ↑                  ↑
        └──── windows-connectivity / IAP Desktop ────┘
```

**Fluxo:**
1. O usuário autenticado solicita acesso à VM pelo IAP.
2. O IAP valida IAM e cria o túnel TCP.
3. A firewall rule permite apenas o tráfego vindo do intervalo oficial do IAP.
4. A conexão chega à VM privada sem necessidade de IP público.