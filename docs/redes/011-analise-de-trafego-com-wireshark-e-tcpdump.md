---
title: Capítulo 011 — Análise de Tráfego com Wireshark e tcpdump
description: Aprenda a diferença entre Wireshark e tcpdump, filtros de captura e de exibição (incluindo filtros combinados), e como transformar uma captura de tráfego caótica em uma investigação organizada.
---

# Capítulo 011 — Análise de Tráfego com Wireshark e tcpdump

> **Entender antes de decorar.**

---

| Informação | Detalhes |
|---|---|
| **Módulo** | 02 — Redes |
| **Nível** | Iniciante |
| **Tempo estimado** | 15 a 20 minutos |
| **Pré-requisito** | [Capítulo 010 — VPNs e Criptografia em Trânsito](010-vpns-e-criptografia-em-transito.md) |

---

## Objetivo deste capítulo

Ao final deste capítulo, você será capaz de:

- diferenciar Wireshark e tcpdump e saber quando usar cada um;
- entender por que uma captura "crua" de tráfego parece caótica;
- diferenciar filtro de captura de filtro de exibição;
- usar filtros básicos e combinados para isolar tráfego relevante;
- entender para que serve a função "Follow TCP Stream";
- saber o que procurar, na prática, ao analisar uma captura de tráfego.

---

## Como sempre, vamos começar utilizando nossa imaginação.

Você abre o Wireshark pela primeira vez, clica em "iniciar captura" na sua própria placa de rede — e em poucos segundos a tela já está lotada de centenas de linhas passando rapidamente, mesmo sem você estar fazendo absolutamente nada.

```text
Opção A
"Tem tráfego demais. Não dá pra entender nada disso."

Opção B
"Preciso aprender a filtrar antes de tentar entender tudo de uma vez."
```

Esse é um momento clássico de quem começa em análise de tráfego. A boa notícia: você já tem toda a base teórica necessária dos capítulos anteriores. Falta só a ferramenta certa — e saber filtrar o ruído.

> **Uma captura de tráfego não é caótica. Ela só ainda não foi filtrada.**

---

# Wireshark e tcpdump: duas ferramentas, um mesmo propósito

| | Wireshark | tcpdump |
|---|---|---|
| **Interface** | Gráfica | Linha de comando |
| **Plataforma** | Windows, Linux, macOS | Nativo em praticamente todo sistema Unix/Linux |
| **Uso típico** | Análise visual detalhada | Captura rápida, inclusive em servidores sem interface gráfica |
| **Ponto forte** | Navegação e inspeção interativa | Leveza e automação via scripts |

Na prática, as duas se complementam: é comum capturar tráfego com tcpdump em um servidor remoto, salvar em um arquivo `.pcap`, e depois abrir esse arquivo no Wireshark para uma análise visual mais profunda — aproveitando o melhor dos dois mundos.

---

# Por que a captura "crua" parece um caos

Mesmo uma máquina "parada" gera tráfego constantemente:

- consultas **DNS** de fundo (capítulo 006);
- tráfego **ARP** perguntando "quem tem esse IP?" (capítulo 002);
- pacotes de **broadcast** dentro da rede local (capítulo 008);
- atualizações automáticas, telemetria de aplicativos, sincronizações em segundo plano.

Isso é normal — e é exatamente por isso que filtros existem: não para "esconder" tráfego, mas para permitir que você foque no que importa para sua investigação específica.

---

# Capturando com o Wireshark — a anatomia da tela

O Wireshark organiza cada pacote capturado em três painéis:

1. **Lista de pacotes** — resumo de cada pacote (origem, destino, protocolo, informação);
2. **Detalhes do pacote** — a árvore de protocolos, camada por camada;
3. **Bytes do pacote** — o conteúdo bruto, em hexadecimal.

O segundo painel deve parecer familiar — é exatamente a estrutura que vimos no capítulo de TCP/IP:

```text
Frame 1: 74 bytes
Ethernet II
Internet Protocol Version 4
Transmission Control Protocol
Hypertext Transfer Protocol
```

