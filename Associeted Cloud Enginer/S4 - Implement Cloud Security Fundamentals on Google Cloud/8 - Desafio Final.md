# Desafio Final - Implement Cloud Security Fundamentals on Google Cloud

## Introducao

Este guia converte o desafio final para execucao via Google Cloud CLI, com foco em IAM de menor privilegio e GKE privado.

Objetivo tecnico:
- criar um papel IAM customizado para Cloud Storage;
- criar uma service account dedicada para o cluster;
- vincular papeis obrigatorios de operacao + papel customizado;
- criar cluster GKE privado em subnet especifica;
- validar acesso administrativo via jumphost e fazer deploy de app de teste.

---

## Pre-requisitos e Variaveis

### Conceito

Os nomes mostrados no enunciado (por exemplo, `Cluster Name`, `Service Account`) podem ser rotulos do desafio. Na CLI, alguns recursos exigem IDs sem espaco. Use variaveis para ajustar aos valores exigidos pelo painel de score.

### Passo 1: Definir variaveis-base

```bash
export PROJECT_ID="$(gcloud config get-value project)"

# Use os valores exibidos no painel do lab
export REGION="us-central1"
export ZONE="us-central1-f"

# Prefixo obrigatorio para novos recursos
export PREFIX="orca"

# Recursos existentes no ambiente do desafio
export BUILD_VPC="$PREFIX-build-vpc"
export BUILD_SUBNET="$PREFIX-build-subnet"
export JUMPHOST_NAME="$PREFIX-jumphost"

# Novos recursos
export CUSTOM_ROLE_ID="${PREFIX}_storage_editor_709"
export CUSTOM_ROLE_TITLE="Custom Security Role"
export CLUSTER_SA_ID="$PREFIX-private-cluster-154-sa"
export CLUSTER_SA_DISPLAY_NAME="Service Account"
export CLUSTER_NAME="$PREFIX-cluster-500"
```

### Passo 2: Validar contexto

```bash
gcloud auth list
gcloud config list project
gcloud services enable iam.googleapis.com container.googleapis.com compute.googleapis.com
```

**Resultado esperado:**
- conta ativa correta;
- projeto correto;
- APIs necessarias habilitadas.

---

## TAREFA 1: Criar papel IAM customizado

### Conceito

O papel customizado cobre permissoes de leitura basica de bucket e criacao/atualizacao de objetos no Cloud Storage.

### Passo 1: Criar papel

```bash
gcloud iam roles create "${CUSTOM_ROLE_ID}" \
	--project="${PROJECT_ID}" \
	--title="${CUSTOM_ROLE_TITLE}" \
	--description="Permissoes para criar e atualizar objetos do Orca" \
	--stage="GA" \
	--permissions="storage.buckets.get,storage.objects.get,storage.objects.list,storage.objects.update,storage.objects.create"
```

### Passo 2: Validar papel

```bash
gcloud iam roles describe "${CUSTOM_ROLE_ID}" --project="${PROJECT_ID}"
```

**Resultado esperado:**
- papel customizado existente com as 5 permissoes de `storage.*` solicitadas.

---

## TAREFA 2: Criar service account dedicada

### Conceito

O cluster privado deve rodar com uma service account dedicada (principio de menor privilegio).

### Passo 1: Criar service account

```bash
gcloud iam service-accounts create "${CLUSTER_SA_ID}" \
	--display-name="${CLUSTER_SA_DISPLAY_NAME}"

export CLUSTER_SA_EMAIL="${CLUSTER_SA_ID}@${PROJECT_ID}.iam.gserviceaccount.com"
```

### Passo 2: Validar service account

```bash
gcloud iam service-accounts describe "${CLUSTER_SA_EMAIL}"
```

**Resultado esperado:**
- service account criada e pronta para ser usada no GKE.

---

## TAREFA 3: Vincular papeis IAM na service account

### Conceito

Vincular os 3 papeis obrigatorios de operacao do GKE e o papel customizado de Storage.

### Passo 1: Vincular papeis built-in

```bash
for ROLE in \
	roles/monitoring.viewer \
	roles/monitoring.metricWriter \
	roles/logging.logWriter
do
	gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
		--member="serviceAccount:${CLUSTER_SA_EMAIL}" \
		--role="${ROLE}"
done
```

### Passo 2: Vincular papel customizado

```bash
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
	--member="serviceAccount:${CLUSTER_SA_EMAIL}" \
	--role="projects/${PROJECT_ID}/roles/${CUSTOM_ROLE_ID}"
```

### Passo 3: Validar bindings

```bash
gcloud projects get-iam-policy "${PROJECT_ID}" \
	--flatten="bindings[].members" \
	--filter="bindings.members:serviceAccount:${CLUSTER_SA_EMAIL}" \
	--format="table(bindings.role)"
```

**Resultado esperado:**
- lista contendo:
	- `roles/monitoring.viewer`
	- `roles/monitoring.metricWriter`
	- `roles/logging.logWriter`
	- `projects/${PROJECT_ID}/roles/${CUSTOM_ROLE_ID}`

---

## TAREFA 4: Criar e configurar cluster GKE privado

### Conceito

O cluster deve ser privado, sem endpoint publico, com `master authorized networks` restrito ao IP interno do jumphost.

