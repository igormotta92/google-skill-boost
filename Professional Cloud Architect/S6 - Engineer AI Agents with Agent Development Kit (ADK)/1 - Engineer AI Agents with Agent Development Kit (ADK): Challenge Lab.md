# Engineer AI Agents with Agent Development Kit (ADK): Challenge Lab

## Visao Geral

Em um challenge lab voce recebe um cenario e um conjunto de tarefas, sem passo a passo. Voce usa o que aprendeu no curso para resolver cada tarefa, e um sistema de avaliacao automatica indica se cada uma foi concluida corretamente. Para fazer 100%, todas as tarefas precisam ser concluidas dentro do tempo.

Cenario: voce entrou na Cymbal Travel, uma agencia que usa agentes de IA para buscar novidades de viagem, validar dados de destinos e auditar brochuras de marketing. Os prototipos feitos com o Agent Development Kit (ADK) estao mal configurados, incompletos ou quebrados, e voce precisa repara-los.

O diretorio `adk_project` contem:

- `my_google_search_agent`: "Travel Scout", que deveria buscar eventos (sem a ferramenta Google Search);
- `geo_validator`: "Destination Verifier", que retorna texto livre (precisa de schema JSON estrito);
- `llm_auditor`: "Brochure Auditor", pipeline multi-agente que so tem o critic (falta o reviser).

## Introducao

Este guia converte o challenge lab (GSP540) para execucao via CLI com `pip`, `gcloud`, `adk` e `python3`. As edicoes de codigo sao feitas no Cloud Shell Editor (passo do lab), com o trecho esperado e comandos `grep` para conferir.

Regras do lab:
- **nao criar nem usar API keys**: os agentes usam o Agent Platform via `.env` (`GOOGLE_GENAI_USE_ENTERPRISE=true`);
- `GOOGLE_CLOUD_LOCATION=global` e obrigatorio;
- `MODEL` = modelo Flash informado no lab (`model_name` no enunciado).

---

## Pre-requisitos e Variaveis

### Conceito
Centralize os valores do lab em variaveis. Em Cloud Shell, refaca esse bloco a cada nova sessao/reconexao (incluindo o `PATH`).

### Passo 1: Definir variaveis

```bash
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export MODEL_NAME="<MODEL_DO_LAB>"   # "model_name" do enunciado (modelo Flash mais recente)
export LOCATION="global"

export LAB_BUCKET="gs://${PROJECT_ID}-bucket"   # o enunciado mostra gs://-bucket (prefixo = project id)
export PROJECT_DIR="$HOME/adk_project"

export SEARCH_AGENT="my_google_search_agent"
export GEO_AGENT="geo_validator"
export AUDITOR_AGENT="llm_auditor"

gcloud config set project "$PROJECT_ID"
```

### Explicacao dos parametros
- `MODEL_NAME`: valor do campo `MODEL` em todos os `.env` (ex.: um modelo `gemini-*-flash`).
- `LAB_BUCKET`: bucket com o `adk_project.zip`; confira com `gcloud storage ls` se o nome for outro.

---

## TAREFA 1: Instalar o ADK e preparar o ambiente

### Conceito
O ADK e instalado via `pip` no Cloud Shell. Os agentes autenticam no Agent Platform com Application Default Credentials (ADC) do usuario do lab.

### Passo 1: Instalar o ADK

```bash
export PATH=$PATH:"/home/${USER}/.local/bin"
python3 -m pip install google-adk
adk --version
```

### Passo 2: Autenticar com ADC (interativo)

```bash
gcloud auth application-default login
```

Responda `Y`, abra o link, escolha o **email do aluno**, clique em **Continue** e **Allow**, copie o codigo e cole no terminal em `Enter authorization code`.

```bash
gcloud auth application-default print-access-token >/dev/null && echo "ADC ok"
```

### Passo 3: Baixar o codigo do lab

```bash
cd "$HOME"
gcloud storage cp "${LAB_BUCKET}/adk_project.zip" .
unzip -o adk_project.zip
cd "$PROJECT_DIR"
pip install -r requirements.txt
ls
```

### Passo 4: Criar os `.env` dos tres agentes

O lab pede o mesmo conteudo em cada pasta de agente. Gerado com `cat` e variaveis:

```bash
cd "$PROJECT_DIR"

for agent in "$SEARCH_AGENT" "$GEO_AGENT" "$AUDITOR_AGENT"; do
cat > "${agent}/.env" <<EOF
GOOGLE_GENAI_USE_ENTERPRISE=true
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=${LOCATION}
MODEL=${MODEL_NAME}
EOF
echo "== ${agent}/.env"; cat "${agent}/.env"
done
```

### Explicacao dos parametros
- `PATH .../.local/bin`: onde o `pip --user` instala o executavel `adk`.
- `application-default login`: cria credenciais ADC usadas pelas bibliotecas do Google.
- `GOOGLE_GENAI_USE_ENTERPRISE=true`: usa o Agent Platform (projeto + localizacao) em vez de API key.
- `GOOGLE_CLOUD_LOCATION=global`: localizacao exigida pelo lab.

### Resultado esperado
- `adk` instalado, ADC valido e pastas `my_google_search_agent`, `geo_validator` e `llm_auditor` com `.env` preenchido.

---

## TAREFA 2: Inicializar e configurar o Travel Scout

### Conceito
O agente `my_google_search_agent` precisa da ferramenta built-in `google_search` para responder com informacoes atuais **ancoradas (grounding)** em resultados da busca.

### Passo 1: Abrir o codigo

Console (passo do lab): **Open Editor** > **View > Toggle hidden files** (para ver o `.env`) > `adk_project/my_google_search_agent/agent.py`.

```bash
cd "$PROJECT_DIR"
cat "${SEARCH_AGENT}/agent.py"
```

### Passo 2: Habilitar o `google_search` no `agent.py`

Garanta o import e o parametro `tools` na definicao do agente (mantendo `name`, `instruction` e demais campos existentes):

```python
from google.adk.agents import Agent
from google.adk.tools import google_search

root_agent = Agent(
    # ... campos ja existentes (name, model, description, instruction) ...
    tools=[google_search],
)
```

Se a `instruction` nao pedir uso da busca, inclua uma orientacao como: "Use the google_search tool to find current information and base your answer on the search results."

Conferir:

```bash
grep -n "google_search" "${SEARCH_AGENT}/agent.py"
```

### Passo 3: Rodar no ADK Dev UI

```bash
cd "$PROJECT_DIR"
adk web --allow_origins "regex:https://.*\.cloudshell\.dev"
```

Abra `http://127.0.0.1:8000` (link no terminal / Web Preview porta `8000`) > selecione `my_google_search_agent` > pergunte:

```text
What are some major events in Tokyo in 2025?
```

Na aba de eventos do Dev UI, confirme a chamada da ferramenta de busca. Se houver erro, use **+ New Session** e repita. Depois pare o servidor com `Ctrl+C`.

### Explicacao dos parametros
- `tools=[google_search]`: ferramenta de busca do Google com grounding.
- `adk web`: Dev UI local para conversar com os agentes da pasta atual.
- `--allow_origins "regex:https://.*\.cloudshell\.dev"`: libera o acesso via Web Preview do Cloud Shell.

### Resultado esperado
- Resposta com eventos de Tokyo baseada em resultados do Google Search.

> Click **Check my progress**: *Run the agent using the ADK's Web UI*.

### Resumo visual

```text
[ADK Dev UI :8000] -- "major events in Tokyo in 2025?"
        |
        v
[my_google_search_agent] (MODEL, Agent Platform, location global)
        |
        | tools=[google_search]  (grounding)
        v
[Google Search] --> resposta com eventos atuais
```

---

## TAREFA 3: Verificar o agente pela CLI

### Conceito
`adk run` abre um chat no terminal com o agente, sem a interface web.

### Passo 1: Conversar pela CLI

```bash
cd "$PROJECT_DIR"
adk run "$SEARCH_AGENT"
```

No prompt, pergunte:

```text
What is the current currency exchange rate for Japan?
```

Saia com `Ctrl+C`.

### Resultado esperado
- Resposta com a cotacao do iene obtida via busca.

> Click **Check my progress**: *Chat with an agent via the command-line interface*.

---

## TAREFA 4: Aplicar padroes estruturados (geo_validator)

### Conceito
Com `output_schema`, o ADK obriga o modelo a responder **somente** no formato do modelo Pydantic. Agentes com schema de saida nao devem transferir a conversa para outros agentes, por isso as transferencias sao desabilitadas.

### Passo 1: Abrir o codigo

```bash
cd "$PROJECT_DIR"
cat "${GEO_AGENT}/agent.py"
```

