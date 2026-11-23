# Desafio Final - Implement Cloud Security Fundamentals on Google Cloud

## Introducao

Este guia converte o desafio final (GSP342) para execucao via Google Cloud CLI, com foco em IAM de menor privilegio e GKE privado.

Objetivo tecnico:
- revisar o papel IAM customizado de Storage (**ja criado pelo lab**);
- verificar a service account dedicada do cluster (**ja criada pelo lab**);
- vincular os 3 papeis obrigatorios de operacao + o papel customizado;
- criar cluster GKE Standard privado (zonal, 1 node) na subnet `orca-build-subnet`;
- validar acesso administrativo via `orca-jumphost` e fazer deploy de app de teste.

Regras do lab:
- **Nao** rode `gcloud iam roles create` nem `gcloud iam service-accounts create`: ambos ja existem.
- A policy IAM do projeto tem bindings condicionais: use `--condition=None` em todo `add-iam-policy-binding`.
- Use apenas `us-east1` / `us-east1-c`.
- Novos recursos devem ter o prefixo `orca-`.

---

## Pre-requisitos e Variaveis

### Conceito

Os valores de role, service account e cluster mudam a cada lab e aparecem no painel **Lab setup and access**. Centralize tudo em variaveis.

### Passo 1: Definir variaveis-base

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID

export REGION="us-east1"
export ZONE="us-east1-c"

# Valores do painel do lab
export CUSTOM_ROLE_ID="orca_storage_editor_219"
export CLUSTER_SA_ID="orca-private-cluster-612-sa"
export CLUSTER_NAME="orca-cluster-466"

# Recursos existentes no ambiente do desafio
export BUILD_VPC="orca-build-vpc"
export BUILD_SUBNET="orca-build-subnet"
export JUMPHOST_NAME="orca-jumphost"

# Derivados
export CUSTOM_ROLE_PATH="projects/${PROJECT_ID}/roles/${CUSTOM_ROLE_ID}"
export CLUSTER_SA_EMAIL="${CLUSTER_SA_ID}@${PROJECT_ID}.iam.gserviceaccount.com"
```

### Passo 2: Validar contexto

```bash
gcloud config set project "${PROJECT_ID}"
gcloud config set compute/zone "${ZONE}"
gcloud auth list
```

**Resultado esperado:**
- conta ativa do aluno;
- projeto correto e zona `us-east1-c` como default.

---

## TAREFA 1: Revisar o papel IAM customizado

### Conceito

O papel `orca_storage_editor_219` foi pre-criado e deve conter as 5 permissoes de Storage pedidas pelo time de desenvolvimento. O caminho completo `projects/<PROJECT_ID>/roles/<ROLE>` sera usado na Tarefa 3.

### Passo 1: Descrever o papel

```bash
gcloud iam roles describe "${CUSTOM_ROLE_ID}" --project="${PROJECT_ID}"
```

### Passo 2: Conferir as permissoes

```bash
gcloud iam roles describe "${CUSTOM_ROLE_ID}" --project="${PROJECT_ID}" \
	--format="value(includedPermissions)" | tr ';' '\n'

echo "${CUSTOM_ROLE_PATH}"
```

**Resultado esperado:**
- `storage.buckets.get`
- `storage.objects.get`
- `storage.objects.list`
- `storage.objects.update`
- `storage.objects.create`
- `name: projects/<PROJECT_ID>/roles/orca_storage_editor_219`

> Click **Check my progress**: *Check that a Custom security role has been created*.

---

## TAREFA 2: Verificar a service account do cluster

### Conceito

A service account `orca-private-cluster-612-sa` foi pre-criada e sera a identidade dos nodes do cluster (menor privilegio, em vez da Compute default SA).

### Passo 1: Confirmar que a SA existe

```bash
gcloud iam service-accounts describe "${CLUSTER_SA_EMAIL}" \
	--format="value(email,displayName,disabled)"

# alternativa do lab
gcloud iam service-accounts list --filter="email:${CLUSTER_SA_EMAIL}"
```

**Resultado esperado:**
- SA `orca-private-cluster-612-sa@<PROJECT_ID>.iam.gserviceaccount.com` listada e nao desabilitada.

> Click **Check my progress**: *Check that a new service account has been created*.

---

## TAREFA 3: Vincular papeis IAM na service account

### Conceito

Vincular os 3 papeis minimos recomendados pelo guia *Harden your cluster's security* (logging/monitoring) e o papel customizado de Storage. Voce so tem permissao para conceder exatamente esses 4 papeis.

### Passo 1: Vincular papeis built-in

```bash
for ROLE in \
	roles/monitoring.viewer \
	roles/monitoring.metricWriter \
	roles/logging.logWriter
do
	gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
		--member="serviceAccount:${CLUSTER_SA_EMAIL}" \
		--role="${ROLE}" \
		--condition=None
