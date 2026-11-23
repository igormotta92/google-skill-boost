# Develop Serverless Applications on Cloud Run: Challenge Lab

## Visao Geral

Em um challenge lab voce recebe um cenario e um conjunto de tarefas, sem passo a passo. Voce usa o que aprendeu no curso para resolver cada tarefa, e um sistema de avaliacao automatica indica se cada uma foi concluida corretamente. Para fazer 100%, todas as tarefas precisam ser concluidas dentro do tempo.

Cenario: a clinica veterinaria Pet Theory quer modernizar seus sistemas com arquitetura serverless. A aplicacao monolitica de **Billing** sera transformada em servicos no Cloud Run. O time de desenvolvimento fornece o codigo de cada componente.

Definicao de pronto do MVP:

1. Deploy da arquitetura de **Staging**.
2. Deploy da arquitetura de **Prod**.
3. Acesso **seguro** entre os componentes de Prod.

## Introducao

Este guia converte o challenge lab (GSP328) para execucao via CLI com `gcloud` e `curl`. Como e um challenge lab, a sequencia de comandos foi inferida a partir dos criterios de cada tarefa.

Pontos de atencao:
- os **nomes dos servicos e das service accounts** mudam a cada lab e aparecem nas tabelas do enunciado (ex.: "Public billing service"); preencha as variaveis com eles;
- as imagens vao para o `gcr.io` do projeto, com os nomes e tags exigidos (`billing-staging-api:0.1`, `frontend-staging:0.1`, `billing-staging-api:0.2`, `billing-prod-api:0.1`, `frontend-prod:0.1`);
- as service accounts de Billing e de Frontend sao **pre-criadas**: apenas atualize o display name e as permissoes.

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao.

### Passo 1: Projeto, regiao e codigo

```bash
gcloud config set project \
  $(gcloud projects list --format='value(PROJECT_ID)' --filter='qwiklabs-gcp')
export PROJECT_ID=$(gcloud config get-value project)

export REGION="<REGION_DO_LAB>"   # ajuste conforme o lab
gcloud config set run/region "$REGION"

cd "$HOME"
gcloud storage cp -r "gs://${PROJECT_ID}-assets/pet-theory" . && cd pet-theory/lab07
export LAB_DIR="$HOME/pet-theory/lab07"
ls "$LAB_DIR"
```

### Passo 2: Nomes do lab (tabelas do enunciado)

```bash
# Servicos Cloud Run (ajuste conforme o lab)
export PUBLIC_BILLING_SERVICE="<PUBLIC_BILLING_SERVICE_DO_LAB>"     # "Public billing service"
export FRONTEND_STAGING_SERVICE="<FRONTEND_STAGING_SERVICE_DO_LAB>" # "Frontend staging service"
export PRIVATE_BILLING_SERVICE="<PRIVATE_BILLING_SERVICE_DO_LAB>"   # "Private billing service"
export BILLING_PROD_SERVICE="<BILLING_PROD_SERVICE_DO_LAB>"         # "Billing production service"
export FRONTEND_PROD_SERVICE="<FRONTEND_PROD_SERVICE_DO_LAB>"       # "Frontend production service"

# Service accounts pre-criadas (ajuste conforme o lab)
export BILLING_SA="<BILLING_SA_DO_LAB>"     # "Billing service account"
export FRONTEND_SA="<FRONTEND_SA_DO_LAB>"   # "Frontend service account"
export BILLING_SA_EMAIL="${BILLING_SA}@${PROJECT_ID}.iam.gserviceaccount.com"
export FRONTEND_SA_EMAIL="${FRONTEND_SA}@${PROJECT_ID}.iam.gserviceaccount.com"

# Imagens (gcr.io)
export IMG_BILLING_STAGING_01="gcr.io/${PROJECT_ID}/billing-staging-api:0.1"
export IMG_FRONTEND_STAGING="gcr.io/${PROJECT_ID}/frontend-staging:0.1"
export IMG_BILLING_STAGING_02="gcr.io/${PROJECT_ID}/billing-staging-api:0.2"
export IMG_BILLING_PROD="gcr.io/${PROJECT_ID}/billing-prod-api:0.1"
export IMG_FRONTEND_PROD="gcr.io/${PROJECT_ID}/frontend-prod:0.1"
```

