# 2 - IAM Custom Roles

## Introducao

Este guia converte o desafio para execucao via CLI com foco em Cloud IAM Custom Roles. O objetivo e criar, listar, atualizar, desabilitar, excluir e restaurar papeis customizados no nivel de projeto, aplicando o principio de menor privilegio.

Permissoes em IAM seguem o padrao:

`<service>.<resource>.<verb>`

Exemplo: `compute.instances.list`.

---

## Pre-requisitos e Variaveis

### Conceito

Antes de criar papeis customizados, padronize variaveis e confirme o escopo do projeto. Ajuste valores conforme ambiente do lab.

### Passo 1: Definir regiao padrao

```bash
gcloud config set compute/region us-central1
```

### Passo 2: Definir variaveis

```bash
# Ajuste conforme ambiente
export PROJECT_ID="${DEVSHELL_PROJECT_ID:-$(gcloud config get-value project)}"
export RESOURCE="//cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID"

# IDs das roles customizadas usadas neste desafio
export CUSTOM_ROLE_EDITOR_ID="editor"
export CUSTOM_ROLE_VIEWER_ID="viewer"
```

### Passo 3: Validar contexto

```bash
gcloud config list --format="text(core.account,core.project,compute.region)"
```

### Explicacao dos parametros

- `PROJECT_ID`: projeto onde as custom roles serao criadas.
- `RESOURCE`: recurso alvo para comandos de descoberta (`list-testable-permissions` e `list-grantable-roles`).
- `CUSTOM_ROLE_EDITOR_ID` e `CUSTOM_ROLE_VIEWER_ID`: IDs das roles customizadas no projeto.

---

## Tarefa 1. Ver permissoes disponiveis para um recurso

### Conceito

Antes de criar uma role customizada, verifique quais permissoes sao testaveis no recurso alvo.

### Passos com comandos CLI

```bash
gcloud iam list-testable-permissions "$RESOURCE"
```

### Explicacao dos parametros

- `list-testable-permissions`: lista permissoes que podem ser avaliadas para o recurso.
- `"$RESOURCE"`: caminho completo do recurso no Resource Manager.

### Resultado esperado

Saida com varias permissoes no formato `service.resource.verb` e estagios (`GA`, `BETA`, etc.).

---

## Tarefa 2. Obter metadados de uma role

### Conceito

Consultar metadados de roles predefinidas ou customizadas (descricao, permissoes, stage e etag).

### Passos com comandos CLI

```bash
# Exemplo com role predefinida
gcloud iam roles describe roles/viewer

# Exemplo com role customizada (apos criacao)
gcloud iam roles describe "$CUSTOM_ROLE_EDITOR_ID" --project "$DEVSHELL_PROJECT_ID"
```

### Explicacao dos parametros

- `roles/viewer`: nome completo de role predefinida.
- `--project`: necessario para consultar role customizada no escopo do projeto.
- `etag`: versao do recurso para controle de concorrencia em updates.

### Resultado esperado

Bloco YAML com `description`, `includedPermissions`, `name`, `stage`, `title` e `etag`.

---

## Tarefa 3. Ver roles concediveis no recurso

### Conceito

Listar roles que podem ser vinculadas ao recurso do projeto.

### Passos com comandos CLI

```bash
gcloud iam list-grantable-roles "$RESOURCE"
```

### Explicacao dos parametros

- `list-grantable-roles`: retorna roles aplicaveis ao recurso informado.

### Resultado esperado

Lista de roles com `name`, `title` e `description`.

---

## Tarefa 4. Criar custom roles

### Conceito

Criar roles customizadas de duas formas: por arquivo YAML e por flags.

### Passos com comandos CLI

1. Criar definicao YAML da role `editor`:

```bash
cat > role-definition.yaml <<'EOF'
title: "Role Editor"
description: "Edit access for App Versions"
stage: "ALPHA"
includedPermissions:
- appengine.versions.create
- appengine.versions.delete
EOF
```

2. Criar role customizada via arquivo:

```bash
gcloud iam roles create "$CUSTOM_ROLE_EDITOR_ID" \
	--project "$DEVSHELL_PROJECT_ID" \
	--file role-definition.yaml
```

3. Criar role customizada `viewer` via flags:

```bash
gcloud iam roles create "$CUSTOM_ROLE_VIEWER_ID" \
	--project "$DEVSHELL_PROJECT_ID" \
	--title "Role Viewer" \
	--description "Custom role description." \
	--permissions "compute.instances.get,compute.instances.list" \
	--stage "ALPHA"
```

### Explicacao dos parametros

- `roles create`: cria role customizada no escopo definido.
- `--file`: usa definicao YAML completa.
- `--permissions`: lista de permissoes separadas por virgula.
- `--stage`: estagio de lancamento da role (`ALPHA`, `BETA`, `GA`, `DISABLED`, `DEPRECATED`).

### Resultado esperado

Roles customizadas `editor` e `viewer` criadas em `projects/$DEVSHELL_PROJECT_ID/roles/...`.

---

## Tarefa 5. Listar custom roles

### Conceito

Validar criacao das roles e distinguir roles customizadas de roles predefinidas.

### Passos com comandos CLI

```bash
# Roles customizadas do projeto
gcloud iam roles list --project "$DEVSHELL_PROJECT_ID"

# Incluir roles deletadas
gcloud iam roles list --project "$DEVSHELL_PROJECT_ID" --show-deleted

# Roles predefinidas globais
gcloud iam roles list
```

### Explicacao dos parametros

- `--project`: restringe a listagem para roles customizadas do projeto.
- `--show-deleted`: inclui roles em estado excluido (janela de restauracao).

### Resultado esperado

Listagem contendo as roles `editor` e `viewer` no projeto.

---

## Tarefa 6. Atualizar custom role existente

### Conceito

Atualizar role por YAML (com `etag`) e por flags, evitando sobrescrever alteracoes concorrentes.

### Passos com comandos CLI

1. Exportar definicao atual da role `editor`:

```bash
gcloud iam roles describe editor --project "$DEVSHELL_PROJECT_ID"

gcloud iam roles describe "$CUSTOM_ROLE_EDITOR_ID" \
	--project "$DEVSHELL_PROJECT_ID" \
	--format=yaml > new-role-definition.yaml
```

2. Editar `new-role-definition.yaml` e adicionar em `includedPermissions`:

```text
- storage.buckets.get
- storage.buckets.list
```

3. Aplicar update da role `editor` via YAML:

```bash
gcloud iam roles update "$CUSTOM_ROLE_EDITOR_ID" \
	--project "$DEVSHELL_PROJECT_ID" \
	--file new-role-definition.yaml
```

4. Atualizar role `viewer` via flags (adicionando permissoes):

```bash
gcloud iam roles update "$CUSTOM_ROLE_VIEWER_ID" \
	--project "$DEVSHELL_PROJECT_ID" \
	--add-permissions "storage.buckets.get,storage.buckets.list"
```

### Explicacao dos parametros

- `roles describe --format=yaml`: captura definicao completa, incluindo `etag`.
- `roles update --file`: aplica definicao YAML atualizada.
- `--add-permissions`: adiciona permissoes sem substituir toda a lista existente.

### Resultado esperado

- Role `editor` com permissoes App Engine + Storage buckets.
- Role `viewer` com permissoes de Compute e Storage list/get.

---

## Tarefa 7. Desabilitar custom role

### Conceito

Desabilitar uma role inativa seus bindings na pratica, sem apagar imediatamente o recurso.

### Passos com comandos CLI

```bash
gcloud iam roles update "$CUSTOM_ROLE_VIEWER_ID" \
	--project "$DEVSHELL_PROJECT_ID" \
	--stage "DISABLED"
```

### Explicacao dos parametros

- `--stage DISABLED`: deixa a role indisponivel para uso efetivo.

### Resultado esperado

Role `viewer` permanece existente, porem com `stage: DISABLED`.

---

## Tarefa 8. Excluir custom role

### Conceito

Excluir a role a torna inativa para novos bindings. Existe janela de restauracao antes da remocao definitiva.

### Passos com comandos CLI

```bash
gcloud iam roles delete "$CUSTOM_ROLE_VIEWER_ID" \
	--project "$DEVSHELL_PROJECT_ID"
```

### Explicacao dos parametros

- `roles delete`: marca role como excluida.
- A role pode ser restaurada em ate 7 dias; depois segue para exclusao definitiva (processo adicional).

### Exemplo: quando a role sera descontinuada (sem excluir imediatamente)

Se a role ainda estiver em uso e voce quiser fazer phase-out com orientacao para migracao, prefira marcar como `DEPRECATED` e informar uma mensagem de deprecacao:

```bash
gcloud iam roles update "$CUSTOM_ROLE_VIEWER_ID" \
	--project "$DEVSHELL_PROJECT_ID" \
	--stage "DEPRECATED" \
	--deprecation-message "Role sera removida em breve. Use projects/$DEVSHELL_PROJECT_ID/roles/editor ou consulte o runbook interno em go/iam-custom-roles"
```

Com isso, a role continua existente por um periodo de transicao, e a mensagem ajuda os consumidores a migrarem para a alternativa recomendada.

### Resultado esperado

Resposta com `deleted: true` para a role `viewer`.

---

## Tarefa 9. Restaurar custom role

### Conceito

Recuperar uma role excluida dentro da janela de restauracao.

### Passos com comandos CLI

```bash
gcloud iam roles undelete "$CUSTOM_ROLE_VIEWER_ID" \
	--project "$DEVSHELL_PROJECT_ID"
```

### Explicacao dos parametros

- `roles undelete`: restaura role deletada dentro da janela permitida.

### Resultado esperado

Role `viewer` restaurada (geralmente em estado `DISABLED`; ajuste o `stage` depois, se necessario).

---

## Validacao

Use os comandos abaixo para validar o estado final:

```bash
# Conferir roles customizadas do projeto
gcloud iam roles list --project "$DEVSHELL_PROJECT_ID" --show-deleted

# Descrever role editor
gcloud iam roles describe "$CUSTOM_ROLE_EDITOR_ID" --project "$DEVSHELL_PROJECT_ID"

# Descrever role viewer
gcloud iam roles describe "$CUSTOM_ROLE_VIEWER_ID" --project "$DEVSHELL_PROJECT_ID"
```

Checklist esperado:

- roles `editor` e `viewer` existem no projeto;
- `editor` contem as permissoes adicionais de Storage (apos update);
- `viewer` pode aparecer como `DISABLED` dependendo da etapa em que voce parou.

---

## Solucao de problemas

- Erro `PERMISSION_DENIED` ao criar role:
	verifique se a conta possui `iam.roles.create` (ex.: `roles/iam.roleAdmin` ou owner no projeto).

- Erro de concorrencia/etag no update:
	rode `describe` novamente, regenere o YAML com etag atual e reaplique o `roles update`.

- Role nao aparece apos criar:
	aguarde alguns segundos e execute novamente `gcloud iam roles list --project "$DEVSHELL_PROJECT_ID"`.

- Falha por projeto incorreto:
	valide `gcloud config get-value project` e compare com `PROJECT_ID`.

---

## Limpeza (opcional)

Se quiser encerrar o ambiente sem manter roles customizadas:

```bash
gcloud iam roles delete "$CUSTOM_ROLE_EDITOR_ID" --project "$DEVSHELL_PROJECT_ID"
gcloud iam roles delete "$CUSTOM_ROLE_VIEWER_ID" --project "$DEVSHELL_PROJECT_ID"
```

Observacao: se alguma role ja estiver deletada, use `gcloud iam roles list --project "$DEVSHELL_PROJECT_ID" --show-deleted` para conferir estado.

---

## Conceitos-chave

- Custom roles podem ser criadas em nivel de organizacao ou projeto (nao em folder).
- Principio de menor privilegio: conceder apenas permissoes necessarias.
- `etag` protege contra sobrescrita concorrente durante atualizacoes.
- `DISABLED` inativa uso da role sem apagar imediatamente.
- `delete` e `undelete` respeitam janela temporal de restauracao.

---

## Fluxo Final

1. Configurar regiao/variaveis e validar contexto.
2. Descobrir permissoes testaveis, metadados e roles concediveis.
3. Criar roles customizadas por YAML e por flags.
4. Listar e atualizar roles (YAML com etag + flags).
5. Desabilitar, excluir e restaurar role customizada.
6. Validar estado final com `list` e `describe`.

---

## Como clientes usam custom roles na pratica

### Padrao comum em ambientes corporativos Google Cloud

Times maduros normalmente seguem este modelo:

- usar roles predefinidas sempre que possivel;
- criar custom roles apenas para lacunas reais de permissao;
- separar roles por persona (humano, CI/CD, operacao, suporte);
- manter escopo minimo (projeto/pasta/organizacao conforme necessidade);
- revisar periodicamente permissoes nao usadas.