done
```

### Passo 2: Vincular papel customizado (caminho completo)

```bash
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
	--member="serviceAccount:${CLUSTER_SA_EMAIL}" \
	--role="${CUSTOM_ROLE_PATH}" \
	--condition=None
```

### Explicacao dos parametros

- `--role="projects/.../roles/..."`: papel customizado de projeto exige o caminho completo; `roles/<ID>` retorna `INVALID_ARGUMENT` (o prefixo `roles/` e reservado aos papeis predefinidos).
- `--condition=None`: adiciona o binding sem condicao e evita o prompt interativo causado pelos bindings condicionais da policy.

### Passo 3: Validar bindings

```bash
gcloud projects get-iam-policy "${PROJECT_ID}" \
	--flatten="bindings[].members" \
	--filter="bindings.members:serviceAccount:${CLUSTER_SA_EMAIL}" \
	--format="table(bindings.role)"
```

**Resultado esperado:**
- `roles/monitoring.viewer`
- `roles/monitoring.metricWriter`
- `roles/logging.logWriter`
- `projects/<PROJECT_ID>/roles/orca_storage_editor_219`

> Click **Check my progress**: *Check that the correct built in and custom security roles have been bound to the new service account*.

---

## TAREFA 4: Criar e configurar cluster GKE privado

### Conceito

O cluster deve ser GKE **Standard zonal**, privado, **sem endpoint publico**, com `master authorized networks` restrito ao IP **interno** do jumphost (`/32`). Para caber na quota do projeto: 1 node e disco `pd-standard` de 50 GB.

### Passo 1: Coletar IP interno do jumphost

```bash
export JUMPHOST_IP="$(gcloud compute instances describe "${JUMPHOST_NAME}" \
	--zone="${ZONE}" \
	--format='value(networkInterfaces[0].networkIP)')"

echo "${JUMPHOST_IP}/32"
```

### Passo 2: Criar cluster privado

```bash
gcloud container clusters create "${CLUSTER_NAME}" \
	--project="${PROJECT_ID}" \
	--zone="${ZONE}" \
	--num-nodes=1 \
	--disk-type=pd-standard \
	--disk-size=50 \
	--network="${BUILD_VPC}" \
	--subnetwork="${BUILD_SUBNET}" \
	--service-account="${CLUSTER_SA_EMAIL}" \
	--enable-ip-alias \
	--enable-private-nodes \
	--enable-private-endpoint \
	--master-ipv4-cidr=172.16.0.0/28 \
	--enable-master-authorized-networks \
	--master-authorized-networks="${JUMPHOST_IP}/32"
```

### Explicacao dos parametros

- `--zone` (sem `--region`): cluster zonal; regional ou Autopilot estouram a quota de CPU/SSD.
- `--num-nodes=1 --disk-type=pd-standard --disk-size=50`: evita o default de 3 nodes com `pd-balanced`, que trava o deploy.
- `--service-account`: nodes rodam com a SA dedicada da Tarefa 2.
- `--enable-private-nodes`: nodes sem IP publico.
- `--enable-private-endpoint`: control plane acessivel apenas por IP interno.
- `--master-ipv4-cidr=172.16.0.0/28`: range `/28` do control plane.
- `--master-authorized-networks="${JUMPHOST_IP}/32"`: so o jumphost acessa o master.

A criacao leva alguns minutos.

### Passo 3 (opcional): Ajustar a rede autorizada depois da criacao

So e necessario se o cluster foi criado sem `--master-authorized-networks`:

```bash
gcloud container clusters update "${CLUSTER_NAME}" \
	--zone="${ZONE}" \
	--enable-master-authorized-networks \
	--master-authorized-networks="${JUMPHOST_IP}/32"
```

### Passo 4: Validar configuracao do cluster

```bash
gcloud container clusters describe "${CLUSTER_NAME}" \
	--zone="${ZONE}" \
	--format="yaml(status,privateClusterConfig,masterAuthorizedNetworksConfig,network,subnetwork,nodeConfig.serviceAccount,nodeConfig.diskType,currentNodeCount)"
```

**Resultado esperado:**
- `status: RUNNING`;
- `enablePrivateNodes: true` e `enablePrivateEndpoint: true`;
- `masterAuthorizedNetworksConfig` contendo `<JUMPHOST_IP>/32`;
- `network: orca-build-vpc` e `subnetwork: orca-build-subnet`;
- `nodeConfig.serviceAccount` igual a `CLUSTER_SA_EMAIL`;
- `diskType: pd-standard` e `currentNodeCount: 1`.

> Click **Check my progress**: *Confirm that a Kubernetes Engine private cluster has been deployed correctly*.

---

## TAREFA 5: Deploy de aplicacao de teste no cluster privado

### Conceito

Com endpoint privado habilitado, o acesso de gerenciamento so funciona a partir da mesma VPC do cluster, via `orca-jumphost`, usando `--internal-ip` no `get-credentials`. Use as credenciais da SA ja configurada na VM (**nao** rode `gcloud auth login` no jumphost).

### Passo 1: Acessar jumphost

```bash
gcloud compute ssh "${JUMPHOST_NAME}" --zone="${ZONE}"
```

### Passo 2: No jumphost, instalar kubectl e plugin de autenticacao GKE

```bash
sudo apt-get update && sudo apt-get install -y kubectl google-cloud-cli-gke-gcloud-auth-plugin
echo "export USE_GKE_GCLOUD_AUTH_PLUGIN=True" >> ~/.bashrc
source ~/.bashrc
```

### Passo 3: Obter credenciais do cluster pelo endpoint interno

As variaveis do Cloud Shell nao existem no jumphost; defina-as novamente:

```bash
export PROJECT_ID="$(gcloud config get-value project)"
export ZONE="us-east1-c"
export CLUSTER_NAME="orca-cluster-466"

