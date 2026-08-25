---
title: Capítulo 004 — Endereçamento IP e Máscaras de Sub-rede
description: Entenda o que é um endereço IP, a diferença entre redes públicas e privadas, e como máscaras de sub-rede e notação CIDR definem os limites de uma rede.
---

# Capítulo 004 — Endereçamento IP e Máscaras de Sub-rede

> **Entender antes de decorar.**

---

| Informação | Detalhes |
|---|---|
| **Módulo** | 02 — Redes |
| **Nível** | Iniciante |
| **Tempo estimado** | 10 a 15 minutos |
| **Pré-requisito** | [Capítulo 003 — Modelo TCP/IP](003-modelo-tcp-ip.md) |

---

## Objetivo deste capítulo

Ao final deste capítulo, você será capaz de:

- explicar o que é um endereço IP e para que ele serve;
- diferenciar endereços IP públicos e privados;
- entender o que é uma máscara de sub-rede e como ela define os limites de uma rede;
- ler e interpretar notação CIDR (como `/24`);
- determinar se dois endereços IP estão na mesma sub-rede;
- reconhecer por que isso importa diretamente para resposta a incidentes.

---

## Como sempre, vamos começar utilizando nossa imaginação.

Imagine que você está atuando em um incidente em andamento.

Chega uma mensagem da liderança:

> "Isolem o host **10.10.5.23** agora. Ele está na mesma sub-rede do servidor comprometido — risco alto de movimentação lateral."

Alguns minutos depois, outro alerta aparece, envolvendo o host **10.10.6.50**.

```text
Opção A
"Também começa com 10.10, deve ser a mesma rede. Vou isolar também."

Opção B
"Preciso confirmar isso calculando a sub-rede, não só olhando os primeiros números."
```

"Começar parecido" não é a mesma coisa que "estar na mesma sub-rede". E em um incidente real, isolar o host errado — ou deixar de isolar o host certo — tem consequências diretas.

> **Endereço IP sem máscara de sub-rede é uma informação incompleta.**

---

# O que é um endereço IP?

Um **endereço IP** é um identificador numérico atribuído a um dispositivo em uma rede, permitindo que ele seja localizado e que dados sejam roteados até ele.

No formato mais comum hoje (**IPv4**), um endereço é composto por 4 blocos de números (octetos), cada um variando de 0 a 255:

```text
192.168.1.10
```

!!! tip "IPv4 e IPv6"
    O IPv4 oferece cerca de 4,3 bilhões de endereços possíveis — número que já não é suficiente para todos os dispositivos conectados no mundo. Por isso existe o **IPv6**, com um espaço de endereçamento muito maior. Neste módulo, vamos focar em IPv4 por ser ainda o mais comum na maioria dos ambientes corporativos, mas é importante saber que o IPv6 existe e vem crescendo em adoção.

---

# IPs públicos x privados

Nem todo endereço IP é acessível pela internet. Alguns intervalos foram reservados especificamente para uso **interno**, dentro de redes privadas:

| Intervalo privado | CIDR | Uso comum |
|---|---|---|
| 10.0.0.0 – 10.255.255.255 | `10.0.0.0/8` | Redes corporativas grandes |
| 172.16.0.0 – 172.31.255.255 | `172.16.0.0/12` | Redes corporativas médias |
| 192.168.0.0 – 192.168.255.255 | `192.168.0.0/16` | Redes domésticas e pequenos escritórios |

Qualquer outro endereço fora desses intervalos é, em geral, um **IP público** — potencialmente acessível pela internet.

**Relevância para segurança:** reconhecer rapidamente se um IP em um log é privado ou público já ajuda a entender se o tráfego é interno ou está saindo/entrando pela internet.

---

# Rede e host — as duas partes de um endereço IP

Um endereço IP pode ser dividido conceitualmente em duas partes:

```text
Parte de REDE     — identifica a rede à qual o dispositivo pertence
Parte de HOST      — identifica o dispositivo específico dentro dessa rede
```

Uma analogia simples: pense em um endereço postal. "Rua das Flores" identifica a rua (a rede); "número 42" identifica a casa específica (o host).

O que define onde termina a parte de rede e começa a parte de host é a **máscara de sub-rede**.

---

# Máscara de sub-rede e notação CIDR

A máscara de sub-rede indica quantos bits do endereço são usados para identificar a rede. Ela pode ser escrita de duas formas equivalentes:

```text
Formato tradicional:  255.255.255.0
Notação CIDR:         /24
```

