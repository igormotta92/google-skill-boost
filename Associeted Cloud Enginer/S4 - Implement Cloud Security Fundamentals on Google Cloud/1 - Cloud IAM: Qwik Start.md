# Cloud IAM: Qwik Start

## Introducao

Este guia converte o lab para execucao via Google Cloud CLI, mantendo a mesma ordem logica das tarefas:

- inspecionar papeis de IAM no nivel de projeto;
- criar bucket Cloud Storage para teste de acesso;
- remover o acesso de projeto do Username 2;
- conceder acesso granular ao Cloud Storage com papel especifico.

O foco e demonstrar, na pratica, a diferenca entre permissao ampla de projeto (primitive roles) e permissao especifica por servico (Storage Object Viewer).

---

## Pre-requisitos e Variaveis

### Conceito

Como o lab usa dois usuarios (Username 1 e Username 2), os comandos abaixo devem ser executados em sessoes separadas do Cloud Shell, autenticadas com cada credencial correspondente.

### Passo 1: Definir variaveis comuns (sessao Username 1)

```bash
# Ajuste conforme ambiente do lab
export PROJECT_ID="$(gcloud config get-value project)"
export USERNAME_2="student-02-f7a8d6517748@qwiklabs.net"

# Bucket deve ser globalmente unico
export BUCKET_NAME="qwiklabs-gcp-02-bd5d30d114c0"

# Regiao multirregional de exemplo (ajuste conforme ambiente)
export BUCKET_LOCATION="US"
```

### Passo 2: Confirmar projeto ativo

```bash
gcloud config list --format="text(core.project,core.account)"
```

### Explicacao dos parametros

- `PROJECT_ID`: projeto em que as permissoes IAM serao alteradas.
- `USERNAME_2`: principal que inicialmente tem `roles/viewer` e depois perde/acessa recursos de forma granular.
- `BUCKET_NAME`: nome unico global do bucket.
- `BUCKET_LOCATION`: localizacao multirregional do bucket (ex.: `US`, `EU`).

---

## Tarefa 1. Explore o console IAM e os papeis no nivel de projeto

### Conceito

Identificar os papeis basicos (primitive roles) e validar, por CLI, que o Username 1 possui privilegios para alterar IAM enquanto o Username 2 tem acesso limitado.

### Passos com comandos CLI

1. Listar bindings IAM do projeto (sessao Username 1):

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
	--flatten="bindings[].members" \
	--format="table(bindings.role,bindings.members)"
```

2. Filtrar papeis do Username 2 (sessao Username 1):

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
	--flatten="bindings[].members" \
	--filter="bindings.members:$USERNAME_2" \
	--format="table(bindings.role,bindings.members)"
```

3. (Opcional) Confirmar na sessao Username 2 que nao e possivel alterar IAM:

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
	--member="user:$USERNAME_2" \
	--role="roles/viewer"
```

### Explicacao dos parametros

- `get-iam-policy`: le a politica IAM atual do projeto.
- `--flatten="bindings[].members"`: transforma lista de membros em linhas individuais para facilitar filtro.
- `--filter`: restringe a saida para um principal especifico.
- `add-iam-policy-binding`: usado aqui apenas como teste de permissao; com `roles/viewer`, deve falhar por falta de `setIamPolicy`.

### Resultado esperado

- Username 1 consegue listar/gerenciar IAM.
- Username 2 aparece com papel `roles/viewer`.
- Tentativa de alteracao IAM com Username 2 retorna erro de permissao.

---

## Tarefa 2. Prepare um bucket do Cloud Storage para teste de acesso

### Conceito

Criar um bucket e um objeto de teste para validar o comportamento de acesso de leitura.

### Passos com comandos CLI

1. Criar bucket multirregional (sessao Username 1):

```bash
gcloud storage buckets create "gs://$BUCKET_NAME" \
	--project="$PROJECT_ID" \
	--location="$BUCKET_LOCATION"
```

2. Criar arquivo local e fazer upload com nome `sample.txt`:

```bash
echo "sample file for iam lab" > sample.txt
gcloud storage cp sample.txt "gs://$BUCKET_NAME/sample.txt"
```

3. Validar existencia do bucket e objeto:

```bash
gcloud storage ls "gs://$BUCKET_NAME"
gcloud storage ls "gs://$BUCKET_NAME/sample.txt"
```

4. Validar que Username 2 (com `roles/viewer`) enxerga bucket/objeto (sessao Username 2):

```bash
gcloud storage ls
gcloud storage ls "gs://$BUCKET_NAME"
```

### Explicacao dos parametros

- `buckets create`: cria bucket Cloud Storage.
- `--location`: define localizacao (usar multirregiao conforme lab).
- `storage cp`: envia arquivo local para objeto no bucket.
- `storage ls`: usado para verificacao de visibilidade e conteudo.

### Resultado esperado

- Bucket criado com sucesso.
- Objeto `sample.txt` presente no bucket.
- Username 2 ainda consegue visualizar recursos por ter `roles/viewer` no projeto.

---

## Tarefa 3. Remova o acesso ao projeto

### Conceito

Remover o papel de visualizacao em nivel de projeto do Username 2 para cortar acesso amplo ao projeto.

### Passos com comandos CLI

1. Remover `roles/viewer` do Username 2 (sessao Username 1):

```bash
gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
	--member="user:$USERNAME_2" \
	--role="roles/viewer"