Cada linha corresponde a uma camada, de baixo para cima — Enlace, Internet, Transporte e Aplicação. O Wireshark literalmente "abre" o encapsulamento pra você, camada por camada, e clicando em cada uma dessas linhas o painel de bytes destaca exatamente qual trecho do pacote corresponde àquela camada.

---

# Filtros de captura x filtros de exibição

Essa é uma das confusões mais comuns entre iniciantes — e são coisas diferentes:

| | Filtro de captura | Filtro de exibição |
|---|---|---|
| **Quando age** | Antes da captura | Depois da captura |
| **Efeito** | Descarta o que não corresponde, permanentemente | Apenas esconde da visualização — os dados continuam ali |
| **Sintaxe** | BPF (Berkeley Packet Filter) | Sintaxe própria do Wireshark |

Alguns filtros de **exibição** úteis para começar:

| Filtro | O que mostra |
|---|---|
| `ip.addr == 8.8.8.8` | Tráfego de/para um IP específico |
| `tcp.port == 443` | Tráfego em uma porta específica |
| `dns` | Apenas tráfego DNS |
| `http.request` | Apenas requisições HTTP |
| `tcp.flags.syn == 1` | Apenas pacotes com flag SYN (útil pra observar tentativas de conexão) |

### Combinando filtros

Filtros de exibição podem ser combinados com operadores lógicos, permitindo investigações mais precisas:

| Filtro combinado | O que mostra |
|---|---|
| `ip.addr == 203.0.113.50 && tcp.port == 4444` | Tráfego para um IP específico, apenas na porta 4444 |
| `dns && !ip.addr == 8.8.8.8` | Todo tráfego DNS, exceto o destinado ao resolver 8.8.8.8 |
| `http.request || tls.handshake.type == 1` | Requisições HTTP ou handshakes TLS iniciais (Client Hello) |

Esse tipo de combinação é exatamente o que permite ir de "tenho uma captura enorme" para "tenho as três linhas que realmente importam para essa investigação".

---

# tcpdump — a versão de linha de comando

A sintaxe básica do tcpdump é direta:

```bash
tcpdump -i eth0
```

Algumas opções essenciais:

| Opção | Função |
|---|---|
| `-i` | Especifica a interface de rede |
| `-n` | Não resolve nomes (evita gerar tráfego DNS extra durante a própria captura) |
| `-c` | Limita a quantidade de pacotes capturados |
| `-w arquivo.pcap` | Salva a captura em arquivo, para abrir depois no Wireshark |
| `-r arquivo.pcap` | Lê uma captura salva anteriormente, em vez de capturar ao vivo |

Exemplo combinando filtro (sintaxe BPF, a mesma dos filtros de captura do Wireshark):

```bash
tcpdump -i eth0 port 53 -n
```

Esse comando captura apenas tráfego na porta 53 (DNS), sem resolver nomes — reduzindo ruído desde a captura.

Outro exemplo comum, capturando tráfego de um host específico e salvando em arquivo para análise posterior:

```bash
tcpdump -i eth0 host 203.0.113.50 -w suspeito.pcap
```

Esse arquivo `suspeito.pcap` pode depois ser transferido e aberto no Wireshark, unindo a agilidade do tcpdump em campo com a profundidade de análise visual do Wireshark.

---

# Seguindo uma conversa — "Follow TCP Stream"

Analisar pacotes isolados tem limite. Muitas vezes, o que você quer é reconstruir a **conversa inteira** entre cliente e servidor. O Wireshark tem uma função para isso: clicar com o botão direito em um pacote TCP e escolher **Follow → TCP Stream**.

Isso remonta toda a sequência de dados trocados como uma única visualização — extremamente útil, por exemplo, para identificar credenciais enviadas em texto claro sobre HTTP (lembra do capítulo sobre HTTP e HTTPS?), ou para simplesmente entender, do início ao fim, o que uma sessão suspeita realmente continha.

---

# O que procurar ao analisar uma captura

Juntando tudo que vimos neste módulo até aqui, alguns pontos de atenção práticos:

