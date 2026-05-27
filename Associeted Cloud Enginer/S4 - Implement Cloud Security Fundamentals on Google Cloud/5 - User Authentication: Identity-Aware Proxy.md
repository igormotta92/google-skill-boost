# User Authentication: Identity-Aware Proxy

## Visao Geral

O Identity-Aware Proxy (IAP) permite proteger aplicacoes web no Google Cloud sem expor o servico diretamente na internet publica.

Em vez de implementar toda a autenticacao dentro da aplicacao, o IAP intercepta as requisicoes HTTP/HTTPS, autentica o usuario com Google Identity e so encaminha o trafego para quem estiver autorizado.

O IAP e util para:

- Restringir acesso a usuarios ou grupos especificos sem alterar (ou com pouca alteracao) no codigo.
- Expor aplicacoes internas para times autorizados com controle centralizado de IAM.
- Reduzir risco de acesso indevido quando comparado a endpoints publicos sem controle de identidade.

Principais beneficios:

- Seguranca: autentica e autoriza antes da aplicacao processar a requisicao.
- Governanca: acesso controlado por IAM (`roles/iap.httpsResourceAccessor`).
- Simplicidade: para varios cenarios, basta configuracao de plataforma.

## Introducao

Este guia converte o desafio para uma execucao orientada por CLI com `gcloud`, mantendo os objetivos originais do lab:

- deploy de app Python no App Engine Standard;
- protecao de acesso com IAP;
- leitura de identidade de usuario via headers do IAP;
- validacao criptografica via JWT assinado (`X-Goog-IAP-JWT-Assertion`).

---

## Pre-requisitos e Variaveis

### Conceito
O desafio usa App Engine + IAP e alterna entre 3 versoes da aplicacao. Para evitar erro de contexto, use variaveis e rode os comandos na ordem.

### Variaveis base (ajuste conforme ambiente)

```bash
# Projeto e identidade
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"
export REGION="<REGIAO_APP_ENGINE>"   # ex: us-central
export STUDENT_EMAIL="<SEU_EMAIL_DO_LAB>"

# Pastas do codigo do lab
export LAB_ROOT="$HOME/user-authentication-with-iap"
export STEP1_DIR="$LAB_ROOT/1-HelloWorld"
export STEP2_DIR="$LAB_ROOT/2-HelloUser"
export STEP3_DIR="$LAB_ROOT/3-HelloVerifiedUser"

# IAP/OAuth (ajuste conforme ambiente/politica)
export OAUTH_BRAND_TITLE="IAP Example"
export OAUTH_CLIENT_DISPLAY_NAME="iap-client"
```

### Passo 1: Definir projeto ativo

```bash
gcloud config set project "$PROJECT_ID"
```

### Passo 2: Habilitar APIs necessarias

```bash
gcloud services enable \
  appengine.googleapis.com \
  iap.googleapis.com \
  cloudresourcemanager.googleapis.com
```

### Passo 3: Desabilitar Flex API (como pedido no lab)

```bash
gcloud services disable appengineflex.googleapis.com --force
```

---

## TAREFA 1: Deploy da aplicacao e protecao com IAP

### Conceito
Primeiro fazemos deploy da versao basica (Hello World). Depois habilitamos IAP no App Engine e concedemos acesso apenas ao usuario autorizado.

### Passos CLI - Baixar codigo do lab

```bash
cd "$HOME"
gsutil cp gs://spls/gsp499/user-authentication-with-iap.zip .
unzip -o user-authentication-with-iap.zip
cd "$LAB_ROOT"
```

### Passos CLI - Deploy da versao 1

```bash
cd "$STEP1_DIR"

# Atualiza runtime conforme enunciado
sed -i 's/python37/python313/g' app.yaml

# Crie App Engine se for o primeiro deploy no projeto
# (se ja existir, este comando pode falhar com "already exists")
gcloud app create --region="$REGION" || true

# Deploy
gcloud app deploy --quiet
```

### Explicacao dos parametros principais

- `gcloud app create --region`: define regiao do App Engine (nao muda depois).
- `gcloud app deploy`: publica a versao da aplicacao no App Engine Standard.
- `--quiet`: evita prompt interativo.

### Passos CLI - Obter URL da aplicacao

