# User Authentication: Identity-Aware Proxy

## Visao Geral

O Identity-Aware Proxy (IAP) permite proteger aplicacoes web no Google Cloud sem expor o servico diretamente na internet publica.

Em vez de implementar toda a autenticacao dentro da aplicacao, o IAP intercepta as requisicoes HTTP/HTTPS, autentica o usuario com Google Identity e so encaminha o trafego para quem estiver autorizado. Alem disso, pode adicionar headers com informacoes do usuario autenticado.

O IAP e util para:

- Restringir acesso a usuarios ou grupos especificos sem alterar (ou com pouca alteracao) no codigo.
- Expor aplicacoes internas para times autorizados com controle centralizado de IAM.
- Reduzir risco de acesso indevido quando comparado a endpoints publicos sem controle de identidade.

Principais beneficios:

- Seguranca: autentica e autoriza antes da aplicacao processar a requisicao.
- Governanca: acesso controlado por IAM (`roles/iap.httpsResourceAccessor` - "IAP-secured Web App User").
- Simplicidade: para varios cenarios, basta configuracao de plataforma.

## Introducao

Este guia converte o desafio para uma execucao orientada por CLI com `gcloud`, mantendo os objetivos originais do lab:

- deploy de app Python (Flask) no **Cloud Run**;
- protecao de acesso com IAP;
- leitura de identidade de usuario via headers do IAP;
- validacao criptografica via JWT assinado (`X-Goog-IAP-JWT-Assertion`).

Observacao: o lab avisa que acoes fora do escopo podem bloquear a conta. Os comandos CLI abaixo sao equivalentes aos passos de Console do lab; onde o Console e o caminho mais seguro (ex.: obter o JWT audience), isso esta indicado.

---

## Pre-requisitos e Variaveis

### Conceito
O desafio faz 3 deploys do mesmo servico Cloud Run (`user-auth-lab`), cada um a partir de uma pasta diferente do codigo. Para evitar erro de contexto, use variaveis e rode os comandos na ordem.

### Variaveis base

```bash
# Projeto e identidade
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"
export REGION="us-central1"
export STUDENT_EMAIL="$(gcloud config get-value account)"

# Servico Cloud Run exigido pelo lab
export SERVICE_NAME="user-auth-lab"

# Service agent do IAP (precisa de run.invoker no servico)
export IAP_SA="service-${PROJECT_NUMBER}@gcp-sa-iap.iam.gserviceaccount.com"

# Codigo do lab
export LAB_BUCKET="gs://${PROJECT_ID}-bucket"
export LAB_ZIP="user-authentication-with-iap.zip"
export LAB_ROOT="$HOME/user-authentication-with-iap"
export STEP1_DIR="$LAB_ROOT/1-HelloWorld"
export STEP2_DIR="$LAB_ROOT/2-HelloUser"
export STEP3_DIR="$LAB_ROOT/3-HelloVerifiedUser"
```

### Passo 1: Definir projeto ativo e validar contexto

```bash
gcloud config set project "$PROJECT_ID"
echo "Projeto: $PROJECT_ID ($PROJECT_NUMBER) | Usuario: $STUDENT_EMAIL"
```

### Passo 2: Habilitar APIs necessarias

Evita os prompts de "enable API" durante o `gcloud run deploy`. A API do IAP ja vem pre-habilitada no lab.

```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  iap.googleapis.com
```

### Passo 3: Baixar o codigo do lab

```bash
cd "$HOME"
gcloud storage cp "${LAB_BUCKET}/${LAB_ZIP}" .
unzip -o "$LAB_ZIP"
cd "$LAB_ROOT"
ls
```

Resultado esperado: pastas `1-HelloWorld`, `2-HelloUser` e `3-HelloVerifiedUser`.

---

## TAREFA 1: Deploy da aplicacao e protecao com IAP

### Conceito
Primeiro fazemos deploy da versao basica (Hello World) com acesso publico. Depois habilitamos IAP no servico Cloud Run e concedemos acesso apenas ao usuario do lab.

### Passo 1: Revisar o codigo

```bash
cd "$STEP1_DIR"
cat main.py
cat requirements.txt
```