- **IPs** fora do esperado ou fora de faixas conhecidas — um destino público inesperado partindo de um host que normalmente só conversa com sistemas internos (capítulo 004);
- **portas** incomuns para o tipo de host observado — uma workstation comum recebendo conexões em portas de administração remota, por exemplo (capítulo 005);
- volume ou padrão anormal de **consultas DNS** — muitas consultas rápidas para subdomínios estranhos do mesmo domínio raiz (capítulo 006);
- tráfego **HTTP** (não HTTPS) carregando dados que deveriam ser protegidos, como formulários de login (capítulo 007);
- volume incomum de tráfego **ARP** ou broadcast, que pode indicar reconhecimento de rede ou um ataque em andamento na camada de Enlace (capítulos 002 e 008) — algo que vamos aprofundar no próximo capítulo, sobre ataques comuns.

---

# Aplicação em um SOC

### Ao tratar uma captura como evidência

- uma captura de pacotes é considerada uma das evidências mais confiáveis em uma investigação — diferente de logs, que podem estar incompletos ou ausentes, o pcap registra o tráfego exatamente como ocorreu;
- capturas guardadas (`.pcap`) permitem reanalisar um incidente depois, com outras ferramentas ou outro olhar, inclusive meses após o evento original.

### Ao correlacionar com outras fontes

- correlacionar um alerta do SIEM com a captura de tráfego correspondente ajuda a confirmar ou refutar rapidamente uma hipótese;
- comparar o horário e os IPs de um alerta com o que aparece na captura evita conclusões precipitadas baseadas apenas em um resumo de log.

### Ao documentar uma investigação

- salvar os filtros de exibição usados durante uma análise, junto com a captura, ajuda outra pessoa (ou você mesmo, no futuro) a reproduzir o raciocínio da investigação.

---

# Cenário prático — Domando o caos do início

Voltando à tela lotada do início do capítulo:

```text
Antes do filtro: centenas de pacotes por segundo, ilegível
Depois de aplicar: ip.addr == 203.0.113.50
```

De centenas de linhas, restam apenas as poucas relacionadas ao IP que você realmente queria investigar. O caos nunca esteve nos dados — estava na ausência de um filtro.

Suponha, agora, que entre esses pacotes restantes você note tráfego TCP na porta 4444 — o mesmo número que discutimos lá no capítulo sobre Portas e Protocolos. Um filtro combinado como `ip.addr == 203.0.113.50 && tcp.port == 4444` isola exatamente essa comunicação, e um clique em **Follow TCP Stream** já permite ler a conversa inteira entre sua máquina e aquele destino específico, sem precisar vasculhar pacote por pacote manualmente.

---

## Perguntas de investigação

??? question "1. Qual a principal diferença entre Wireshark e tcpdump?"
    Wireshark é uma ferramenta gráfica voltada para análise visual interativa. tcpdump é uma ferramenta de linha de comando, mais leve, comum em servidores sem interface gráfica — frequentemente usada para capturar e depois analisar no Wireshark.

??? question "2. Qual a diferença entre um filtro de captura e um filtro de exibição?"
    O filtro de captura age antes da captura, descartando permanentemente o que não corresponde. O filtro de exibição age depois, apenas ocultando da visualização — os dados capturados continuam intactos.

??? question "3. Por que uma máquina 'parada' ainda gera tráfego visível em uma captura?"
    Porque processos em segundo plano continuam gerando tráfego: consultas DNS, ARP, atualizações automáticas e telemetria de aplicativos, mesmo sem interação direta do usuário.

??? question "4. Para que serve a função 'Follow TCP Stream' do Wireshark?"
    Para reconstruir a conversa completa entre cliente e servidor como um fluxo único, em vez de analisar pacotes isolados — útil, por exemplo, para identificar dados sensíveis trafegando sem criptografia.

??? question "5. Por que um arquivo de captura (pcap) é considerado uma evidência tão confiável?"
    Porque registra o tráfego exatamente como ele ocorreu na rede, sem depender da geração ou retenção de logs por parte de outros sistemas, que podem estar incompletos ou indisponíveis.

---

# O que não fazer