```bash
export APP_HOST="$(gcloud app describe --project="$PROJECT_ID" --format='value(defaultHostname)')"
export APP_URL="https://${APP_HOST}"

echo "$APP_URL"
gcloud app browse
```

### Passos CLI - Configurar OAuth para IAP (quando necessario)

Observacao: em alguns tenants/labs, parte do consentimento pode exigir Console. Quando permitido, use CLI:

```bash
# Criar brand OAuth interno (pode retornar erro se ja existir)
gcloud iap oauth-brands create \
  --application_title="$OAUTH_BRAND_TITLE" \
  --support_email="$STUDENT_EMAIL" || true

# Descobrir nome do brand
export BRAND_NAME="$(gcloud iap oauth-brands list --format='value(name)' | head -n1)"

# Criar cliente OAuth para IAP (pode retornar erro se ja existir)
gcloud iap oauth-clients create "$BRAND_NAME" \
  --display_name="$OAUTH_CLIENT_DISPLAY_NAME" || true

# Listar clientes
# Guarde CLIENT_ID e CLIENT_SECRET retornados na criacao ou via describe/list
```

Se o ambiente exigir cliente explicito no enable do IAP:

```bash
export CLIENT_ID="<CLIENT_ID>"
export CLIENT_SECRET="<CLIENT_SECRET>"
```

### Passos CLI - Habilitar IAP para App Engine

Tentativa direta (funciona em ambientes com configuracao gerenciada):

```bash
gcloud iap web enable --resource-type=app-engine
```

Se o comando pedir credenciais OAuth, rode:

```bash
gcloud iap web enable \
  --resource-type=app-engine \
  --oauth2-client-id="$CLIENT_ID" \
  --oauth2-client-secret="$CLIENT_SECRET"
```

### Passos CLI - Autorizar usuario no IAP

```bash
gcloud iap web add-iam-policy-binding \
  --resource-type=app-engine \
  --member="user:${STUDENT_EMAIL}" \
  --role="roles/iap.httpsResourceAccessor"
```

### Resultado esperado

- Aplicacao publicada no App Engine.
- IAP habilitado para o recurso App Engine.
- Usuario do lab autorizado com papel de acesso a app protegida por IAP.

## Test access
Navigate back to your app and reload the page. You should now see your web app, since you already logged in with a user you authorized.

If you still see the "You don't have access" page, IAP did not recheck your authorization. In that case, do the following steps:

Open your web browser to the home page address with /_gcp_iap/clear_login_cookie added to the end of the URL, as in https://iap-example-999999.appspot.com/_gcp_iap/clear_login_cookie.
You will see a new Sign in with Google screen, with your account already showing. Do not click the account. Instead, click Use another account, and re-enter your credentials.

---

## TAREFA 2: Acessar informacoes de identidade do usuario

### Conceito
Com IAP ativo, a aplicacao pode ler headers injetados pelo proxy, como:

- `X-Goog-Authenticated-User-Email`
- `X-Goog-Authenticated-User-ID`

A versao 2 do app exibe esses valores na pagina.

curl -X GET https://qwiklabs-gcp-01-36add0e40dfa.uc.r.appspot.com/ -H "X-Goog-Authenticated-User-Email: totally fake email"

### Passos CLI - Deploy da versao 2

```bash
cd "$STEP2_DIR"
sed -i 's/python37/python313/g' app.yaml
gcloud app deploy --quiet
```

### Passos CLI - Teste funcional

```bash
echo "$APP_URL"
gcloud app browse
```

Abra a URL no navegador e valide:

- com usuario autorizado: acesso permitido e exibicao de email/ID;
- com usuario nao autorizado: tela de bloqueio de acesso.

### Passos CLI - Simular app sem IAP (teste de risco)

Desabilite IAP:

```bash
gcloud iap web disable --resource-type=app-engine
```

Teste spoof de header:

```bash
curl -X GET "$APP_URL" \
  -H "X-Goog-Authenticated-User-Email: totally fake email"
```

### Explicacao dos parametros principais

- `iap web disable`: remove protecao do proxy para mostrar o risco.
- `curl -H`: injeta header manual para demonstrar que, sem IAP, a app nao consegue confiar nesse dado.

### Resultado esperado