O número depois da barra (`/24`) indica quantos bits, da esquerda para a direita, pertencem à parte de rede.

| CIDR | Máscara | Hosts utilizáveis | Uso comum |
|---|---|---|---|
| `/8` | 255.0.0.0 | ~16 milhões | Redes muito grandes |
| `/16` | 255.255.0.0 | 65.534 | Redes de médio/grande porte |
| `/24` | 255.255.255.0 | 254 | Rede local comum (casa, escritório) |
| `/30` | 255.255.255.252 | 2 | Links ponto a ponto (ex: entre dois roteadores) |

!!! tip "Conexão com o Lab-004"
    Se você já fez o laboratório de firewall com pfSense deste projeto, a rede LAN padrão configurada foi `192.168.1.1/24` — exatamente esse formato. Isso significa uma rede com 254 endereços utilizáveis para hosts, de `192.168.1.1` a `192.168.1.254`.

---

# Determinando se dois IPs estão na mesma sub-rede

Para saber se dois endereços estão na mesma sub-rede `/24`, basta comparar os três primeiros octetos — eles precisam ser idênticos.

```text
10.10.5.23  /24  →  rede 10.10.5.0
10.10.5.200 /24  →  rede 10.10.5.0   ✔ mesma sub-rede

10.10.5.23  /24  →  rede 10.10.5.0
10.10.6.50  /24  →  rede 10.10.6.0   ✘ sub-redes diferentes
```

Voltando ao cenário do início do capítulo: `10.10.5.23` e `10.10.6.50` **não** estão na mesma sub-rede `/24`, mesmo os dois começando com "10.10". A diferença está no terceiro octeto — `.5` contra `.6` — o que já define redes distintas.

---

# Por que isso importa para segurança?

- **Segmentação de rede** (que vimos no capítulo de Defesa em Profundidade, em Fundamentos) depende diretamente de sub-redes bem definidas;
- entender os limites de uma sub-rede ajuda a estimar o **raio de impacto** (blast radius) de um host comprometido — movimentação lateral costuma ser mais direta dentro da mesma sub-rede;
- regras de firewall, DHCP e roteamento são configuradas usando notação CIDR — sem entender isso, fica difícil interpretar essas configurações.

---

# Aplicação em um SOC

### Ao analisar um alerta

- o IP de origem é interno (privado) ou externo (público)?
- o IP de destino é interno ou externo?
- origem e destino estão na mesma sub-rede?

### Ao ler configurações

- regras de firewall costumam referenciar sub-redes inteiras (ex: `192.168.1.0/24`) em vez de IPs individuais;
- entender CIDR permite saber exatamente qual o alcance de uma regra.

---

# Cenário prático — Fechando o incidente do início

Retomando o alerta:

```text
Host comprometido: 10.10.5.23/24
Novo alerta:        10.10.6.50
```

Com o raciocínio deste capítulo, dá para responder com precisão:

> `10.10.5.23/24` pertence à rede `10.10.5.0`.

> `10.10.6.50`, assumindo a mesma máscara `/24`, pertence à rede `10.10.6.0`.

> São sub-redes diferentes — o segundo host não está automaticamente sob o mesmo risco imediato de movimentação lateral que motivou o isolamento do primeiro.

Isso não significa ignorar o segundo alerta — significa investigá-lo com a prioridade correta, em vez de tratá-lo como extensão automática do primeiro incidente.

---

## Perguntas de investigação

??? question "1. O que significa a notação /24 em um endereço IP?"
    Significa que os primeiros 24 bits do endereço (os três primeiros octetos, em uma rede convencional) identificam a rede, restando os bits finais para identificar hosts individuais.

??? question "2. Um IP começando com 192.168 é sempre uma rede privada?"
    Sim, todo o intervalo 192.168.0.0/16 é reservado para uso privado. Mas isso não significa que dois endereços 192.168.x.x estejam automaticamente na mesma sub-rede — depende da máscara aplicada.

??? question "3. Por que isolar o host errado durante um incidente pode ser um problema?"
    Porque pode causar interrupção desnecessária em um sistema não comprometido, além de desviar atenção e recursos do host que realmente precisa de contenção.

??? question "4. Dois hosts com o mesmo terceiro octeto estão necessariamente na mesma sub-rede /24?"
    Nesse caso específico (/24), sim — os três primeiros octetos precisam ser idênticos. Mas com máscaras diferentes (como /16 ou /8), essa regra muda.