### Passo 3: Conferir as service accounts pre-criadas

```bash
gcloud iam service-accounts list --format="table(email,displayName)"
```

### Explicacao dos parametros
- `--filter='qwiklabs-gcp'`: seleciona o projeto do lab.
- `run/region`: regiao padrao dos comandos `gcloud run`.
- Variaveis `*_SERVICE` / `*_SA`: nomes exatos das tabelas do lab (o check procura por eles).

---

## TAREFA 1: Habilitar um servico publico (Billing staging 0.1)

### Conceito
API REST de billing publica: imagem gerada pelo Cloud Build a partir de `unit-api-billing` e servico Cloud Run sem autenticacao.

### Passo 1: Build e deploy

```bash
cd "$LAB_DIR/unit-api-billing"
gcloud builds submit --tag "$IMG_BILLING_STAGING_01"

gcloud run deploy "$PUBLIC_BILLING_SERVICE" \
  --image "$IMG_BILLING_STAGING_01" \
  --region "$REGION" \
  --allow-unauthenticated
```

### Passo 2: Testar

```bash
export PUBLIC_BILLING_URL=$(gcloud run services describe "$PUBLIC_BILLING_SERVICE" --region "$REGION" --format "value(status.url)")
curl -s -w "\n[HTTP %{http_code}]\n" "$PUBLIC_BILLING_URL"
```

### Resultado esperado
- Servico respondendo `200` sem token.

> Click **Check my progress**: *Deploy a Public Billing Service*.

Se o check nao registrar, aguarde 30 segundos e tente de novo.

---

## TAREFA 2: Fazer deploy do frontend (staging)

### Conceito
Frontend de staging, publico, gerado a partir de `staging-frontend-billing`.

### Passo 1: Build e deploy

```bash
cd "$LAB_DIR/staging-frontend-billing"
gcloud builds submit --tag "$IMG_FRONTEND_STAGING"

gcloud run deploy "$FRONTEND_STAGING_SERVICE" \
  --image "$IMG_FRONTEND_STAGING" \
  --region "$REGION" \
  --allow-unauthenticated
```

### Passo 2: Testar

```bash
export FRONTEND_STAGING_URL=$(gcloud run services describe "$FRONTEND_STAGING_SERVICE" --region "$REGION" --format "value(status.url)")
curl -s -o /dev/null -w "[HTTP %{http_code}]\n" "$FRONTEND_STAGING_URL"
```

### Resultado esperado
- Frontend respondendo `200`.

> Click **Check my progress**: *Deploy the Frontend Service*.

### Resumo visual

```text
Staging (publico):

[Internet] --> [FRONTEND_STAGING_SERVICE] (gcr.io/.../frontend-staging:0.1)

[Internet] --> [PUBLIC_BILLING_SERVICE]   (gcr.io/.../billing-staging-api:0.1)
```

---

## TAREFA 3: Fazer deploy de um servico privado (Billing staging 0.2)

### Conceito
A nova versao do billing de staging passa a exigir autenticacao. O servico publico anterior e removido e o novo e chamado com token de identidade.

### Passo 1: Remover o servico publico

```bash
gcloud run services delete "$PUBLIC_BILLING_SERVICE" --region "$REGION" --quiet
```

### Passo 2: Build e deploy privado

```bash
cd "$LAB_DIR/staging-api-billing"
gcloud builds submit --tag "$IMG_BILLING_STAGING_02"

gcloud run deploy "$PRIVATE_BILLING_SERVICE" \
  --image "$IMG_BILLING_STAGING_02" \
  --region "$REGION" \
  --no-allow-unauthenticated
```

