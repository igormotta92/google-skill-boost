# Networks - Guia de Estudo

## 1) Conceitos Essenciais

- IP: endereco de um dispositivo na rede.
- Sub-rede: divisao logica de uma rede maior.
- VPC: rede virtual privada na nuvem.
- Firewall: conjunto de regras que permite ou bloqueia trafego.
- NAT: traducao de enderecos (e portas) entre redes.
- CIDR: notacao de rede no formato `IP/prefixo` (ex.: `10.0.0.0/24`).
- LAN: rede local (casa, escritorio, Wi-Fi interno. Normalmente usa IP privado (como 192.168.x.x, 10.x.x.x).).
- WAN: rede de longa distancia que conecta LANs (ex.: internet).

## 2) CIDR no IPv4

No IPv4, um endereco tem 32 bits.

$$
	\text{Total de IPs} = 2^{(32 - \text{prefixo})}
$$

Leitura rapida:
- Prefixo menor (ex.: `/16`) -> bloco maior.
- Prefixo maior (ex.: `/32`) -> bloco menor.

Exemplos:
- `200.10.10.5/32` -> 1 IP.
- `200.10.10.0/24` -> 256 IPs.

### Prefixos mais usados (IPv4)

| Prefixo | Mascara | Total de IPs |
|---|---|---|
| /32 | 255.255.255.255 | 1 |
| /30 | 255.255.255.252 | 4 |
| /29 | 255.255.255.248 | 8 |
| /28 | 255.255.255.240 | 16 |
| /27 | 255.255.255.224 | 32 |
| /26 | 255.255.255.192 | 64 |
| /25 | 255.255.255.128 | 128 |
| /24 | 255.255.255.0 | 256 |
| /23 | 255.255.254.0 | 512 |
| /22 | 255.255.252.0 | 1024 |
| /21 | 255.255.248.0 | 2048 |
| /20 | 255.255.240.0 | 4096 |
| /19 | 255.255.224.0 | 8192 |
| /18 | 255.255.192.0 | 16384 |
| /17 | 255.255.128.0 | 32768 |
| /16 | 255.255.0.0 | 65536 |

Regra pratica:
- Para acesso administrativo (SSH/RDP), prefira blocos restritos, como `/32`, quando fizer sentido.

## 3) CIDR no IPv6

No IPv6, um endereco tem 128 bits.

$$
	\text{Total de IPs} = 2^{(128 - \text{prefixo})}
$$

Pontos importantes:
- Faixa de prefixo: `/0` a `/128`.
- `/128` representa um unico endereco.
- `/64` e o tamanho de sub-rede mais comum.

Exemplos:
- `2001:db8::1/128` -> 1 IP.
- `2001:db8:abcd:10::/64` -> sub-rede IPv6 padrao.

## 4) IPv4 x IPv6

| Tema | IPv4 | IPv6 |
|---|---|---|
| Tamanho do endereco | 32 bits | 128 bits |
| Faixa de prefixo | `/0` a `/32` | `/0` a `/128` |
| Quantidade de enderecos | Limitada | Muito maior |
| NAT | Muito comum | Menos necessario |
| Formato | Ex.: `192.168.1.10` | Ex.: `2001:db8::10` |

Quando usar:
- IPv4: ambientes legados e compatibilidade imediata.
- IPv6: escala de enderecamento e arquitetura futura.
- Melhor caminho: dual-stack (IPv4 + IPv6) durante a migracao.

## 5) NAT (Network Address Translation)

NAT traduz enderecos entre rede privada e rede publica. Normalmente tambem traduz portas (PAT).

### Fluxo basico de NAT

SNAT (NAT saida):
1. Origem interna envia `10.0.1.25:53000`.
2. NAT troca para `200.10.10.8:41021`.
3. Resposta volta para `200.10.10.8:41021`.
4. NAT consulta estado da sessao e entrega para `10.0.1.25:53000`.

DNAT (NAT entrada):
1. Cliente externo envia para `200.10.10.20:443`.
2. NAT troca destino para `10.0.2.15:8443`.
3. Servico interno responde de `10.0.2.15:8443` para o cliente.
4. NAT traduz retorno para `200.10.10.20:443` e mantém a sessao consistente.

### Tipos comuns

- SNAT: altera origem (saida para internet).
- DNAT: altera destino (publicar servico interno).
- PAT/NAT overload: varios hosts internos compartilham 1 IP publico via portas.

### NAT com pool de IPs publicos

- O NAT escolhe dinamicamente um IP do pool para cada sessao.
- A identificacao da conexao depende de IP + porta + estado.
- Util para escalar saida e distribuir carga entre IPs publicos.

Regra de ouro:
- NAT nao substitui firewall; ele resolve traducao/endereco, nao politica de seguranca.

## 6) Resumo para Revisao

- CIDR define tamanho do bloco; prefixo maior significa bloco menor.
- IPv4 e limitado e depende bastante de NAT.
- IPv6 tem muito mais espaco e reduz a necessidade de NAT.
- Em migracao real, dual-stack costuma ser a estrategia mais segura.

## 7) Glossario Rapido

- IP: identificador de rede de um dispositivo.
- CIDR: notacao de bloco por prefixo.
- Sub-rede: segmento logico de rede.
- VPC: rede privada virtual na nuvem.
- Firewall: regra de controle de trafego.
- NAT: traducao de enderecos/portas.
- Dual-stack: IPv4 e IPv6 simultaneamente.
- LAN: rede local de curto alcance.
- WAN: rede ampla que conecta redes locais.