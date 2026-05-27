# Tipos de Conexao com o Google Cloud

Este guia resume as principais formas de conectar seu ambiente (on-premises, datacenter, outros provedores ou internet) ao Google Cloud.

## Visao Rapida

| Tipo | O que e | Melhor para |
|---|---|---|
| Cloud VPN | Tunel IPsec criptografado pela internet publica | Inicio rapido, custo menor, trafego moderado |
| Direct Peering | Ligacao direta com a borda do Google para acessar APIs/servicos publicos do Google | Latencia melhor para consumo de servicos publicos |
| Carrier Peering | Conexao via operadora parceira ate a borda do Google | Empresas que preferem contratar conectividade com operadora |
| Dedicated Interconnect | Link fisico dedicado entre seu datacenter e o Google | Alto volume, previsibilidade e baixa latencia |
| Partner Interconnect | Conexao privada ao Google via parceiro (sem link fisico proprio) | Conexao privada com menos complexidade de implantacao |
| Cross-Cloud Interconnect | Conexao privada entre Google Cloud e outro provedor de nuvem | Arquiteturas multi-cloud com alto desempenho |

## 1) Cloud VPN

Conexao segura usando IPsec sobre internet publica.

### Quando usar
- Voce precisa conectar rapido.
- O trafego nao e extremamente alto.
- Quer reduzir investimento inicial.

### Exemplo de cenario
Uma empresa com ERP no datacenter local quer replicar dados para BigQuery diariamente. Usa Cloud VPN para integrar on-premises com uma VPC no Google Cloud, sem contratar circuito dedicado.

## 2) Direct Peering

Conexao direta entre sua rede e a rede de borda do Google para acessar servicos publicos do Google (nao e acesso privado a VPC).

### Quando usar
- Voce consome bastante APIs e servicos publicos do Google.
- Quer reduzir latencia e melhorar estabilidade comparado a internet comum.

### Exemplo de cenario
Uma empresa de midia publica grandes volumes no YouTube e usa varias APIs Google. Com Direct Peering, melhora tempo de resposta para esses servicos publicos.

## 3) Carrier Peering

Semelhante ao Direct Peering, mas feito por meio de uma operadora parceira.

### Quando usar
- Sua estrategia e contratar conectividade com uma unica operadora.
- Quer simplificar operacao sem negociar peering direto com o Google.

### Exemplo de cenario
Uma rede varejista nacional ja usa uma operadora para WAN. A mesma operadora entrega Carrier Peering para acesso otimizado a servicos publicos Google.

## 4) Dedicated Interconnect

Conexao fisica dedicada entre seu datacenter e o Google (10 Gbps ou 100 Gbps por circuito).

### Quando usar
- Trafego alto e continuo.
- Baixa latencia e previsibilidade sao criticas.
- Requisito de confiabilidade empresarial.

### Exemplo de cenario
Uma area de dados move dezenas de TB por dia de on-premises para GCS e BigQuery. Dedicated Interconnect reduz custo por volume e aumenta estabilidade em relacao a internet/VPN.

## 5) Partner Interconnect

Conexao privada com Google Cloud por um parceiro homologado (sem precisar de presenca fisica em colocation do Google).

### Quando usar
- Voce quer conectividade privada, mas nao quer/nao pode implantar circuito fisico dedicado.
- Precisa de menor barreira de entrada do que Dedicated Interconnect.

### Exemplo de cenario
Uma empresa media quer acessar workloads em VPC privada no Google Cloud com melhor seguranca e desempenho do que VPN, usando um parceiro regional.

## 6) Cross-Cloud Interconnect

Conexao privada dedicada entre Google Cloud e outro provedor de nuvem.

### Quando usar
- Ambiente multi-cloud (ex.: Google Cloud + AWS/Azure).
- Integracoes de alto volume e baixa latencia entre nuvens.

### Exemplo de cenario
Uma empresa mantem banco em outra nuvem e analytics no BigQuery. Cross-Cloud Interconnect reduz dependencia da internet publica e melhora desempenho de ETL entre clouds.

## Como escolher rapidamente

- Comecar rapido e barato: Cloud VPN.
- Acessar servicos publicos Google com melhor rota: Direct Peering ou Carrier Peering.
- Conectividade privada com VPC e alto desempenho: Dedicated Interconnect ou Partner Interconnect.
- Integracao privada entre nuvens: Cross-Cloud Interconnect.

## Regra pratica

Se a prioridade for **tempo de implantacao**, comece de VPN e evolua para Interconnect.
Se a prioridade for **escala e previsibilidade**, va direto para Dedicated/Partner Interconnect.