### Passo 3: Capturar a URL e testar

```bash
export BILLING_URL=$(gcloud run services describe "$PRIVATE_BILLING_SERVICE" \
  --region "$REGION" \
  --format "value(status.url)")
echo "$BILLING_URL"

# Sem token: 403
curl -s -o /dev/null -w "[HTTP %{http_code}]\n" "$BILLING_URL"

# Com token: 200
curl -X GET -H "Authorization: Bearer $(gcloud auth print-identity-token)" "$BILLING_URL"
```

### Explicacao dos parametros
- `services delete`: remove o billing publico, como pede o lab.
- `--no-allow-unauthenticated`: exige `run.invoker` para chamar.
- `print-identity-token`: token OIDC do usuario (dono do projeto ja tem `run.invoker`).

### Resultado esperado
- `403` sem token e resposta valida com token.

> Click **Check my progress**: *Deploy a Private Billing Service*.

### Resumo visual

```text
Antes:  [Internet] --> [PUBLIC_BILLING_SERVICE] :0.1 (publico)

        delete + build :0.2 + deploy --no-allow-unauthenticated

Depois: [Internet] --x--> [PRIVATE_BILLING_SERVICE] :0.2 (403 sem token)
        [Usuario + identity token] --> [PRIVATE_BILLING_SERVICE] (200)
```

---

## TAREFA 4: Configurar a service account do Billing

### Conceito
A SA de billing (pre-criada) sera a identidade do servico de Prod. O lab exige apenas atualizar o display name. Sem papeis adicionais (least privilege).

### Passo 1: Atualizar o display name

```bash
gcloud iam service-accounts update "$BILLING_SA_EMAIL" \
  --display-name="Billing Service Cloud Run"

gcloud iam service-accounts describe "$BILLING_SA_EMAIL" --format="value(email,displayName)"
```

### Resultado esperado
- `displayName: Billing Service Cloud Run`.

> Click **Check my progress**: *Create a Billing Service Account*.

---

## TAREFA 5: Fazer deploy do billing de producao

### Conceito
Billing de Prod: privado e executando com a SA de billing como identidade do servico.

### Passo 1: Build e deploy com a SA

```bash
cd "$LAB_DIR/prod-api-billing"
gcloud builds submit --tag "$IMG_BILLING_PROD"

gcloud run deploy "$BILLING_PROD_SERVICE" \
  --image "$IMG_BILLING_PROD" \
  --region "$REGION" \
  --no-allow-unauthenticated \
  --service-account "$BILLING_SA_EMAIL"
```

### Passo 2: Capturar a URL e testar

```bash
export PROD_BILLING_URL=$(gcloud run services describe "$BILLING_PROD_SERVICE" \
  --region "$REGION" \
  --format "value(status.url)")
echo "$PROD_BILLING_URL"

curl -X GET -H "Authorization: Bearer $(gcloud auth print-identity-token)" "$PROD_BILLING_URL"
```

### Explicacao dos parametros
- `--service-account`: identidade com a qual o container roda (chamadas a APIs do Google).
- `--no-allow-unauthenticated`: so chamadores com `run.invoker`.

### Resultado esperado
- Servico de Prod respondendo com token e usando a SA de billing.

> Click **Check my progress**: *Deploy the Billing Service*.

### Resumo visual

```text
[BILLING_SA] display name "Billing Service Cloud Run"
        |
        | --service-account (identidade do servico)
        v
[BILLING_PROD_SERVICE] (gcr.io/.../billing-prod-api:0.1, privado)
        ^
        | identity token (usuario)
```

---

## TAREFA 6: Service account do Frontend

### Conceito
A SA do frontend (pre-criada) precisa invocar o billing privado de Prod: recebe `roles/run.invoker` **no servico** de billing, e nao no projeto inteiro.