### Passo 2: Ajustar o `agent.py`

Adicione o modelo Pydantic e os parametros na definicao do agente (mantendo o restante do arquivo, inclusive o bloco que executa o agente programaticamente):

```python
from pydantic import BaseModel, Field

class CountryCapital(BaseModel):
    capital: str = Field(description="Capital city of the country")

root_agent = Agent(
    # ... campos ja existentes (name, description, instruction) ...
    model=os.getenv("MODEL"),          # modelo Flash do lab
    output_schema=CountryCapital,
    disallow_transfer_to_parent=True,
    disallow_transfer_to_peers=True,
)
```

Se o arquivo nao importar `os`, adicione `import os` (ou use o nome do modelo diretamente em `model=`).

Conferir:

```bash
grep -n -E "CountryCapital|output_schema|disallow_transfer|model=" "${GEO_AGENT}/agent.py"
```

### Passo 3: Executar o agente programaticamente

```bash
cd "$PROJECT_DIR"
python3 "${GEO_AGENT}/agent.py"
```

### Explicacao dos parametros
- `CountryCapital(BaseModel)` com `capital: str`: schema JSON esperado pela Booking Engine.
- `output_schema=CountryCapital`: resposta validada contra o schema.
- `disallow_transfer_to_parent/peers=True`: o agente nao delega a conversa.
- `model`: modelo Flash informado no lab.

### Resultado esperado
- Saida em JSON, por exemplo `{"capital": "Paris"}`, e nao uma frase.

> Click **Check my progress**: *Run an agent programmatically*.

### Resumo visual

```text
Antes:  [geo_validator] --> "The capital of France is Paris."

        + CountryCapital(BaseModel)  output_schema
        + disallow_transfer_to_parent / peers

Depois: [geo_validator] --> {"capital": "Paris"}
```

---

## TAREFA 5: Restaurar o Brochure Auditor (llm_auditor)

### Conceito
O `llm_auditor` e um **SequentialAgent**: executa os sub-agentes em ordem. O `critic_agent` checa a afirmacao na web e o `reviser_agent` reescreve com base no resultado. O reviser foi comentado, entao o pipeline para apos a critica.

### Passo 1: Localizar o reviser comentado

```bash
cd "$PROJECT_DIR"
grep -n -i "reviser\|sub_agents" "${AUDITOR_AGENT}/agent.py"
```

### Passo 2: Reativar o reviser no `agent.py`

Descomente o import e inclua o reviser na lista, **depois** do critic:

```python
from google.adk.agents import SequentialAgent

from .sub_agents.critic import critic_agent
from .sub_agents.reviser import reviser_agent

llm_auditor = SequentialAgent(
    # ... name e description existentes ...
    sub_agents=[critic_agent, reviser_agent],
)

root_agent = llm_auditor
```

Conferir que nao ha mais linhas comentadas do reviser:

```bash
grep -n -i "reviser" "${AUDITOR_AGENT}/agent.py"
```

### Passo 3: Testar no ADK Dev UI

```bash
cd "$PROJECT_DIR"
adk web --allow_origins "regex:https://.*\.cloudshell\.dev"
```

Selecione `llm_auditor` e envie:

```text
Double check this: You can take a direct train from Hawaii to Japan.
```

Na aba de eventos, confirme a execucao do `critic_agent` e depois do `reviser_agent`. Se houver erro, use **+ New Session**.

### Explicacao dos parametros
- `SequentialAgent`: orquestra sub-agentes em sequencia, passando o estado adiante.
- `sub_agents=[critic_agent, reviser_agent]`: ordem critica -> revisao.

### Resultado esperado
- O critic aponta que a afirmacao e falsa e o reviser devolve a versao corrigida (Havai e Japao sao separados pelo Oceano Pacifico; nao ha trem direto).

> Click **Check my progress**: *Preview a multi-agent example*.

### Resumo visual

```text
Antes:  [llm_auditor SequentialAgent] --> [critic_agent] --> fim

Depois: [llm_auditor SequentialAgent]
              |
              v
        [critic_agent] -- fact-check na web --> achados
              |
              v
        [reviser_agent] -- reescreve a afirmacao --> texto corrigido
```

---

## Validacao

