---
title: Capítulo 004 — Permissões de Arquivos no Linux
description: Aprenda a ler a notação rwx do ls -l, usar chmod e chown, entender a notação octal, e reconhecer por que arquivos world-writable e bits SUID/SGID são pontos de atenção em qualquer investigação.
---

# Capítulo 004 — Permissões de Arquivos no Linux

> **Entender antes de decorar.**

---

| Informação | Detalhes |
|---|---|
| **Módulo** | 03 — Linux |
| **Nível** | Iniciante |
| **Tempo estimado** | 15 a 20 minutos |
| **Pré-requisito** | [Capítulo 003 — Comandos Essenciais de Navegação e Leitura de Arquivos](003-comandos-essenciais-de-navegacao-e-leitura-de-arquivos.md) |

---

## Objetivo deste capítulo

Ao final deste capítulo, você será capaz de:

- interpretar a notação de permissões (rwx) mostrada em `ls -l`;
- diferenciar permissões de proprietário, grupo e outros;
- entender a diferença entre permissão de execução em um arquivo e em um diretório;
- usar `chmod` em notação simbólica e numérica (octal);
- usar `chown` para alterar proprietário e grupo de um arquivo;
- reconhecer por que arquivos "world-writable" e bits SUID/SGID são pontos de atenção em uma investigação.

---

## Como sempre, vamos começar utilizando nossa imaginação.

Continuando a investigação do servidor comprometido: enquanto navega por `/tmp` (lembra do capítulo 002?), você encontra um arquivo que chama atenção antes mesmo de você abrir o conteúdo:

```text
-rwxrwxrwx 1 root root 45000 Set 20 03:14 update.sh
```

```text
Opção A
"É só um script qualquer, seguindo procurando outra coisa."

Opção B
"Essas permissões, sozinhas, já são um problema."
```

Antes de qualquer análise de conteúdo, essa linha já denuncia algo: **qualquer usuário do sistema pode ler, modificar e executar esse arquivo**. Ao final deste capítulo, você vai entender exatamente por que isso é tão grave.

> **Em uma investigação, permissões contam parte da história antes mesmo de você abrir o arquivo.**

---

# Lendo a notação de permissões

A string de permissões que aparece em `ls -l` tem 10 caracteres, divididos assim:

```text
-    rwx        rwx        rwx
tipo proprietário  grupo      outros
```

O primeiro caractere indica o **tipo**: `-` para arquivo comum, `d` para diretório, `l` para link simbólico. Os nove caracteres seguintes se dividem em três grupos de três, representando três "círculos" de acesso diferentes.

---

# As três permissões: r, w, x

| Permissão | Sigla | Em um arquivo | Em um diretório |
|---|---|---|---|
| Leitura | `r` | Ler o conteúdo do arquivo | Listar o conteúdo do diretório |
| Escrita | `w` | Modificar o arquivo | Criar, renomear ou excluir itens dentro dele |
| Execução | `x` | Executar o arquivo como programa/script | **Entrar** no diretório (navegar até ele) |

!!! tip "A pegadinha mais comum sobre permissão de execução"
    Em um diretório, a permissão `x` **não** significa "executar" o diretório — significa poder **atravessá-lo** com `cd`. Um diretório sem `x` para você, mesmo que tenha `r`, permite listar os nomes dos arquivos dentro dele, mas não permite entrar nem acessar o conteúdo desses arquivos.

---

# Proprietário, grupo e outros — três círculos de acesso

Cada arquivo pertence a um **usuário** (proprietário) e a um **grupo**. As permissões são aplicadas em três círculos concêntricos de abrangência:

```text
Proprietário → a pessoa (ou processo) dono do arquivo
Grupo        → todos os usuários que pertencem ao grupo associado
Outros       → todo o restante do sistema, sem exceção
```

No exemplo do início do capítulo, `-rwxrwxrwx` significa que **todos os três círculos** têm leitura, escrita e execução completas — não existe restrição nenhuma.

---

# Mudando permissões com chmod

### Notação simbólica

```bash
chmod u+x script.sh     # adiciona execução para o proprietário (user)
chmod g-w script.sh     # remove escrita do grupo
chmod o=r script.sh     # define outros como apenas leitura
chmod a+x script.sh     # adiciona execução para todos (all)
```

### Notação numérica (octal)

Cada permissão tem um valor numérico: `r = 4`, `w = 2`, `x = 1`. Somando os valores de cada círculo, chegamos a um número de três dígitos:

| Octal | Significado |
|---|---|
| `755` | proprietário: rwx (7) · grupo: r-x (5) · outros: r-x (5) — comum em scripts executáveis |
| `644` | proprietário: rw- (6) · grupo: r-- (4) · outros: r-- (4) — comum em arquivos de dados |
| `700` | proprietário: rwx (7) · grupo: --- (0) · outros: --- (0) — acesso restrito só ao proprietário |
| `777` | todos com rwx completo — praticamente nunca deveria ser usado deliberadamente |

```bash
chmod 755 script.sh
```

---

# chown — mudando o proprietário

```bash
chown novo_usuario:novo_grupo arquivo.txt
```

Esse comando altera tanto o proprietário quanto o grupo associado a um arquivo — útil, por exemplo, ao restaurar arquivos de um backup que pertenciam a outro usuário, ou ao corrigir permissões após uma investigação.

---

# Por que isso importa tanto para segurança

### Arquivos "world-writable"

Um arquivo com permissão de escrita para "outros" (o terceiro grupo de `rwx`) pode ser modificado por **qualquer usuário** do sistema — inclusive um atacante que tenha conseguido qualquer nível mínimo de acesso. Combinado com permissão de execução, como no `777` do início do capítulo, o risco se torna crítico: qualquer pessoa pode alterar o conteúdo do arquivo e ele ainda vai rodar.

### SUID e SGID — permissões especiais

Existem dois bits especiais que merecem menção, mesmo neste capítulo introdutório: **SUID** (Set User ID) e **SGID** (Set Group ID). Quando aplicados a um executável, fazem com que ele rode com os privilégios do **proprietário do arquivo**, não de quem o executou.

```text
Exemplo: um binário com SUID pertencente a root
Qualquer usuário que o executa → roda temporariamente com privilégios de root
```

Isso tem usos legítimos (o comando `passwd`, por exemplo, precisa desse mecanismo para funcionar), mas é também um dos vetores mais clássicos de **escalonamento de privilégio** quando configurado incorretamente ou presente em um binário que não deveria tê-lo. Analistas frequentemente auditam sistemas em busca de binários SUID inesperados como parte de uma investigação — vamos ver essa técnica com mais profundidade em capítulos futuros deste módulo.

---

# Aplicação em um SOC

### Ao identificar arquivos suspeitos

- prestar atenção especial a arquivos world-writable (`w` habilitado para "outros"), sobretudo em diretórios como `/tmp`;
- um script com permissão `777` em um local temporário é motivo suficiente para investigação imediata, independente do conteúdo.

### Ao auditar permissões críticas

- verificar arquivos de configuração sensíveis (por exemplo, dentro de `/etc`) quanto a permissões excessivamente abertas;
- procurar por binários com SUID/SGID fora do esperado, especialmente em locais não padronizados do sistema.

### Ao documentar uma investigação

- registrar a notação completa de permissões encontrada (não só "estava aberto demais"), permitindo reconstrução exata do que foi observado.

---

# Cenário prático — Decifrando o arquivo em /tmp

Voltando ao arquivo do início do capítulo:

```text
-rwxrwxrwx 1 root root 45000 Set 20 03:14 update.sh
```

> **Tipo:** arquivo comum (`-`).

> **Proprietário (root):** leitura, escrita e execução completas.

> **Grupo:** leitura, escrita e execução completas.

> **Outros:** leitura, escrita e execução completas — ou seja, **qualquer usuário do sistema** pode modificar e rodar esse script.

> **Localização:** `/tmp`, já identificado no capítulo 002 como área comum para estacionar arquivos maliciosos.

> **Nome:** `update.sh` — um nome genérico, do tipo frequentemente escolhido para não chamar atenção em uma varredura superficial.

Somando tudo: um script com permissões totalmente abertas, num diretório clássico de estacionamento temporário, com nome genérico. Nenhum desses fatores prova sozinho que é malicioso — mas juntos, formam exatamente o tipo de padrão que justifica uma investigação de conteúdo imediata.

---

## Perguntas de investigação

??? question "1. O que cada um dos 9 caracteres depois do tipo de arquivo representa em `ls -l`?"
    Três grupos de três caracteres (rwx), representando respectivamente as permissões de leitura, escrita e execução para o proprietário, o grupo e os demais usuários (outros).

??? question "2. Qual a diferença entre a permissão de execução em um arquivo e em um diretório?"
    Em um arquivo, execução permite rodá-lo como programa ou script. Em um diretório, execução permite atravessá-lo (entrar nele via `cd`), não "executá-lo" no sentido tradicional.

??? question "3. Como a notação octal 644 se traduz em permissões rwx?"
    644 equivale a `rw-r--r--`: o proprietário tem leitura e escrita (6 = 4+2), enquanto grupo e outros têm apenas leitura (4 cada).