### Passo 1: Atualizar o display name

```bash
gcloud iam service-accounts update "$FRONTEND_SA_EMAIL" \
  --display-name="Billing Service Cloud Run Invoker"
```

### Passo 2: Conceder `run.invoker` no billing de Prod

```bash
gcloud run services add-iam-policy-binding "$BILLING_PROD_SERVICE" \
  --region "$REGION" \
  --member="serviceAccount:${FRONTEND_SA_EMAIL}" \
  --role=roles/run.invoker

gcloud run services get-iam-policy "$BILLING_PROD_SERVICE" --region "$REGION"
```

### Explicacao dos parametros
- `service-accounts update --display-name`: ajuste exigido pelo lab na SA pre-criada.
- `run services add-iam-policy-binding`: permissao restrita a um unico servico (least privilege).

### Resultado esperado
- SA do frontend com `roles/run.invoker` no `BILLING_PROD_SERVICE`.

> Click **Check my progress**: *Create the Frontend Service Account*.

---

## TAREFA 7: Refazer o deploy do frontend (producao)

### Conceito
Frontend de Prod publico, executando com a SA do frontend. Assim ele consegue chamar o billing privado com a propria identidade.

### Passo 1: Conferir o codigo do frontend de Prod

```bash
cd "$LAB_DIR/prod-frontend-billing"
ls
grep -rIl -i "billing" --exclude-dir=node_modules . | head
```

Se o codigo depender de uma URL do billing configurada em arquivo ou variavel, ajuste para `PROD_BILLING_URL` conforme o proprio codigo indicar.

### Passo 2: Build e deploy com a SA

```bash
gcloud builds submit --tag "$IMG_FRONTEND_PROD"

gcloud run deploy "$FRONTEND_PROD_SERVICE" \
  --image "$IMG_FRONTEND_PROD" \
  --region "$REGION" \
  --allow-unauthenticated \
  --service-account "$FRONTEND_SA_EMAIL"
```

### Passo 3: Testar

```bash
export FRONTEND_PROD_URL=$(gcloud run services describe "$FRONTEND_PROD_SERVICE" --region "$REGION" --format "value(status.url)")
echo "$FRONTEND_PROD_URL"
curl -s -o /dev/null -w "[HTTP %{http_code}]\n" "$FRONTEND_PROD_URL"
```

Abra `FRONTEND_PROD_URL` no navegador: a pagina **Pet Theory: Billing Service** exibe dados consumidos do billing privado via SA.

### Explicacao dos parametros
- `--allow-unauthenticated`: frontend acessivel ao publico.
- `--service-account FRONTEND_SA_EMAIL`: identidade usada pelo frontend para chamar o billing privado.

### Resultado esperado
- Frontend de Prod respondendo `200` e exibindo os dados do billing.

> Click **Check my progress**: *Deploy the Frontend Service*.

### Resumo visual

```text
Prod (acesso seguro entre componentes):

[Internet] --> [FRONTEND_PROD_SERVICE] (publico, frontend-prod:0.1)
                   identidade: FRONTEND_SA
                         |
                         | token OIDC da FRONTEND_SA (run.invoker)
                         v
               [BILLING_PROD_SERVICE] (privado, billing-prod-api:0.1)
                   identidade: BILLING_SA
```

---

## Validacao

```bash
gcloud run services list --region "$REGION" \
  --format="table(metadata.name,status.url,spec.template.spec.serviceAccountName)"

gcloud container images list --repository="gcr.io/${PROJECT_ID}"

gcloud iam service-accounts list --format="table(email,displayName)"

gcloud run services get-iam-policy "$BILLING_PROD_SERVICE" --region "$REGION" \
  --format="table(bindings.role,bindings.members)"
```