- `main.py`: app Flask que renderiza `templates/index.html`.
- `templates/privacy.html`: exemplo de politica de privacidade.
- `requirements.txt`: dependencias Python.

Nenhum arquivo precisa ser alterado nesta etapa.

### Passo 2: Deploy no Cloud Run

```bash
gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --allow-unauthenticated \
  --region="$REGION"
```

Se perguntar para criar o repositorio do Artifact Registry, digite `Y` e `ENTER`.

### Passo 3: Obter a URL do servico

```bash
export APP_URL="$(gcloud run services describe "$SERVICE_NAME" \
  --region="$REGION" \
  --format='value(status.url)')"
echo "$APP_URL"

curl -s "$APP_URL" | head -n 20
```

### Explicacao dos parametros principais

- `--source .`: builda a imagem a partir do codigo local via Cloud Build (buildpacks).
- `--allow-unauthenticated`: concede `roles/run.invoker` a `allUsers` (acesso publico inicial).
- `--region`: regiao do servico Cloud Run.
- `status.url`: URL publica `https://user-auth-lab-<hash>.run.app`.

> Click **Check my progress**: *Deploy a Cloud Run service*.

### Passo 4: Habilitar IAP no servico

Console (passo do lab): **Security > Identity-Aware Proxy** > toggle IAP em `user-auth-lab` > **Turn On**.

Equivalente CLI:

```bash
gcloud beta run services update "$SERVICE_NAME" \
  --region="$REGION" \
  --iap
```

Garantir que o service agent do IAP existe e pode invocar o servico:

```bash
gcloud beta services identity create --service=iap.googleapis.com --project="$PROJECT_ID"

gcloud run services add-iam-policy-binding "$SERVICE_NAME" \
  --region="$REGION" \
  --member="serviceAccount:${IAP_SA}" \
  --role="roles/run.invoker"
```

Teste: abra `$APP_URL` no navegador, faca login com o usuario do lab. Deve aparecer a tela **"You don't have access"** (IAP ativo, mas nenhum usuario autorizado ainda).

### Passo 5: Autorizar o usuario do lab no IAP

Console (passo do lab): marque `user-auth-lab` > **Add Principal** > email do aluno > role **Cloud IAP > IAP-secured Web App User** > **Save**.

Equivalente CLI:

```bash
gcloud iap web add-iam-policy-binding \
  --resource-type=cloud-run \
  --service="$SERVICE_NAME" \
  --region="$REGION" \
  --member="user:${STUDENT_EMAIL}" \
  --role="roles/iap.httpsResourceAccessor"
```

### Explicacao dos parametros principais

- `run services update --iap`: liga o IAP diretamente no servico Cloud Run (sem Load Balancer).
- `services identity create`: cria o service agent `service-PROJECT_NUMBER@gcp-sa-iap...` caso ainda nao exista.
- `roles/run.invoker` para o IAP SA: o IAP precisa invocar o servico em nome do usuario.
- `iap web add-iam-policy-binding --resource-type=cloud-run`: aplica a policy de acesso no recurso protegido pelo IAP.
- `roles/iap.httpsResourceAccessor`: "IAP-secured Web App User".

### Resultado esperado

Recarregue a pagina da app: o "Hello World" aparece. A mudanca de role pode levar ~1 minuto.

Se continuar com acesso negado, limpe o cookie de login do IAP:

```bash
echo "${APP_URL}/_gcp_iap/clear_login_cookie"
```

Abra essa URL, clique em **Use another account** e entre novamente com as credenciais do aluno.

> Click **Check my progress**: *Enable and add policy to IAP*.

---

## TAREFA 2: Acessar informacoes de identidade do usuario

### Conceito
Com IAP ativo, a aplicacao pode ler headers injetados pelo proxy:

- `X-Goog-Authenticated-User-Email`
- `X-Goog-Authenticated-User-ID`

A versao 2 do app exibe esses valores na pagina (prefixados com `accounts.google.com:`).

### Passo 1: Deploy da versao 2

```bash
cd "$STEP2_DIR"

gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --region="$REGION"
```

### Passo 2: Examinar os arquivos alterados

```bash
cat main.py
cat templates/index.html
```

Trechos relevantes:

```python
user_email = request.headers.get('X-Goog-Authenticated-User-Email')
user_id = request.headers.get('X-Goog-Authenticated-User-ID')

page = render_template('index.html', email=user_email, id=user_id)
```

