# Load Balancers no Google Cloud

Um Load Balancer recebe o trafego em um endereco IP e o distribui entre varios servidores. Assim, os usuarios acessam um unico endereco, mesmo que existam varios servidores por tras dele.

## Network Load Balancer

Distribui trafego de rede, como TCP e UDP. Ele encaminha conexoes com base no IP, na porta e no protocolo, sem analisar o caminho de uma URL.

**Exemplo:** um servico TCP esta disponivel no IP `203.0.113.10`, porta `443`. O Network Load Balancer recebe as conexoes nesse IP e as encaminha para um dos servidores saudaveis configurados.

**Use quando:** precisar distribuir trafego TCP/UDP ou de outros protocolos de rede.

Ele nao escolhe necessariamente o servidor "mais disponivel": verifica quais servidores estao saudaveis e distribui as conexoes entre os servidores elegiveis, conforme o tipo e a configuracao do balanceador.

## Application Load Balancer

Distribui trafego HTTP/HTTPS e pode escolher o servidor com base no dominio ou no caminho da URL.

**Exemplo:** `loja.exemplo.com/api` vai para o grupo de servidores da API, enquanto `loja.exemplo.com/imagens` vai para o grupo de imagens. Dentro do grupo escolhido, o balanceador encaminha a requisicao para um servidor saudavel. Os usuarios podem usar o mesmo IP publico.

**Use quando:** hospedar sites ou APIs e precisar direcionar requisicoes por dominio ou caminho.

### Como ele sabe para onde enviar?

O balanceador nao descobre isso sozinho: alguem configura um **mapa de URLs** com regras. Cada regra associa um dominio ou caminho a um **servico de backend**, que indica o grupo de servidores que pode atender aquela requisicao. Verificacoes de saude ajudam a evitar servidores indisponiveis.

**Backend nao significa Network Load Balancer (NLB).** Nesse contexto, e a configuracao/grupo de destino do Application Load Balancer — por exemplo, um grupo de instancias ou endpoints. O Application Load Balancer escolhe um servidor saudavel desse grupo; nao precisa encaminhar a requisicao para outro NLB.

### Por que o NLB do lab usa um pool?

Os dois tipos precisam saber quais destinos podem receber trafego. No ALB, essa funcao e feita por um **backend service**; o **mapa de URLs** escolhe qual backend service usar. No exemplo de Network Load Balancer mais antigo deste material, o grupo de instancias aparece como **target pool**. E uma diferenca de configuracao e nomenclatura, nao porque apenas o NLB precisa agrupar servidores. Em balanceadores de rede passthrough mais novos do Google Cloud, tambem e possivel usar backend service.

```text
/api      -> backend-api      -> servidores da API
/imagens  -> backend-imagens  -> servidores de imagens
outros    -> backend-site     -> servidores do site
```

Assim, ao receber `https://loja.exemplo.com/api/clientes`, ele identifica o caminho `/api`, consulta a regra configurada e encaminha a requisicao para um servidor saudavel do `backend-api`.

### Diferenca principal entre Application e Network

Nao e apenas o mapeamento por caminho. O **Application Load Balancer** entende HTTP/HTTPS: pode rotear por dominio e caminho e aplicar regras de aplicacao. O **Network Load Balancer** trata conexoes na camada de rede/transporte, usando IP, porta e protocolo; nao interpreta `/api` ou `/imagens` como caminhos de URL.

Ambos podem verificar a saude dos destinos e distribuir trafego entre destinos saudaveis. A escolha depende do protocolo e dos recursos necessarios, nao apenas de querer distribuir carga.

### ALB com apenas um backend

Nesse caso, o ALB pode encaminhar todas as requisicoes para o mesmo backend padrao, sem regras personalizadas por caminho ou dominio. **O resultado parece parecido** com um NLB porque ambos distribuem trafego entre destinos saudaveis, mas nao sao equivalentes: o ALB continua entendendo HTTP/HTTPS e pode aplicar recursos de aplicacao; o NLB trabalha com conexoes de rede, sem interpretar caminhos como `/api`. No Google Cloud, detalhes como terminacao TLS e preservacao do IP do cliente dependem do tipo de balanceador escolhido.

### Exemplos de uso

**Application Load Balancer (HTTP/HTTPS):**