```

2. Validar que binding foi removido:

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
	--flatten="bindings[].members" \
	--filter="bindings.members:$USERNAME_2" \
	--format="table(bindings.role,bindings.members)"
```

3. Testar perda de acesso na sessao Username 2:

```bash
gcloud storage ls "gs://$BUCKET_NAME"
```

### Explicacao dos parametros

- `remove-iam-policy-binding`: remove papel de um principal no escopo do projeto.
- A propagacao pode levar alguns segundos (tipicamente ate ~80s no contexto do lab).

### Resultado esperado

- Username 2 nao aparece mais com `roles/viewer`.
- Comandos de listagem no bucket passam a retornar erro de permissao (apos propagacao).

---

## Tarefa 4. Adicione permissoes do Cloud Storage

### Conceito

Conceder acesso minimo necessario apenas para objetos do Cloud Storage, sem devolver acesso de visualizacao geral do projeto.

### Passos com comandos CLI

1. Conceder papel `roles/storage.objectViewer` no projeto (sessao Username 1):

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
	--member="user:$USERNAME_2" \
	--role="roles/storage.objectViewer"
```

2. Validar binding criado:

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
	--flatten="bindings[].members" \
	--filter="bindings.members:$USERNAME_2" \
	--format="table(bindings.role,bindings.members)"
```

3. Testar acesso ao bucket pela sessao Username 2:

```bash
gcloud storage ls "gs://$BUCKET_NAME"
```

### Explicacao dos parametros

- `roles/storage.objectViewer`: permite listar/ler objetos no Cloud Storage, sem permitir alteracao.
- Mantem principio de menor privilegio: acesso especifico ao servico em vez de acesso amplo de projeto.

### Resultado esperado

- Username 2 continua sem `roles/viewer` no projeto.
- Username 2 consegue listar `gs://$BUCKET_NAME/sample.txt` via CLI.

---

## Validacao

Execute as verificacoes abaixo para confirmar o estado final:

```bash
# Sessao Username 1: papeis finais do Username 2
gcloud projects get-iam-policy "$PROJECT_ID" \
	--flatten="bindings[].members" \
	--filter="bindings.members:$USERNAME_2" \
	--format="table(bindings.role,bindings.members)"

# Sessao Username 2: listar objeto do bucket
gcloud storage ls "gs://$BUCKET_NAME"
```

Estado esperado:

- `roles/viewer` ausente para Username 2.
- `roles/storage.objectViewer` presente para Username 2.
- listagem do bucket retorna `sample.txt`.

---

## Troubleshooting/Solucao de problemas

- `AccessDeniedException` logo apos mudar IAM:
	aguarde 60-120 segundos e repita o comando (tempo de propagacao IAM).

- Erro ao criar bucket por nome ja existente:
	altere `BUCKET_NAME` para outro valor globalmente unico e execute novamente.

- Comando executado com conta errada:
	valide conta ativa com:

```bash
gcloud auth list
gcloud config list --format="text(core.account,core.project)"
```

---

## Limpeza (opcional)

Se quiser remover os recursos apos concluir o lab:

```bash
# Sessao Username 1
gcloud storage rm --recursive "gs://$BUCKET_NAME"
gcloud storage buckets delete "gs://$BUCKET_NAME"

gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
	--member="user:$USERNAME_2" \
	--role="roles/storage.objectViewer"
```

---

## Conceitos-chave

- Primitive roles (`roles/viewer`, `roles/editor`, `roles/owner`) atuam em nivel de projeto e sao amplos.
- IAM e propagado de forma eventual; mudancas nao sao instantaneas.
- `roles/storage.objectViewer` exemplifica permissao granular por servico.
- Menor privilegio reduz risco: conceder somente o acesso necessario.

---

## Fluxo Final

1. Inspecionar IAM e confirmar diferenca de privilegios entre Username 1 e Username 2.
2. Criar bucket + `sample.txt` para cenario de teste de acesso.
3. Remover `roles/viewer` do Username 2 e validar perda de acesso.
4. Conceder `roles/storage.objectViewer` e validar acesso apenas ao Cloud Storage.