Checklist esperado:
- staging: `FRONTEND_STAGING_SERVICE` publico e `PRIVATE_BILLING_SERVICE` privado (o publico foi removido);
- prod: `BILLING_PROD_SERVICE` privado com `BILLING_SA`; `FRONTEND_PROD_SERVICE` publico com `FRONTEND_SA`;
- display names `Billing Service Cloud Run` e `Billing Service Cloud Run Invoker`;
- `FRONTEND_SA` com `run.invoker` no billing de Prod;
- imagens `billing-staging-api` (0.1, 0.2), `frontend-staging`, `billing-prod-api` e `frontend-prod` no `gcr.io`.

---

## Resumo Visual da Arquitetura

Compilado: estado final ao termino do lab. Os detalhes de cada etapa estao no `### Resumo visual` das Tarefas 2, 3, 5 e 7.

```text
STAGING
  [Internet] --> [FRONTEND_STAGING_SERVICE] (publico)
  [Usuario + token] --> [PRIVATE_BILLING_SERVICE] (privado, :0.2)

PROD
  [Internet] --> [FRONTEND_PROD_SERVICE] (publico, SA: FRONTEND_SA)
                          |
                          | run.invoker (token OIDC)
                          v
                 [BILLING_PROD_SERVICE] (privado, SA: BILLING_SA)

Imagens: gcr.io/PROJECT_ID/{billing-staging-api, frontend-staging,
         billing-prod-api, frontend-prod}
```

---

## Troubleshooting

- Variaveis com `<..._DO_LAB>`:
  - preencha todas com os nomes das tabelas do enunciado antes de rodar.
- Check nao registra logo apos o deploy:
  - o activity tracking pode demorar; aguarde 30 segundos e clique novamente.
- `gcloud builds submit` falha por falta de `Dockerfile`:
  - confira se esta dentro da pasta de codigo correta (`unit-api-billing`, `staging-frontend-billing`, etc.).
- Push para `gcr.io` falha por repositorio inexistente:
  ```bash
  gcloud artifacts repositories create gcr.io --repository-format=docker --location=us
  ```
- `service-accounts update` com `NOT_FOUND`:
  - confira o nome exato da SA pre-criada com `gcloud iam service-accounts list`.
- Frontend de Prod sem dados do billing:
  - confira o binding `run.invoker` da `FRONTEND_SA` no `BILLING_PROD_SERVICE` e se o frontend foi publicado com `--service-account`.
- `403` no curl autenticado:
  - gere o token de novo (`gcloud auth print-identity-token`) e confira a conta ativa.

---

## Limpeza (Opcional)

Nao remova as service accounts: elas sao pre-criadas pelo lab.

```bash
gcloud run services delete "$FRONTEND_STAGING_SERVICE" "$PRIVATE_BILLING_SERVICE" \
  "$BILLING_PROD_SERVICE" "$FRONTEND_PROD_SERVICE" --region "$REGION" --quiet
```

---

## Conceitos-Chave

- Staging x Prod: mesma aplicacao em ambientes separados, com nivel de seguranca diferente.
- Servico publico x privado: `--allow-unauthenticated` x `--no-allow-unauthenticated`.
- Identidade do servico: `--service-account` define com qual SA o container roda.
- Service-to-service: o chamador (frontend) precisa de `run.invoker` no servico chamado (billing).
- Least privilege: permissao no nivel do servico, nao do projeto.

---

## Fluxo Final

1. Definir projeto, regiao, nomes do lab e baixar `pet-theory/lab07`.
2. Build `billing-staging-api:0.1` e deploy publico.
3. Build `frontend-staging:0.1` e deploy publico.
4. Remover o billing publico; build `billing-staging-api:0.2` e deploy privado; testar com token.
5. Atualizar o display name da SA de billing.
6. Build `billing-prod-api:0.1` e deploy privado com a SA de billing.
7. Atualizar o display name da SA do frontend e dar `run.invoker` no billing de Prod.
8. Build `frontend-prod:0.1` e deploy publico com a SA do frontend; validar a pagina de billing.