- **Site e API:** `/` vai para os servidores do site; `/api` vai para os servidores da API.
- **Varios dominios:** `loja.exemplo.com` vai para a loja; `suporte.exemplo.com` vai para o portal de suporte.
- **Aplicacao web publica:** clientes acessam um IP publico por HTTPS; o balanceador encaminha cada requisicao para o grupo correto e para um destino saudavel.
- **Servico interno HTTP:** sistemas na VPC acessam um IP privado e regras de URL direcionam as requisicoes entre servicos internos.

**Network Load Balancer (TCP/UDP):**

- **Jogo online via UDP:** clientes conectam ao IP e porta do servico; o balanceador encaminha o trafego para servidores de jogo saudaveis.
- **Servico TCP que nao usa HTTP:** clientes conectam a uma porta TCP, e o balanceador distribui as conexoes sem interpretar URLs.
- **Servico TCP/UDP privado:** aplicacoes na VPC acessam um IP privado para chegar a um conjunto de servidores, sem expor o servico diretamente a internet.
- **Preservacao do IP do cliente:** em cenarios que exigem que o servidor receba o IP original do cliente, escolha um Network Load Balancer compativel com esse requisito (por exemplo, um passthrough).

## Internal Load Balancer

Internal indica que o acesso e privado, pela VPC (ou redes conectadas a ela), e nao diretamente pela internet. Nao e um terceiro metodo de balanceamento: um Load Balancer interno pode ser de rede (TCP/UDP) ou de aplicacao (HTTP/HTTPS).

**Exemplo:** servidores da VPC acessam `10.0.0.10`, o IP privado do Load Balancer, para chegar a um servico interno. Ele pode encaminhar por IP/porta (tipo Network) ou por URL (tipo Application), conforme o balanceador criado.

**Use quando:** sistemas internos precisarem acessar um servico de forma confiavel pela rede privada.

## Como o programador usa

Em geral, o balanceador e configurado na infraestrutura. O codigo da aplicacao usa o endereco do balanceador, sem escolher diretamente uma instancia.

- **Application:** o codigo faz requisicoes HTTP/HTTPS normalmente. Por exemplo, `fetch("https://api.exemplo.com/clientes")`; regras do balanceador podem encaminhar `/clientes` para o backend da API.
- **Network:** o cliente da aplicacao conecta ao host e porta, por exemplo `servico.exemplo.com:9000`, usando o protocolo TCP ou UDP esperado pelo servico. Nao se usa caminho HTTP como `/api`.
- **Internal:** o codigo de outro servico usa um nome/IP privado, por exemplo `http://pagamentos.interno`, acessivel pela VPC. O DNS interno deve apontar para o IP privado do balanceador.

Como responsavel pelo servico, o programador tambem configura a aplicacao para escutar na porta esperada e, quando aplicavel, disponibiliza um endpoint de verificacao de saude, como `/health`. O balanceador usa essa verificacao para evitar enviar trafego a instancias indisponiveis.

## Conferindo seu entendimento

- **Network:** sim, ele distribui conexoes entre servidores saudaveis. A decisao usa IP, porta e protocolo; nao o caminho da URL.
- **Application:** sim, ele pode direcionar uma URL para um grupo de servidores e, dentro desse grupo, distribuir as requisicoes entre os servidores saudaveis.
- **Internal:** quase. Ele nao faz os dois automaticamente; significa que o acesso e privado. Pode combinar o escopo interno com um balanceador Network ou Application.

## Fluxo de exemplo

```text
USUARIO
  |
  | HTTPS: https://loja.exemplo.com/api
  v
[IP publico do Application Load Balancer]
  |
  | Le dominio e caminho: /api
  v
[Regra: /api -> grupo da API]
  |
  | Escolhe um servidor saudavel do grupo
  +--------------------+
  |                    |
  v                    v
[API 1 - saudavel]   [API 2 - saudavel]

Se a URL for /imagens, a regra encaminha para o grupo de imagens.

Exemplo interno:
[Servico na VPC] -> [IP privado 10.0.0.10] -> [servidor saudavel]
                   balanceador interno
```

## Resumo

| Tipo | Trafego / direcionamento | IP tipico |
|---|---|---|
| Network | IP, porta e protocolo (ex.: TCP/UDP); distribui conexoes saudaveis | Publico ou privado |
| Application | HTTP/HTTPS; pode rotear por dominio/caminho e distribuir dentro do grupo | Publico ou privado |
| Internal | Acesso pela rede privada; pode ser Network ou Application | Privado |

**Regra pratica:** Network e Application descrevem como o trafego e tratado; Internal descreve onde o servico e acessivel. Portanto, um Load Balancer pode ser, por exemplo, um *Internal Application Load Balancer*.