```html
Hello, {{ email }}! Your persistent ID is {{ id }}.
```

### Passo 3: Testar

```bash
echo "$APP_URL"
```

Abra a URL no navegador (recarregue se necessario): devem aparecer seu email e o persistent ID.

### Passo 4: Desligar IAP (simular bypass)

Console (passo do lab): **Security > Identity-Aware Proxy** > toggle IAP de `user-auth-lab` para off > **Allow public access**.

Equivalente CLI:

```bash
gcloud beta run services update "$SERVICE_NAME" \
  --region="$REGION" \
  --no-iap

# "Allow public access": garante allUsers como invoker
gcloud run services add-iam-policy-binding "$SERVICE_NAME" \
  --region="$REGION" \
  --member="allUsers" \
  --role="roles/run.invoker"
```

Recarregue a pagina: ela aparece, mas sem informacao de usuario.

### Passo 5: Spoof de header

```bash
curl -X GET "$APP_URL" \
  -H "X-Goog-Authenticated-User-Email: totally fake email"
```

### Explicacao dos parametros principais

- `--no-iap`: remove a protecao do proxy para demonstrar o risco.
- `allUsers` + `run.invoker`: torna o servico publico novamente.
- `curl -H`: injeta header manual; sem IAP, a app nao tem como saber que o dado e falso.

### Resultado esperado

O HTML retornado mostra `Hello, totally fake email! Your persistent ID is None.`

> Click **Check my progress**: *Access User Identity Information*.

---

## TAREFA 3: Usar validacao criptografica (JWT do IAP)

### Conceito
A versao 3 usa o header `X-Goog-IAP-JWT-Assertion` e valida a assinatura digital com as chaves publicas do Google, conferindo tambem o **audience** (para qual recurso o token foi emitido). Assim, dados de identidade so sao aceitos se vieram do IAP sem alteracao.

### Passo 1: Obter o JWT audience (Client ID)

Console (recomendado, e o que o lab pede): **Security > Identity-Aware Proxy** > linha `user-auth-lab` > role a tabela ate a coluna **Actions** > **⋮** > **Get JWT audience code** > copie o **Client ID** > **OK**.

Guarde o valor em variavel:

```bash
export IAP_AUDIENCE="/projects/1067617003678/locations/us-central1/services/user-auth-lab"
echo "$IAP_AUDIENCE"
```

### Passo 2: Deploy da versao 3 com `IAP_AUDIENCE`

```bash
cd "$STEP3_DIR"

gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --set-env-vars IAP_AUDIENCE="${IAP_AUDIENCE}" \
  --region="$REGION"
```

> Click **Check my progress**: *Use Cryptographic Verification*.

### Passo 3: Examinar os arquivos

```bash
cat auth.py
cat main.py
```

Funcao principal em `auth.py`:

```python
def user():
    assertion = request.headers.get('X-Goog-IAP-JWT-Assertion')
    if assertion is None:
        return None, None

    info = jwt.decode(
        assertion,
        keys(),
        algorithms=['ES256'],
        audience=audience()
    )

    return info['email'], info['sub']
```

- `keys()`: busca as chaves publicas do Google usadas para assinar o JWT.
- `audience()`: le `IAP_AUDIENCE` do ambiente.
- `sub`: ID unico e persistente do usuario.

### Passo 4: Testar sem IAP

```bash
echo "$APP_URL"
```

Como o IAP esta desligado desde a Tarefa 2, a pagina mostra `None` para email e ID verificados.

### Passo 5: Religar IAP e autorizar o service agent

Console (passo do lab): toggle IAP de `user-auth-lab` para on > **Turn On**.

Equivalente CLI:

```bash
gcloud beta run services update "$SERVICE_NAME" \
  --region="$REGION" \
  --iap
```

Conceder ao service agent do IAP permissao para invocar o servico (comando do lab):

```bash
gcloud run services add-iam-policy-binding "$SERVICE_NAME" \
  --region="$REGION" \
  --member="serviceAccount:${IAP_SA}" \
  --role="roles/run.invoker"
```

### Resultado esperado