### Passo 1: Coletar IP interno do jumphost

```bash
export JUMPHOST_IP="$(gcloud compute instances describe "${JUMPHOST_NAME}" \
	--zone="${ZONE}" \
	--format='value(networkInterfaces[0].networkIP)')"

echo "${JUMPHOST_IP}"
```

### Passo 2: Criar cluster privado

```bash
gcloud container clusters create "${CLUSTER_NAME}" \
	--project="${PROJECT_ID}" \
	--zone="${ZONE}" \
	--network="${BUILD_VPC}" \
	--subnetwork="${BUILD_SUBNET}" \
	--service-account="${CLUSTER_SA_EMAIL}" \
	--enable-ip-alias \
	--enable-private-nodes \
	--enable-private-endpoint \
	--enable-master-authorized-networks \
	--master-authorized-networks="${JUMPHOST_IP}/32" \
	--num-nodes="1"
```

### Passo 3: Garantir rede autorizada do jumphost (Configurações feitas durante criação)

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
	--format="yaml(privateClusterConfig,masterAuthorizedNetworksConfig,subnetwork,network,nodeConfig.serviceAccount)"
```

**Resultado esperado:**
- `enablePrivateNodes: true`;
- `enablePrivateEndpoint: true`;
- `masterAuthorizedNetworksConfig` contendo `JUMPHOST_IP/32`;
- cluster na VPC `orca-build-vpc` e subnet `orca-build-subnet`;
- service account do cluster igual a `CLUSTER_SA_EMAIL`.

---

## TAREFA 5: Deploy de aplicacao de teste no cluster privado

### Conceito

Com endpoint privado habilitado, o acesso de gerenciamento deve ser feito a partir do jumphost, usando `--internal-ip` para credenciais do cluster.

### Passo 1: Acessar jumphost

```bash
gcloud compute ssh "${JUMPHOST_NAME}" --zone="${ZONE}"
```

### Passo 2: No jumphost, instalar plugin de autenticacao GKE

```bash
sudo apt-get update
sudo apt-get install -y google-cloud-sdk-gke-gcloud-auth-plugin kubectl
echo 'export USE_GKE_GCLOUD_AUTH_PLUGIN=True' >> ~/.bashrc
source ~/.bashrc
```

### Passo 3: Obter credenciais do cluster pelo endpoint interno

```bash
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="us-central1"
export ZONE="us-central1-f"
export PREFIX="orca"
export CLUSTER_NAME="$PREFIX-cluster-500"

gcloud container clusters get-credentials "${CLUSTER_NAME}" \
	--project="${PROJECT_ID}" \
	--zone="${ZONE}" \
	--internal-ip
```

### Passo 4: Fazer deploy da aplicacao de teste

```bash
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
kubectl expose deployment hello-server --type=LoadBalancer --port=80 --target-port=8080
kubectl get pods
kubectl get svc hello-server
```

**Resultado esperado:**
- deployment `hello-server` criado e pod em `Running`;
- service criada com exposicao da aplicacao.

---

## Validacao Final

Execute estes checks para confirmar o desafio completo:

```bash
# Papel customizado
gcloud iam roles describe "${CUSTOM_ROLE_ID}" --project="${PROJECT_ID}" --format="value(name,title)"

# Service account
gcloud iam service-accounts describe "${CLUSTER_SA_EMAIL}" --format="value(email,displayName)"

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

- Erro de nome em recurso com espaco:
	para CLI, use IDs sem espaco (`orca-*`) e mantenha o nome humano em `--display-name`/`--title` quando aplicavel.
- Erro ao conectar no cluster privado:
	confirme `--internal-ip`, plugin `gke-gcloud-auth-plugin` e conexao a partir do `orca-jumphost`.
- Erro de permissao no cluster:
	valide se `CLUSTER_SA_EMAIL` tem os 4 papeis IAM exigidos.
- Erro de subnet/regiao:
	confira se `ZONE` pertence a `REGION` e se o create do cluster usa `--network="${BUILD_VPC}"` junto com `--subnetwork="${BUILD_SUBNET}"`.

---

## Limpeza Opcional

```bash
gcloud container clusters delete "${CLUSTER_NAME}" --zone="${ZONE}" --quiet
gcloud iam service-accounts delete "${CLUSTER_SA_EMAIL}" --quiet
gcloud iam roles delete "${CUSTOM_ROLE_ID}" --project="${PROJECT_ID}" --quiet
```

---

## Conceitos-Chave

- Menor privilegio para service account de cluster reduz superficie de risco.
- Cluster privado com endpoint privado + master authorized networks aumenta seguranca operacional.
- Acesso administrativo ao GKE privado depende de caminho de rede interno (jumphost/proxy).
- Roles customizados permitem granularidade de permissao alem dos papeis predefinidos.

---

## Fluxo Final

1. Definir variaveis e habilitar APIs.
2. Criar papel customizado de Storage.
3. Criar service account dedicada.
4. Vincular papeis obrigatorios + papel customizado.
5. Criar cluster privado em `orca-build-subnet` com endpoint privado.
6. Autorizar IP interno do `orca-jumphost` com `/32`.
7. Conectar via jumphost com `--internal-ip` e fazer deploy de `hello-server`.
