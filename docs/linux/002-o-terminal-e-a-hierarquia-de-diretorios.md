---
title: Capítulo 002 — O Terminal e a Hierarquia de Diretórios do Linux
description: Entenda o que é um shell, como ler um prompt de comando, e como funciona a estrutura de diretórios (FHS) do Linux — o mapa básico para não se perder em qualquer sistema Linux.
---

# Capítulo 002 — O Terminal e a Hierarquia de Diretórios do Linux

> **Entender antes de decorar.**

---

| Informação | Detalhes |
|---|---|
| **Módulo** | 03 — Linux |
| **Nível** | Iniciante |
| **Tempo estimado** | 15 a 20 minutos |
| **Pré-requisito** | [Capítulo 001 — Introdução ao Linux e por que ele importa para Segurança](001-introducao-ao-linux-e-por-que-importa-para-seguranca.md) |

---

## Objetivo deste capítulo

Ao final deste capítulo, você será capaz de:

- explicar a diferença entre terminal e shell;
- ler e interpretar as informações mostradas em um prompt de comando;
- reconhecer a estrutura padrão de diretórios do Linux (FHS);
- identificar quais diretórios são mais relevantes para uma investigação de segurança;
- diferenciar caminhos absolutos de caminhos relativos.

---

## Como sempre, vamos começar utilizando nossa imaginação.

Lembra da missão do capítulo anterior? Você conseguiu acesso remoto ao servidor Linux comprometido. Abre a sessão, e a tela mostra:

```text
root@webserver01:/var/www#
```

```text
Opção A
"Vou digitar comandos aleatórios até encontrar alguma coisa suspeita."

Opção B
"Preciso entender o que essa linha está me dizendo antes de sair digitando qualquer coisa."
```

Essa única linha, o **prompt**, já carrega mais informação do que parece. E antes de explorar qualquer coisa nesse servidor, você precisa de um mapa mental de onde as coisas ficam — é exatamente isso que este capítulo constrói.

> **Um terminal Linux não é uma tela em branco. É um mapa que, uma vez entendido, você nunca mais desaprende.**

---

# Terminal x Shell — duas coisas que parecem uma só

Esses dois termos costumam ser usados como sinônimos, mas representam camadas diferentes:

| Termo | O que é |
|---|---|
| **Terminal** | A janela/interface onde você digita e vê o texto — o "aparelho" |
| **Shell** | O programa que interpreta os comandos que você digita — a "inteligência" por trás |

Quando você abre um terminal, ele está rodando um shell por baixo — geralmente o **Bash** (Bourne Again Shell), o mais comum e o padrão na maioria das distribuições, embora existam outros, como o **Zsh** e o **Fish**, com recursos extras de personalização.

---

# Anatomia do prompt

Voltando ao prompt do início do capítulo:

```text
root@webserver01:/var/www#
```

Cada parte carrega uma informação:

```text
root          → usuário atual
@webserver01  → nome do host (servidor)
:/var/www     → diretório atual
#             → indicador de privilégio
```

!!! tip "O símbolo final é um sinal de alerta, não só estética"
    Um `$` no final do prompt indica um usuário comum. Um `#` indica que você está operando como **root** — o superusuário, com acesso irrestrito ao sistema. Isso é tão relevante para investigação quanto para uso diário: um atacante operando como root tem alcance completo sobre o servidor, e é exatamente esse símbolo que revela isso rapidamente durante uma análise.

---

# A Hierarquia de Diretórios do Linux (FHS)

O Linux organiza seus arquivos seguindo um padrão chamado **FHS** (Filesystem Hierarchy Standard), consistente entre a grande maioria das distribuições. Tudo começa em um único diretório raiz, representado por `/`.

```mermaid
flowchart TB
    R["/"] --> ETC[/etc]
    R --> VAR[/var]
    R --> HOME[/home]
    R --> ROOT[/root]
    R --> TMP[/tmp]
    R --> PROC[/proc]
    R --> BIN[/bin, /usr/bin]
```

Alguns diretórios que você vai encontrar (e usar) com muita frequência:

| Diretório | Conteúdo | Por que importa para segurança |
|---|---|---|
| `/etc` | Arquivos de configuração do sistema | Alvo comum de alterações maliciosas para persistência |
| `/var/log` | Logs do sistema e de aplicações | Principal fonte de evidências em uma investigação |
| `/home` | Diretórios pessoais dos usuários | Onde arquivos de usuários comuns costumam ficar |
| `/root` | Diretório pessoal do superusuário (root) | Acesso normalmente restrito, alvo de escalonamento de privilégio |
| `/tmp` | Arquivos temporários | Área comum para "estacionar" payloads maliciosos, por ser gravável por qualquer usuário |
| `/proc` | Representação virtual de processos em execução | Permite inspecionar processos ativos como se fossem arquivos |
| `/bin`, `/usr/bin` | Executáveis e comandos do sistema | Onde ficam os programas que você usa a partir do terminal |

!!! tip "Lembra do capítulo anterior?"
    O diretório `/proc` é a filosofia "tudo é um arquivo" em ação: cada processo em execução no sistema aparece ali como se fosse uma pasta, contendo arquivos que descrevem seu estado — permitindo investigar processos suspeitos usando as mesmas ferramentas que você usaria para ler qualquer outro arquivo de texto.

---

# Caminhos absolutos x relativos

Ao se referir a um arquivo ou diretório, existem duas formas de indicar o caminho:

```text
Caminho absoluto:
/var/log/auth.log
(sempre começa a partir da raiz /, funciona de qualquer lugar)

Caminho relativo:
log/auth.log
(parte de onde você já está no momento)
```

Durante uma investigação, é uma boa prática preferir caminhos **absolutos** sempre que possível — eles eliminam qualquer ambiguidade sobre de qual diretório você está partindo, algo especialmente importante ao documentar evidências que outra pessoa vai revisar depois.

---

# Aplicação em um SOC

### Ao iniciar uma investigação

- verificar o prompt já revela o usuário e o nível de privilégio em uso no momento da sessão;
- ir direto a `/var/log` costuma ser um dos primeiros passos em qualquer triagem inicial.

### Ao procurar indícios de comprometimento

- verificar `/tmp` e diretórios com permissão de escrita ampla em busca de arquivos fora do padrão;
- revisar `/etc` em busca de alterações recentes em arquivos de configuração críticos.

### Ao documentar achados

- sempre registrar caminhos **absolutos** nas anotações de investigação, evitando ambiguidade para quem revisar o relatório depois.

---

# Cenário prático — Voltando ao servidor comprometido

Retomando o prompt do início do capítulo:

```text
root@webserver01:/var/www#
```

Agora você consegue ler essa linha por completo:

> Você está autenticado como **root** — o mais alto nível de privilégio do sistema.

> Está no host **webserver01**.

> Seu diretório atual é `/var/www` — provavelmente onde os arquivos do site hospedado nesse servidor ficam.

Com o mapa de diretórios em mente, os próximos passos óbvios de uma triagem inicial já ficam claros: verificar `/var/log` em busca de atividade suspeita, checar `/tmp` em busca de arquivos fora do lugar, e revisar `/etc` por alterações de configuração recentes — tudo isso antes mesmo de aprender qualquer comando específico, que é o que vamos construir no próximo capítulo.

---

## Perguntas de investigação

??? question "1. Qual a diferença entre terminal e shell?"
    Terminal é a interface onde você digita comandos e vê a saída. Shell é o programa que interpreta esses comandos — o terminal é a "janela", o shell é a "inteligência" por trás dela.

??? question "2. O que o símbolo final do prompt (`$` ou `#`) indica?"
    Indica o nível de privilégio da sessão atual: `$` para um usuário comum, `#` para o superusuário (root), que tem acesso irrestrito ao sistema.

??? question "3. Por que `/var/log` costuma ser um dos primeiros lugares verificados em uma investigação?"
    Porque é onde o sistema e a maioria das aplicações registram seus logs, funcionando como a principal fonte de evidências sobre o que aconteceu no servidor.

??? question "4. Por que `/tmp` é frequentemente mencionado em investigações de malware?"
    Porque é um diretório gravável por qualquer usuário do sistema, tornando-se um local comum para armazenar temporariamente arquivos maliciosos durante um ataque.