- Recarregue a pagina: email e ID **verificados** aparecem corretamente.
- O email verificado nao tem o prefixo `accounts.google.com:`.
- Com IAP desligado ou com bypass, os dados verificados ficam ausentes ou invalidos, pois a assinatura so pode ser gerada pelo Google.

---

## Validacao

### Validar servico Cloud Run e URL

```bash
gcloud run services describe "$SERVICE_NAME" \
  --region="$REGION" \
  --format="value(status.url, metadata.annotations.'run.googleapis.com/iap-enabled')"
```

### Validar variavel IAP_AUDIENCE no servico

```bash
gcloud run services describe "$SERVICE_NAME" \
  --region="$REGION" \
  --format="yaml(spec.template.spec.containers[0].env)"
```

### Validar invokers do Cloud Run

```bash
gcloud run services get-iam-policy "$SERVICE_NAME" --region="$REGION"
```

Esperado: `serviceAccount:${IAP_SA}` com `roles/run.invoker`.

### Validar politica IAM do IAP

```bash
gcloud iap web get-iam-policy \
  --resource-type=cloud-run \
  --service="$SERVICE_NAME" \
  --region="$REGION"
```

Esperado: `user:${STUDENT_EMAIL}` com `roles/iap.httpsResourceAccessor`.

---

## Troubleshooting

### `gcloud run deploy` pede para habilitar API / criar repositorio

Digite `Y`. Para evitar os prompts, rode o Passo 2 de pre-requisitos (enable das APIs) antes do primeiro deploy.

### Comandos `--iap` ou `--resource-type=cloud-run` nao reconhecidos

Versao do `gcloud` sem suporte a IAP direto no Cloud Run. Atualize os componentes ou use o Console (**Security > Identity-Aware Proxy**) para ligar/desligar o IAP e adicionar o principal.

```bash
gcloud components update   # fora do Cloud Shell
gcloud version
```

### Usuario segue sem acesso

Possiveis causas:

- binding IAM ainda propagando (aguarde 1-2 minutos);
- login em conta diferente da autorizada;
- cookie de sessao antigo (use `${APP_URL}/_gcp_iap/clear_login_cookie`).

### Erro 403 / "Forbidden" com IAP ligado

O service agent do IAP nao tem `roles/run.invoker`. Rode novamente o binding de `${IAP_SA}` (Tarefa 3, Passo 5).

### Pagina mostra `None` mesmo com IAP ligado (Tarefa 3)

- `IAP_AUDIENCE` vazio ou incorreto: confira com o comando de validacao e refaca o deploy com o Client ID do Console.
- IAP religado ha pouco tempo: aguarde e recarregue.

---

## Limpeza (opcional)

```bash
gcloud run services delete "$SERVICE_NAME" --region="$REGION" --quiet
```

---

## Conceitos-chave

- IAP: controle de acesso a aplicacoes web baseado em identidade, aplicado antes da requisicao chegar na app.
- IAP no Cloud Run: habilitado direto no servico; o service agent do IAP precisa de `roles/run.invoker`.
- IAM no IAP: `roles/iap.httpsResourceAccessor` ("IAP-secured Web App User") libera o acesso do usuario.
- Headers de identidade (`X-Goog-Authenticated-User-*`): uteis, mas so confiaveis quando o IAP nao pode ser contornado.
- JWT assertion do IAP (`X-Goog-IAP-JWT-Assertion`): validacao criptografica com chaves publicas do Google + audience.
- Seguranca em profundidade: autenticacao, autorizacao e verificacao criptografica.

---

## Fluxo Final

1. Definir variaveis, habilitar APIs e baixar o codigo de `gs://${PROJECT_ID}-bucket`.
2. Deploy de `1-HelloWorld` no Cloud Run (`user-auth-lab`, publico).
3. Ligar IAP, dar `run.invoker` ao service agent do IAP e conceder `IAP-secured Web App User` ao aluno.
4. Deploy de `2-HelloUser` e validar email/ID via headers.
5. Desligar IAP (acesso publico) e simular spoof com `curl`.
6. Obter o JWT audience (Client ID) e fazer deploy de `3-HelloVerifiedUser` com `IAP_AUDIENCE`.
7. Religar IAP, garantir `run.invoker` para o IAP SA e validar identidade verificada.