- não tente analisar uma captura pacote por pacote sem aplicar filtros antes;
- não confunda filtro de captura com filtro de exibição — eles agem em momentos diferentes;
- não descarte o tcpdump como "a versão fraca" do Wireshark — em servidores remotos, muitas vezes é a única opção viável;
- não ignore o painel de detalhes do pacote — é ali que a estrutura de camadas fica visível;
- não descarte filtros combinados por parecerem complexos — geralmente são a diferença entre uma investigação rápida e uma busca manual demorada.

---

# Erros comuns

### "Preciso entender cada pacote da captura, um por um"

Não.

Com o volume de tráfego de uma rede real, isso é inviável. O caminho é filtrar primeiro, depois investigar o que sobrou.

### "Filtro de captura e filtro de exibição são a mesma coisa"

Não.

Um decide o que é gravado; o outro só decide o que é mostrado, sobre dados já gravados.

### "tcpdump é a versão fraca do Wireshark"

Não.

São ferramentas com propósitos diferentes. tcpdump se destaca justamente onde o Wireshark não está disponível: servidores remotos, ambientes sem interface gráfica, automação via script.

---

# Resumo

Neste capítulo, aprendemos que:

- **Wireshark** é uma ferramenta gráfica de análise; **tcpdump** é uma ferramenta de linha de comando, ideal para captura remota;
- uma captura "crua" parece caótica porque tráfego de fundo (DNS, ARP, broadcast) é normal e constante;
- **filtros de captura** decidem o que é gravado; **filtros de exibição** decidem o que é mostrado depois, e podem ser **combinados** para investigações mais precisas;
- a função **Follow TCP Stream** reconstrói uma conversa completa entre cliente e servidor;
- capturas de tráfego (`.pcap`) são consideradas evidência confiável em investigações, por registrarem o tráfego exatamente como ocorreu.

```text
Não pergunte apenas:

"O que esse pacote contém?"

Pergunte também:

"Que filtro reduz esse tráfego ao que realmente importa?"
"Essa é uma conversa que vale reconstruir por completo?"
"Esse padrão bate com o que já vimos nos capítulos anteriores?"
```

> **A ferramenta não organiza o caos por você. Ela só te dá o controle pra fazer isso.**

---

# Checkpoint

Antes de seguir para o próximo capítulo, confirme se você consegue responder:

- [ ] Qual a diferença de propósito entre Wireshark e tcpdump?
- [ ] Por que uma captura de tráfego "parada" ainda mostra atividade?
- [ ] Qual a diferença entre filtro de captura e filtro de exibição?
- [ ] Como escrever e combinar filtros de exibição básicos no Wireshark?
- [ ] Para que serve o Follow TCP Stream?
- [ ] Por que um arquivo pcap é considerado evidência confiável?

---

# Glossário

| Termo | Definição |
|---|---|
| **Wireshark** | Ferramenta gráfica de captura e análise de tráfego de rede. |
| **tcpdump** | Ferramenta de linha de comando para captura de tráfego, nativa na maioria dos sistemas Unix/Linux. |
| **Filtro de captura** | Filtro aplicado antes da captura, descartando permanentemente o que não corresponde. |
| **Filtro de exibição** | Filtro aplicado após a captura, ocultando temporariamente o que não corresponde, sem descartar dados. |
| **PCAP** | Formato de arquivo usado para armazenar capturas de tráfego de rede. |
| **Follow TCP Stream** | Recurso do Wireshark que reconstrói a conversa completa de uma sessão TCP. |

---

# Referências

- [Wireshark — Documentação oficial](https://www.wireshark.org/docs/)
- [tcpdump — Site oficial](https://www.tcpdump.org/)
- [Cisco Networking Academy](https://www.netacad.com/courses/networking)

---

# Próximo capítulo

No último capítulo deste módulo, vamos estudar **Ataques Comuns em Redes** — Sniffing, MITM, ARP Spoofing e Port Scanning — aplicando tudo que construímos até aqui.

[← Capítulo anterior: VPNs e Criptografia em Trânsito](010-vpns-e-criptografia-em-transito.md){ .md-button }

[Próximo: Ataques Comuns em Redes →](012-ataques-comuns-em-redes.md){ .md-button .md-button--primary }

---

> **Entender antes de decorar.**