??? question "5. Por que usar caminhos absolutos é uma boa prática ao documentar uma investigação?"
    Porque eliminam qualquer ambiguidade sobre a localização de um arquivo, independente de onde a pessoa que lê o relatório esteja posicionada no sistema.

---

# O que não fazer

- não comece a digitar comandos aleatoriamente sem antes entender o prompt e a estrutura de diretórios;
- não ignore o símbolo de privilégio (`$` ou `#`) no prompt — ele muda completamente o contexto de risco de qualquer ação;
- não documente achados usando apenas caminhos relativos, que podem confundir quem revisar depois;
- não trate `/proc` como "só mais uma pasta" — é uma janela direta para processos em execução.

---

# Erros comuns

### "Todo terminal é igual, não importa o shell por trás"

Não completamente.

Embora a maioria dos comandos básicos funcione de forma parecida, diferentes shells (Bash, Zsh, Fish) têm particularidades de sintaxe e recursos que podem gerar comportamento inesperado se você assumir que são idênticos.

### "root é só mais um usuário com nome diferente"

Não.

Root tem acesso irrestrito ao sistema — a diferença de privilégio entre um usuário comum e root é uma das informações mais críticas de qualquer investigação.

### "Não preciso entender a estrutura de diretórios, só preciso saber os comandos"

Não.

Comandos sem contexto de onde as coisas ficam levam a investigações desorganizadas e acham evidências por acidente, não por método.

---

# Resumo

Neste capítulo, aprendemos que:

- **terminal** é a interface; **shell** é o programa que interpreta os comandos digitados;
- o **prompt** revela usuário, host, diretório atual e nível de privilégio em uma única linha;
- o **FHS** organiza o Linux em diretórios padronizados, com destaque para `/etc`, `/var/log`, `/tmp` e `/proc` em contextos de segurança;
- caminhos podem ser **absolutos** (a partir da raiz) ou **relativos** (a partir da posição atual) — absolutos são preferíveis para documentação.

```text
Não pergunte apenas:

"Que comando eu uso agora?"

Pergunte também:

"Em que diretório eu estou, e isso faz sentido?"
"Que nível de privilégio essa sessão tem?"
"Esse é o lugar certo do sistema para eu estar procurando isso?"
```

> **Antes de aprender a andar rápido em um terminal, aprenda a saber onde você está.**

---

# Checkpoint

Antes de seguir para o próximo capítulo, confirme se você consegue responder:

- [ ] Qual a diferença entre terminal e shell?
- [ ] O que cada parte de um prompt como `root@webserver01:/var/www#` representa?
- [ ] O que é o FHS?
- [ ] Quais diretórios merecem atenção prioritária em uma investigação de segurança?
- [ ] Qual a diferença entre caminho absoluto e relativo?

---

# Glossário

| Termo | Definição |
|---|---|
| **Terminal** | Interface onde comandos são digitados e a saída é exibida. |
| **Shell** | Programa que interpreta os comandos digitados no terminal, como o Bash. |
| **Prompt** | Linha exibida pelo shell indicando usuário, host, diretório atual e nível de privilégio. |
| **Root** | Superusuário do sistema Linux, com acesso irrestrito. |
| **FHS** | Filesystem Hierarchy Standard; padrão de organização de diretórios usado pela maioria das distribuições Linux. |
| **Caminho absoluto** | Localização de um arquivo especificada a partir da raiz (`/`). |
| **Caminho relativo** | Localização de um arquivo especificada a partir do diretório atual. |

---

# Referências

- [Filesystem Hierarchy Standard — especificação oficial](https://refspecs.linuxfoundation.org/fhs.shtml)
- [Linux Foundation](https://www.linuxfoundation.org/)

---

# Próximo capítulo

No próximo capítulo, vamos colocar a mão na massa com os **comandos essenciais de navegação** — a base prática para se mover com confiança em qualquer sistema Linux.

[← Capítulo anterior: Introdução ao Linux](001-introducao-ao-linux-e-por-que-importa-para-seguranca.md){ .md-button }

---

> **Entender antes de decorar.**