gcloud container clusters get-credentials "${CLUSTER_NAME}" \
	--internal-ip \
	--project="${PROJECT_ID}" \
	--zone="${ZONE}"
```

### Passo 4: Fazer deploy da aplicacao de teste

```bash
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
kubectl get deployments
kubectl get pods
```

Opcional (exposicao via load balancer; a app escuta na porta 8080):

```bash
kubectl expose deployment hello-server --type=LoadBalancer --port=80 --target-port=8080
kubectl get svc hello-server
```

**Resultado esperado:**
- deployment `hello-server` criado e pod em `Running`.

> Click **Check my progress**: *Check that an application has been deployed to a private Kubernetes Engine cluster*.

---

## Validacao Final

Execute no Cloud Shell:

```bash
# Papel customizado
gcloud iam roles describe "${CUSTOM_ROLE_ID}" --project="${PROJECT_ID}" \
	--format="value(name,includedPermissions)"

# Service account
gcloud iam service-accounts describe "${CLUSTER_SA_EMAIL}" --format="value(email)"

# Bindings IAM da service account
gcloud projects get-iam-policy "${PROJECT_ID}" \
	--flatten="bindings[].members" \
	--filter="bindings.members:serviceAccount:${CLUSTER_SA_EMAIL}" \
	--format="table(bindings.role)"

# Cluster privado e autorizacao de rede
gcloud container clusters describe "${CLUSTER_NAME}" --zone="${ZONE}" \
	--format="value(privateClusterConfig.enablePrivateNodes,privateClusterConfig.enablePrivateEndpoint)"

gcloud container clusters describe "${CLUSTER_NAME}" --zone="${ZONE}" \
	--format="yaml(masterAuthorizedNetworksConfig.cidrBlocks)"
```

---

## Troubleshooting

- `INVALID_ARGUMENT` ao vincular o papel customizado:
	use `projects/${PROJECT_ID}/roles/${CUSTOM_ROLE_ID}`, nunca `roles/${CUSTOM_ROLE_ID}`.
- Prompt pedindo condicao no `add-iam-policy-binding`:
	faltou `--condition=None`.
- Cluster travado em `PROVISIONING` ou erro de quota:
	confirme zonal (`--zone`), `--num-nodes=1`, `--disk-type=pd-standard --disk-size=50`. Se necessario, delete e recrie.
- `kubectl` sem conexao / timeout no jumphost:
	confirme `--internal-ip` no `get-credentials` e que o IP **interno** do jumphost (`/32`) esta em `masterAuthorizedNetworksConfig`.
- Erro de plugin de autenticacao:
	reinstale `google-cloud-cli-gke-gcloud-auth-plugin` e confirme `USE_GKE_GCLOUD_AUTH_PLUGIN=True`.
- Comandos dentro do jumphost com variaveis vazias:
	as variaveis do Cloud Shell nao sao herdadas; rode o bloco de `export` do Passo 3 da Tarefa 5.

---

## Limpeza Opcional

Nao remova o papel customizado nem a service account: ambos sao recursos pre-criados e verificados pelo lab.

```bash
gcloud container clusters delete "${CLUSTER_NAME}" --zone="${ZONE}" --quiet
```

---

## Conceitos-Chave

- Menor privilegio para service account de cluster reduz superficie de risco.
- Papel customizado de projeto e referenciado pelo caminho completo `projects/<ID>/roles/<ROLE>`.
- Cluster privado com endpoint privado + master authorized networks aumenta seguranca operacional.
- Acesso administrativo ao GKE privado depende de caminho de rede interno (jumphost/proxy na mesma VPC).

---

## Fluxo Final

1. Definir variaveis com os valores do painel (role, SA, cluster) e zona `us-east1-c`.
2. Revisar o papel customizado `orca_storage_editor_219`.
3. Verificar a SA `orca-private-cluster-612-sa`.
4. Vincular os 3 papeis built-in + papel customizado (caminho completo, `--condition=None`).
5. Criar `orca-cluster-466` zonal, 1 node `pd-standard`, privado, em `orca-build-subnet`, com o IP interno do jumphost `/32`.
6. Conectar via `orca-jumphost` com `--internal-ip` e fazer deploy de `hello-server`.
