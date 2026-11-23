# ALB e NLB na pratica

Este guia compara o **Application Load Balancer (ALB)** e o **Network Load Balancer (NLB)** do Google Cloud com foco em como uma aplicacao os utiliza.

## A ideia principal

Os dois recebem trafego em um endereco e podem distribui-lo entre destinos saudaveis. A diferenca e **que tipo de informacao usam para encaminhar o trafego**:

- **ALB:** entende requisicoes HTTP/HTTPS, como dominio e caminho (`/api`).
- **NLB:** encaminha conexoes de rede conforme IP, porta e protocolo (TCP/UDP). Nao interpreta caminhos de URL.

Pense no ALB como um recepcionista que le o endereco do pedido (`/api` ou `/imagens`). O NLB e como uma central que encaminha uma conexao para um destino, sem ler o conteudo da requisicao HTTP.

## O que o programador faz?

Normalmente, o programador **nao escolhe a instancia** que vai responder. A equipe configura o balanceador e seus destinos; o codigo cliente usa o endereco do balanceador.

### Com ALB: a aplicacao faz uma requisicao web

```bash
curl https://app.exemplo.com/api/pedidos
```

Uma regra configurada na infraestrutura pode encaminhar `/api` ao grupo da API. O codigo cliente continua usando uma URL normal.

```text
Navegador ou cliente
        |
        | HTTPS: app.exemplo.com/api/pedidos
        v
[Application Load Balancer]
        |
        | regra: /api -> backend da API
        v
[API 1]   [API 2]   [API 3]
            ^
            +-- escolhe um destino saudavel
```

Com apenas um backend, as requisicoes podem ir todas para ele por uma regra padrao. O ALB ainda entende HTTP/HTTPS; apenas nao ha varios backends entre os quais escolher por URL.

### Com NLB: a aplicacao conecta a um endereco e porta

Para um servico TCP que nao e uma API HTTP, por exemplo, um cliente Python pode abrir uma conexao:

```python
import socket

with socket.create_connection(("tcp.exemplo.com", 9000)) as conexao:
    conexao.sendall(b"mensagem do protocolo do servico")
    resposta = conexao.recv(1024)
```

O cliente conecta ao IP/hostname e porta. O NLB encaminha a conexao para um destino saudavel, mas nao sabe se os dados representam `/api`, JSON ou uma pagina web.

```text
Cliente TCP
    |
    | tcp.exemplo.com:9000
    v
[Network Load Balancer]
    |
    | IP + porta + protocolo TCP
    v
[Servidor 1]   [Servidor 2]   [Servidor 3]
```

Se o servico usa HTTP, tambem e possivel enviar HTTP por um NLB em configuracoes compativeis. Mas o NLB nao passa a rotear por caminho: `/api` e `/imagens` continuam sendo parte do HTTP, que o NLB nao usa para escolher o destino.

## Backend, grupo e pool: como se relacionam?

Todo balanceador precisa saber **quais servidores podem receber trafego**. O grupo configurado nao e outro balanceador.

```text
ALB:
URL map (/api) -> backend service -> grupo de servidores

NLB do exemplo antigo do lab:
forwarding rule (IP:porta) -> target pool -> instancias
```

- O **backend service** e um recurso de configuracao que liga o balanceador a destinos, como um grupo de instancias. O ALB usa um **URL map** para escolher qual backend service atende cada regra.
- O **target pool** do lab de NLB e um recurso que agrupa instancias para receber o trafego. E esse grupo que a regra de encaminhamento do lab aponta.
- Portanto, **o ALB tambem tem destinos agrupados**. Os nomes e recursos de configuracao variam conforme o tipo e a geracao do balanceador; NLBs passthrough mais novos tambem podem usar backend services.
- Health checks verificam se os destinos estao aptos. O balanceador distribui entre destinos saudaveis, nao necessariamente para o servidor "mais livre".

## O que acontece se o ALB tiver um unico backend?

Imagine que `app.exemplo.com` tenha uma unica regra padrao apontando para `backend-app`. Todas as requisicoes vao para esse backend, e ele pode conter uma ou varias instancias.

```text
https://app.exemplo.com/qualquer-caminho
                  |
                  v
          [Application LB]
                  |
            [backend-app]
             /         \
          [VM 1]      [VM 2]
```

Nesse cenario, **o resultado pode parecer com o de um NLB**: ambos entregam trafego a servidores saudaveis. Mas nao sao equivalentes:

| Pergunta | ALB | NLB |
|---|---|---|
| Entende HTTP/HTTPS? | Sim | Nao interpreta HTTP como camada de aplicacao |
| Pode direcionar por `/api`? | Sim, com URL map configurado | Nao |
| Encaminha por IP, porta e protocolo? | Recebe o protocolo de aplicacao configurado | Sim, essa e a base do encaminhamento |
| Precisa de destinos configurados? | Sim, por exemplo em backend service | Sim, por exemplo em target pool ou backend service |

Use ALB quando o servico e web e precisa de recursos HTTP/HTTPS, mesmo que inicialmente exista apenas um backend. Use NLB quando precisa encaminhar conexoes TCP/UDP ou quando requisitos de rede do servico pedem esse tipo de balanceador.

## E o Internal Load Balancer?

**Internal** descreve quem consegue acessar o endereco: clientes na VPC ou em redes conectadas, usando um IP privado. Nao e uma terceira forma de decidir entre servidores.

```text
Servico dentro da VPC
        |
        | acessa IP privado 10.0.0.10
        v
[Load Balancer interno]
        |
        v
[Destinos privados saudaveis]
```

O balanceador interno pode ser de aplicacao (HTTP/HTTPS, incluindo regras de URL) ou de rede (conforme o protocolo). Um cliente externo da internet nao acessa diretamente esse IP privado.

## Como isso aparece nos labs deste repositorio?

- No lab [Set Up Network Load Balancers](../Associeted%20Cloud%20Enginer/S1%20-%20Implementing%20Cloud%20Load%20Balancing%20for%20Compute%20Engine/Set%20Up%20Network%20Load%20Balancers.md), procure `target-pools create`, `target-pools add-instances` e a forwarding rule com `--target-pool`. Esses passos definem o grupo de destinos e ligam a ele o IP/porta de entrada.
- No lab [Set Up Application Load Balancers](../Associeted%20Cloud%20Enginer/S1%20-%20Implementing%20Cloud%20Load%20Balancing%20for%20Compute%20Engine/Set%20Up%20Application%20Load%20Balancers.md), acompanhe a configuracao de backend service, URL map, proxy HTTP(S) e forwarding rule. O URL map escolhe o backend a partir das regras HTTP.
- O lab [Set Up Internal Load Balancers](../Associeted%20Cloud%20Enginer/S1%20-%20Implementing%20Cloud%20Load%20Balancing%20for%20Compute%20Engine/Set%20Up%20Internal%20Load%20Balancers.md) mostra um IP interno e um backend service acessados por outra instancia da VPC.

**Resumo em uma frase:** ALB decide com base em informacoes da requisicao web; NLB encaminha conexoes de rede; *internal* diz que o endereco e privado.