??? question "5. Por que o esgotamento de endereços IPv4 é relevante para quem trabalha com segurança?"
    Porque técnicas como NAT (que permitem várias máquinas privadas compartilharem um único IP público) tornam-se ainda mais centrais, e entender essas técnicas ajuda a interpretar corretamente logs onde múltiplos hosts aparentam ter a mesma origem.

---

# O que não fazer

- não assuma que dois IPs parecidos estão na mesma sub-rede sem verificar a máscara;
- não trate `/24` e `255.255.255.0` como conceitos diferentes — são a mesma coisa, escritos de forma diferente;
- não ignore a máscara de sub-rede ao analisar um alerta — ela muda completamente a interpretação;
- não confunda "IP privado" com "IP seguro" — privado só indica o escopo de uso, não o nível de proteção.

---

# Erros comuns

### "Todo IP que começa com 192.168 é da minha rede"

Não necessariamente.

É um IP de uma faixa **privada** — mas "sua rede" especificamente depende da sub-rede configurada, não só da faixa geral.

### "/24 e 255.255.255.0 são coisas diferentes"

Não.

São exatamente a mesma máscara, apenas escritas em notações diferentes — CIDR e decimal.

### "Sub-rede é só matemática, não tem nada a ver com segurança"

Não.

Sub-rede define literalmente até onde um problema pode se espalhar lateralmente antes de esbarrar em uma fronteira de rede diferente.

---

# Resumo

Neste capítulo, aprendemos que:

- um **endereço IP** identifica um dispositivo em uma rede;
- existem faixas de IP reservadas para uso **privado** (10.x, 172.16-31.x, 192.168.x);
- todo endereço IP tem uma parte de **rede** e uma parte de **host**, definidas pela **máscara de sub-rede**;
- a notação **CIDR** (como `/24`) é uma forma abreviada de expressar essa máscara;
- é possível determinar se dois IPs estão na mesma sub-rede comparando a parte de rede de cada um;
- entender sub-redes é essencial para avaliar o alcance de um incidente e interpretar regras de firewall.

```text
Não pergunte apenas:

"Qual é o IP?"

Pergunte também:

"Qual é a máscara de sub-rede?"
"Esse IP é público ou privado?"
"Esse host está na mesma sub-rede do host comprometido?"
```

> **Um IP sozinho é só um número. Com a máscara certa, ele vira um mapa.**

---

# Checkpoint

Antes de seguir para o próximo capítulo, confirme se você consegue responder:

- [ ] O que é um endereço IP?
- [ ] Quais são as três principais faixas de IP privado?
- [ ] O que define a parte de rede e a parte de host em um endereço IP?
- [ ] O que significa a notação CIDR `/24`?
- [ ] Como determinar se dois IPs estão na mesma sub-rede `/24`?
- [ ] Por que sub-redes importam para avaliar o alcance de um incidente?
- [ ] `/24` e `255.255.255.0` representam coisas diferentes ou a mesma coisa?

---

# Glossário

| Termo | Definição |
|---|---|
| **Endereço IP** | Identificador numérico atribuído a um dispositivo em uma rede. |
| **IPv4** | Versão do protocolo IP que usa endereços de 32 bits, no formato de 4 octetos. |
| **IPv6** | Versão mais recente do protocolo IP, com espaço de endereçamento muito maior que o IPv4. |
| **Octeto** | Cada um dos quatro blocos numéricos (0-255) que compõem um endereço IPv4. |
| **Máscara de sub-rede** | Define quais bits de um endereço IP correspondem à rede e quais correspondem ao host. |
| **CIDR** | Classless Inter-Domain Routing; notação abreviada (ex: `/24`) para representar a máscara de sub-rede. |
| **Rede privada** | Faixa de endereços IP reservada para uso interno, não roteável diretamente pela internet. |

---

# Referências

- [RFC 1918 — Address Allocation for Private Internets](https://datatracker.ietf.org/doc/html/rfc1918)
- [Cloudflare — O que é um endereço IP?](https://www.cloudflare.com/learning/dns/glossary/what-is-my-ip-address/)
- [Cisco Networking Academy](https://www.netacad.com/courses/networking)

---

# Próximo capítulo

No próximo capítulo, vamos estudar **Portas e Protocolos Essenciais** — como TCP, UDP e ICMP se conectam ao que já vimos até aqui.

[← Capítulo anterior: Modelo TCP/IP](003-modelo-tcp-ip.md){ .md-button }

[Próximo: Portas e Protocolos →](005-portas-e-protocolos-essenciais.md){ .md-button .md-button--primary }

---

> **Entender antes de decorar.**