### Padrao para times de desenvolvimento

Estrutura comum por equipe:

- `dev-readonly`: leitura para diagnostico e troubleshooting;
- `dev-deployer`: deploy sem poderes administrativos globais;
- `dev-support-l2`: operacoes controladas (restart, list, get, logs);
- `ci-build-deployer`: service account de pipeline com permissoes minimas de build/deploy.

### Convencao de nomes recomendada

Uma convencao simples e rastreavel:

- `team_<squad>_<escopo>_<nivel>`

Exemplos:

- `team_payments_gke_deployer`
- `team_data_bq_job_runner`
- `team_media_storage_readonly`

### Exemplos de uso (role x permissoes)

1. Leitura de objetos Cloud Storage para time de suporte:

```yaml
title: "Team Media Storage ReadOnly"
description: "Leitura de objetos para suporte de conteudo"
stage: "GA"
includedPermissions:
- storage.objects.get
- storage.objects.list
```

2. Execucao de jobs no BigQuery para analytics:

```yaml
title: "Team Data BQ Job Runner"
description: "Executa jobs e le metadados minimos no BigQuery"
stage: "GA"
includedPermissions:
- bigquery.jobs.create
- bigquery.jobs.get
- bigquery.datasets.get
```

3. Deploy em GKE por pipeline CI/CD (exemplo simplificado):

```yaml
title: "Team Payments GKE Deployer"
description: "Permissoes minimas para pipeline de deploy"
stage: "BETA"
includedPermissions:
- container.clusters.get
- container.deployments.create
- container.deployments.get
- container.deployments.update
```

### Pergunta comum: preciso dar acesso ate service.resource.verb?

Sim. Em IAM, permissoes sao concedidas no nivel de permissao completa, no formato `service.resource.verb`.

- Voce nao concede permissao diretamente como `service.resource` em custom role.
- Voce lista uma ou mais permissoes especificas `...verb` em `includedPermissions`.
- Se quiser algo mais amplo, use role predefinida adequada (quando fizer sentido), em vez de tentar simplificar o identificador da permissao.

### Boas praticas de governanca

- usar `GA` para roles estaveis e `ALPHA/BETA` apenas quando necessario;
- documentar owner, sistema consumidor e justificativa da role;
- usar `DEPRECATED` com `deprecation_message` antes de remover;
- auditar bindings e remover acessos obsoletos;
- automatizar criacao/atualizacao via IaC quando possivel.

### Exemplo: criar grupo e associar usuario via gcloud

Observacao: em muitos ambientes estes comandos estao disponiveis em `alpha`/`beta`, e exigem privilegios administrativos de Cloud Identity/Google Workspace.

1. Definir variaveis:

```bash
export PROJECT_ID="${DEVSHELL_PROJECT_ID:-$(gcloud config get-value project)}"
export GROUP_EMAIL="devs-plataforma@suaempresa.com"
export GROUP_NAME="Devs Plataforma"
export GROUP_DESC="Grupo IAM para time de desenvolvimento"
export USER_EMAIL="usuario@suaempresa.com"
```

2. Habilitar API necessaria:

```bash
gcloud services enable cloudidentity.googleapis.com --project "$PROJECT_ID"
```

3. Criar grupo (tente `alpha`; se nao existir, use `beta`):

```bash
gcloud alpha identity groups create \
	--group-email="$GROUP_EMAIL" \
	--display-name="$GROUP_NAME" \
	--description="$GROUP_DESC" \
	--labels="cloudidentity.googleapis.com/groups.discussion_forum="
```

4. Adicionar usuario ao grupo:

```bash
gcloud alpha identity groups memberships add \
	--group-email="$GROUP_EMAIL" \
	--member-email="$USER_EMAIL" \
	--role="MEMBER"
```

5. Validar membros do grupo:

```bash
gcloud alpha identity groups memberships list \
	--group-email="$GROUP_EMAIL"
```

6. Conceder role IAM para o grupo no projeto:

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
	--member="group:$GROUP_EMAIL" \
	--role="roles/viewer"
```

7. Validar binding IAM aplicado ao grupo:

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
	--flatten="bindings[].members" \
	--filter="bindings.members:group:$GROUP_EMAIL" \
	--format="table(bindings.role,bindings.members)"
```

Se sua instalacao nao suportar `alpha identity`, repita os mesmos comandos substituindo `alpha` por `beta`.