??? question "4. Por que um arquivo com permissão 777 é considerado um risco de segurança?"
    Porque concede leitura, escrita e execução irrestritas a qualquer usuário do sistema, permitindo que qualquer pessoa modifique o conteúdo do arquivo e ele ainda seja executado normalmente.

??? question "5. O que é o bit SUID, e por que ele é relevante para investigações de escalonamento de privilégio?"
    É uma permissão especial que faz um executável rodar com os privilégios do proprietário do arquivo, não de quem o executa. Se um binário com SUID pertencente a um usuário privilegiado (como root) for mal configurado ou indevido, pode se tornar um caminho para um atacante obter privilégios elevados.

---

# O que não fazer

- não use `chmod 777` como solução genérica para "problemas de permissão" — resolve o erro, mas cria um risco maior;
- não assuma que permissão de execução em um diretório significa "executar" o diretório;
- não ignore bits SUID/SGID durante uma auditoria, especialmente em binários fora de locais padronizados;
- não documente uma investigação com descrições vagas como "permissão estava aberta" — registre a notação exata.

---

# Erros comuns

### "chmod 777 resolve qualquer problema de permissão"

Não.

Resolve o sintoma imediato (erro de acesso), mas abre a porta para qualquer usuário do sistema modificar e executar o arquivo — trocando um problema pequeno por um risco muito maior.

### "Permissão de execução em diretório significa que ele pode ser 'executado'"

Não.

Significa que é possível atravessá-lo (entrar nele). Diretórios não são "executados" no sentido de rodar como programas.

### "SUID é só uma configuração técnica sem relevância de segurança"

Não.

É um dos vetores mais clássicos de escalonamento de privilégio quando presente em binários que não deveriam tê-lo, ou configurado incorretamente.

---

# Resumo

Neste capítulo, aprendemos que:

- a notação de permissões em `ls -l` se divide em tipo + três grupos de **rwx** (proprietário, grupo, outros);
- execução em arquivo significa "rodar"; execução em diretório significa "atravessar";
- **chmod** altera permissões, tanto em notação simbólica quanto numérica (octal);
- **chown** altera proprietário e grupo de um arquivo;
- arquivos **world-writable** (especialmente combinados com execução) são um risco de segurança significativo;
- os bits **SUID** e **SGID** fazem um executável rodar com privilégios do proprietário, sendo um vetor clássico de escalonamento de privilégio quando mal configurados.

```text
Não pergunte apenas:

"Esse arquivo tem permissão de execução?"

Pergunte também:

"Quem, exatamente, tem cada tipo de acesso a esse arquivo?"
"Essa permissão faz sentido para a localização e função desse arquivo?"
"Existe algum bit especial (SUID/SGID) que muda o que esse arquivo pode fazer?"
```

> **Permissão não é só sobre o que um arquivo faz. É sobre quem tem o poder de fazer alguma coisa com ele.**

---

# Checkpoint

Antes de seguir para o próximo capítulo, confirme se você consegue responder:

- [ ] O que os 10 caracteres de permissão em `ls -l` representam?
- [ ] Qual a diferença entre execução em arquivo e em diretório?
- [ ] Como converter permissões rwx para notação octal e vice-versa?
- [ ] Como usar `chmod` e `chown`?
- [ ] Por que um arquivo world-writable é um risco?
- [ ] O que é o bit SUID, e por que ele importa para investigação?

---

# Glossário

| Termo | Definição |
|---|---|
| **Permissão** | Conjunto de regras (leitura, escrita, execução) que definem o que pode ser feito com um arquivo ou diretório. |
| **rwx** | Abreviação para read (leitura), write (escrita) e execute (execução). |
| **chmod** | Comando usado para alterar permissões de um arquivo ou diretório. |
| **chown** | Comando usado para alterar o proprietário e grupo de um arquivo. |
| **Notação octal** | Representação numérica das permissões, somando os valores 4 (r), 2 (w) e 1 (x) por círculo de acesso. |
| **World-writable** | Arquivo ou diretório com permissão de escrita liberada para qualquer usuário do sistema. |
| **SUID / SGID** | Bits especiais que fazem um executável rodar com os privilégios do proprietário ou grupo do arquivo, respectivamente. |

---

# Referências

- [man7.org — Página de manual do chmod](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [GNU Coreutils — Manual oficial](https://www.gnu.org/software/coreutils/manual/coreutils.html)

---

# Próximo capítulo

No próximo capítulo, vamos sair dos arquivos parados e entrar no que está **rodando** no sistema — processos, como listá-los, e como identificar um processo que não deveria estar ali.

[← Capítulo anterior: Comandos Essenciais de Navegação](003-comandos-essenciais-de-navegacao-e-leitura-de-arquivos.md){ .md-button }

---

> **Entender antes de decorar.**