- Com IAP desligado, pagina pode aceitar header falso.
- Fica evidente a necessidade de validacao criptografica quando houver risco de bypass.

---

## TAREFA 3: Usar validacao criptografica (JWT do IAP)

### Conceito
A versao 3 usa o header `X-Goog-IAP-JWT-Assertion` e valida assinatura digital com chaves publicas do Google para garantir autenticidade dos dados de identidade.

### Passos CLI - Deploy da versao 3

```bash
cd "$STEP3_DIR"
sed -i 's/python37/python313/g' app.yaml
gcloud app deploy --quiet
```

### Passos CLI - Reabilitar IAP

```bash
gcloud iap web enable --resource-type=app-engine
```

Se solicitado, habilite com cliente OAuth:

```bash
gcloud iap web enable \
  --resource-type=app-engine \
  --oauth2-client-id="$CLIENT_ID" \
  --oauth2-client-secret="$CLIENT_SECRET"
```

### Resultado esperado

- Com IAP ligado, app mostra identidade validada.
- Sem IAP (ou com bypass), dados validados ficam ausentes/invalidos.

---

## Validacao

### Validar status do App Engine e URL

```bash
gcloud app describe --project="$PROJECT_ID" \
  --format="table(id,locationId,defaultHostname)"
```

### Validar servicos/API

```bash
gcloud services list --enabled \
  --filter="name:appengine.googleapis.com OR name:iap.googleapis.com"
```

### Validar politica IAM do IAP

```bash
gcloud iap web get-iam-policy --resource-type=app-engine
```

Esperado: membro `user:${STUDENT_EMAIL}` com role `roles/iap.httpsResourceAccessor`.

### Validar acesso da aplicacao

- Usuario autorizado: abre app normalmente.
- Usuario nao autorizado: recebe bloqueio do IAP.
- Com IAP desligado: resposta pode refletir header falso via `curl`.

---

## Troubleshooting

### Erro de deploy no App Engine

Possiveis causas:

- app nao criado na regiao;
- APIs nao habilitadas;
- propagacao temporaria de IAM/API.

Comandos de checagem:

```bash
gcloud app describe --project="$PROJECT_ID"
gcloud services list --enabled | grep -E "appengine|iap"
```

### Erro ao habilitar IAP

Possiveis causas:

- OAuth brand/client nao criado;
- permissao insuficiente para configurar OAuth consent;
- necessidade de informar `--oauth2-client-id` e `--oauth2-client-secret`.

Comandos de checagem:

```bash
gcloud iap oauth-brands list
gcloud iap oauth-clients list "$BRAND_NAME"
```

### Usuario segue sem acesso

Possiveis causas:

- binding IAM nao aplicado ainda (delay de propagacao);
- login em conta diferente da autorizada;
- cookie de sessao antigo.

Comandos/acoes:

```bash
gcloud iap web get-iam-policy --resource-type=app-engine
```

Depois, aguarde 1-2 minutos e refaca login.

---

## Limpeza (opcional)

Se quiser remover versoes e recursos do lab:

```bash
# Opcional: remover servico default do App Engine (cuidado em projetos compartilhados)
gcloud app services delete default --quiet

# Opcional: desabilitar IAP
gcloud iap web disable --resource-type=app-engine

# Opcional: desabilitar APIs
gcloud services disable iap.googleapis.com appengine.googleapis.com --force
```

---

## Conceitos-chave

- IAP: controle de acesso a aplicacoes web baseado em identidade.
- IAM no IAP: papel principal para acesso web e `roles/iap.httpsResourceAccessor`.
- Headers de identidade: uteis, mas devem ser confiados somente quando protegidos por IAP.
- JWT assertion do IAP: permite validacao criptografica da identidade.
- Seguranca em profundidade: autenticacao, autorizacao e verificacao criptografica.

---

## Fluxo Final

1. Baixar codigo e preparar variaveis.
2. Deploy `1-HelloWorld` no App Engine.
3. Configurar OAuth (quando necessario), habilitar IAP e conceder acesso IAM.
4. Deploy `2-HelloUser` e validar headers de identidade.
5. Desligar IAP e simular spoof para entender risco.
6. Deploy `3-HelloVerifiedUser` e validar fluxo com JWT assinado.
7. Validar acesso autorizado vs nao autorizado e status final do IAP.