```bash
cd "$PROJECT_DIR"

# .env dos tres agentes
for agent in "$SEARCH_AGENT" "$GEO_AGENT" "$AUDITOR_AGENT"; do
  echo "== $agent"; grep -E "ENTERPRISE|PROJECT|LOCATION|MODEL" "${agent}/.env"
done

# Codigo
grep -n "google_search" "${SEARCH_AGENT}/agent.py"
grep -n -E "CountryCapital|output_schema|disallow_transfer" "${GEO_AGENT}/agent.py"
grep -n -E "reviser_agent|sub_agents" "${AUDITOR_AGENT}/agent.py"

# Execucao programatica
python3 "${GEO_AGENT}/agent.py"
```

Checklist esperado:
- `.env` com `GOOGLE_GENAI_USE_ENTERPRISE=true`, projeto do lab, `global` e o modelo do lab;
- Travel Scout com `tools=[google_search]`, respondendo no Web UI e na CLI;
- `geo_validator` retornando JSON `{"capital": ...}`;
- `llm_auditor` com `sub_agents=[critic_agent, reviser_agent]`.

---

## Resumo Visual da Arquitetura

Compilado: estado final ao termino do lab. Os detalhes de cada agente estao no `### Resumo visual` das Tarefas 2, 4 e 5.

```text
[Cloud Shell] adk web / adk run / python3
        |
        v
[adk_project]
  [my_google_search_agent] --tools--> [Google Search]
  [geo_validator]          --output_schema--> {"capital": "..."}
  [llm_auditor] SequentialAgent: [critic_agent] -> [reviser_agent]
        |
        | .env: GOOGLE_GENAI_USE_ENTERPRISE=true, location global
        v
[Agent Platform] (PROJECT_ID, MODEL Flash)  -- autenticacao ADC
```

---

## Troubleshooting

- `adk: command not found`:
  - rode `export PATH=$PATH:"/home/${USER}/.local/bin"` novamente (vale para cada sessao).
- Erro de API key / `GOOGLE_API_KEY`:
  - o `.env` nao foi carregado ou falta `GOOGLE_GENAI_USE_ENTERPRISE=true`; confira o arquivo dentro da pasta do agente (use **Toggle hidden files** no Editor).
- Erro de permissao / credenciais:
  - refaca `gcloud auth application-default login` com o email do aluno.
- Modelo nao encontrado / `404` de modelo:
  - confira `MODEL` com o nome exato do lab e `GOOGLE_CLOUD_LOCATION=global`.
- Erro no Dev UI apos editar o codigo:
  - pare (`Ctrl+C`), rode `adk web` de novo a partir de `~/adk_project` e use **+ New Session**.
- Resposta sem grounding no Travel Scout:
  - confirme `tools=[google_search]` e reforce na `instruction` o uso da busca.
- `geo_validator` ainda responde em frase:
  - confira `output_schema=CountryCapital` na definicao do agente que o script executa.
- `ImportError` no `llm_auditor`:
  - o import deve ser `from .sub_agents.reviser import reviser_agent` (descomentado) e a pasta `sub_agents/reviser` deve existir.

---

## Limpeza (Opcional)

```bash
# Parar processos do ADK (Ctrl+C nas abas, ou)
pkill -f "adk web"

# Remover o projeto local
rm -rf "$HOME/adk_project" "$HOME/adk_project.zip"
```

---

## Conceitos-Chave

- ADK (Agent Development Kit): framework para construir, testar (`adk web`, `adk run`) e executar agentes.
- Agent Platform via `.env`: projeto + localizacao + ADC, sem API keys.
- Built-in tool `google_search`: grounding das respostas em resultados de busca.
- `output_schema` (Pydantic): resposta estruturada e validada; combina com transferencias desabilitadas.
- `SequentialAgent`: pipeline multi-agente em ordem fixa (critic -> reviser).

---

## Fluxo Final

1. Definir variaveis (`MODEL` do lab, bucket do projeto).
2. Instalar o ADK, autenticar com ADC e baixar o `adk_project`.
3. Criar os `.env` dos tres agentes (Agent Platform, `global`, modelo do lab).
4. Adicionar `tools=[google_search]` ao Travel Scout e testar no `adk web`.
5. Testar o Travel Scout pela CLI (`adk run`).
6. Adicionar `CountryCapital` + `output_schema` + transferencias desabilitadas ao `geo_validator` e rodar com `python3`.
7. Reativar o `reviser_agent` no `llm_auditor` e testar no `adk web`.
