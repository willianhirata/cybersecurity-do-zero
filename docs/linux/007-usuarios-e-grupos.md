---
title: Capítulo 007 — Usuários e Grupos
description: Entenda como o Linux representa identidades com usuários, grupos e UIDs, onde essas informações ficam armazenadas, como o sudo concede privilégios e como contas e permissões viram pistas em uma investigação.
---

# Capítulo 007 — Usuários e Grupos

> **Entender antes de decorar.**

---

| Informação | Detalhes |
|---|---|
| **Módulo** | 03 — Linux |
| **Nível** | Iniciante |
| **Tempo estimado** | 15 a 20 minutos |
| **Pré-requisito** | [Capítulo 006 — Serviços e o systemd](006-servicos-e-o-systemd.md) |

---

## Objetivo deste capítulo

Ao final deste capítulo, você será capaz de:

- explicar o que é um usuário no Linux e por que todo processo e todo arquivo possuem um dono;
- diferenciar root, contas de sistema e contas de usuários comuns usando o UID;
- ler e interpretar os campos de `/etc/passwd`, `/etc/shadow` e `/etc/group`;
- entender o papel dos grupos e identificar grupos que concedem poder além do que parece;
- diferenciar `su`, `sudo` e login direto como root, e saber onde o sudo é configurado;
- usar `id`, `who`, `last` e os logs de autenticação para reconstruir quem fez o quê;
- reconhecer contas e privilégios criados por atacantes como técnica de persistência.

---

## Como sempre, vamos começar utilizando nossa imaginação.

Voltando ao servidor que você vem investigando.

No capítulo anterior, você encontrou o serviço `system-update-checker.service`, o responsável por fazer o `update.sh` voltar a rodar depois de cada reboot. A persistência foi explicada.

Mas fica uma pergunta que ainda não foi respondida.

Para criar um arquivo em `/etc/systemd/system/` e ainda executar `systemctl enable`, é preciso ter privilégio de administrador. O processo do nginx, que gerou o `update.sh`, roda com uma conta comum de serviço.

```text
Opção A
"O arquivo é do root. Então foi o administrador que criou, deve ser legítimo."

Opção B
"Para esse arquivo existir, alguém com privilégio administrativo agiu.
Preciso descobrir quem, de onde, e quando."
```

Dizer que "foi o root" não responde nada. Em sistemas Linux, o root é uma conta que muitas pessoas e processos podem alcançar de formas diferentes. A investigação agora deixa de ser sobre **o que está rodando** e passa a ser sobre **quem está por trás**.

> **Todo processo roda em nome de alguém. Todo arquivo pertence a alguém. Descobrir quem é esse alguém é metade de qualquer investigação.**

---

# Usuários: a identidade que o sistema enxerga

Para o Linux, um **usuário** não é necessariamente uma pessoa. É uma **identidade** à qual o sistema associa arquivos, processos e permissões.

Internamente, o sistema não trabalha com nomes. Ele trabalha com números. Cada usuário possui um **UID** (User ID), e cada grupo possui um **GID** (Group ID). O nome `willian` é apenas um rótulo amigável que o sistema traduz para o número correspondente.

Esse é o mesmo mecanismo que você viu no capítulo 004, quando `ls -l` mostrou dono e grupo de cada arquivo, e no capítulo 005, quando `ps aux` mostrou a coluna USER de cada processo. Agora vamos entender de onde essas informações vêm.

## Os três tipos de conta

| Tipo | UID típico | Para que serve | Exemplo |
|---|---|---|---|
| **root** | `0` | Administrador total do sistema. Ignora praticamente todas as restrições de permissão | `root` |
| **Contas de sistema / serviço** | geralmente abaixo de `1000` | Isolar serviços (cada serviço roda com a menor permissão possível) | `www-data`, `sshd`, `mysql` |
| **Contas de usuário comum** | geralmente a partir de `1000` | Pessoas que usam o sistema | `willian`, `deploy` |

!!! tip "O número, não o nome, é o que manda"
    O que torna uma conta "root" não é se chamar `root`. É ter **UID 0**. Uma conta com outro nome, mas com UID 0, tem exatamente o mesmo poder. É por isso que uma das primeiras checagens em uma investigação é procurar por contas com UID 0 além do `root` verdadeiro. A faixa de UIDs (abaixo ou a partir de 1000) é uma convenção da maioria das distribuições modernas, mas pode variar um pouco entre sistemas.

## Por que contas de serviço existem

Lembra do nginx rodando como `www-data`? Isso é intencional. Se o servidor web rodasse como root e fosse comprometido, o invasor herdaria o poder total do sistema. Rodando como `www-data`, o estrago inicial fica limitado ao que aquela conta consegue ler e escrever.

```text
Servidor web como root:
comprometeu o nginx → comprometeu o sistema inteiro

Servidor web como www-data:
comprometeu o nginx → ainda precisa de um segundo passo para chegar no root
```

Essa ideia tem nome: **princípio do menor privilégio**. Cada identidade deve ter apenas o acesso estritamente necessário para fazer seu trabalho. Quando você vê um processo ou usuário com mais poder do que sua função justifica, isso por si só já é um achado.

---

# Onde o Linux guarda as identidades

Usuários e grupos não ficam em um banco de dados escondido. Ficam em arquivos de texto simples dentro de `/etc`, o mesmo diretório de configurações que vimos no capítulo 002. Isso é ótimo para investigação: tudo é legível e auditável.

## /etc/passwd — a lista de contas

Apesar do nome, esse arquivo **não guarda senhas** (por razões históricas, o nome ficou). Ele lista as contas existentes, e cada linha possui sete campos separados por `:`.

```text
deploy:x:1001:1001:Deploy User:/home/deploy:/bin/bash
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
```

| Posição | Exemplo | Significado |
|---|---|---|
| 1 | `deploy` | Nome do usuário |
| 2 | `x` | Indica que o hash da senha está em `/etc/shadow` |
| 3 | `1001` | UID |
| 4 | `1001` | GID do grupo primário |
| 5 | `Deploy User` | Comentário / nome completo |
| 6 | `/home/deploy` | Diretório home |
| 7 | `/bin/bash` | Shell de login |

Repare no último campo. A conta `www-data` usa `/usr/sbin/nologin`, o que significa que ninguém consegue abrir uma sessão interativa com ela. A conta `deploy` usa `/bin/bash`, então pode abrir um terminal. Uma conta de serviço com shell interativo é algo que merece atenção.

Para procurar contas com poder de root além do esperado:

```bash
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

O resultado esperado é apenas `root`. Qualquer outro nome na lista é um achado importante.

## /etc/shadow — onde ficam os hashes

É aqui que ficam as senhas, mas nunca em texto puro. O arquivo guarda o **hash** da senha: o resultado de uma transformação matemática de mão única.

```text
deploy:$6$Xk3...(hash longo)...:20357:0:99999:7:::
www-data:*:19800:0:99999:7:::
```

O segundo campo conta uma história importante:

| Valor | Significado |
|---|---|
| Um hash longo (começa com `$`) | A conta possui senha definida |
| `*` ou `!` | A conta não aceita login por senha (bloqueada ou sem senha) |

Lembra do capítulo 004? Esse arquivo precisa ser protegido por permissões. Por padrão, só o root (e, em algumas distribuições, um grupo específico, como o `shadow`) consegue lê-lo:

```bash
ls -l /etc/passwd /etc/shadow
-rw-r--r-- 1 root root   1842 Sep 20 09:12 /etc/passwd
-rw-r----- 1 root shadow  1127 Sep 20 09:12 /etc/shadow
```

O `/etc/passwd` precisa ser legível por todos porque programas comuns usam esse arquivo para traduzir UIDs em nomes. Já o `/etc/shadow` guarda segredos, por isso a permissão é restrita. Se o `/etc/shadow` aparecer legível por qualquer usuário, é uma falha séria de configuração.

## /etc/group — os grupos e seus membros

Cada linha define um grupo, seu GID e a lista de membros adicionais:

```text
sudo:x:27:willian,deploy
www-data:x:33:
```

---

# Grupos: permissões compartilhadas

Um **grupo** é uma forma de dar a vários usuários as mesmas permissões sem precisar configurar um por um. Todo usuário possui um **grupo primário** (definido em `/etc/passwd`) e pode participar de vários **grupos secundários** (definidos em `/etc/group`).

Quando você roda `ls -l` e vê:

```text
-rw-r----- 1 root adm 4096 Sep 28 06:00 auth.log
```

o terceiro campo (`root`) é o dono e o quarto (`adm`) é o grupo. Qualquer membro do grupo `adm` consegue ler esse log, conforme as permissões que você aprendeu no capítulo 004.

## Alguns grupos que merecem respeito

| Grupo | O que costuma permitir | Por que importa |
|---|---|---|
| `sudo` (Debian/Ubuntu) ou `wheel` (RHEL e derivados) | Executar comandos como root via `sudo` | Quem entra nesse grupo vira, na prática, administrador |
| `adm` | Ler logs do sistema | Acesso a evidências sensíveis |
| `shadow` | Ler o `/etc/shadow` | Acesso aos hashes de senha |
| `docker` | Controlar o serviço Docker | Equivale, na prática, a ter acesso root, pois é possível iniciar contêineres com acesso ao sistema de arquivos do host |

!!! tip "Nem todo caminho para o root passa pela palavra 'root'"
    O grupo `docker` é o exemplo clássico. Uma conta que não aparece em `sudo` e não tem UID 0 ainda pode ter poder equivalente ao do administrador. Ao auditar privilégios, olhar apenas quem está em `sudo` não é suficiente. É preciso entender o que cada grupo realmente permite.

---

# root, su e sudo: três formas de ter poder

Existem três caminhos comuns para executar algo com privilégios administrativos:

| Forma | Como funciona | Observação |
|---|---|---|
| **Login direto como root** | Entrar no sistema já como `root` | Menos rastreável. Todos entram como a mesma identidade. Muitos sistemas bloqueiam isso por padrão em acesso remoto |
| **`su`** | Troca para outra conta (por padrão, root) usando a **senha da conta de destino** | Exige conhecer a senha do root |
| **`sudo`** | Executa **um comando** com os privilégios de outra conta (por padrão, root), usando a **senha do próprio usuário** | Registra quem executou o quê |

A grande vantagem do `sudo` é a **rastreabilidade**. Com o `sudo`, cada comando fica registrado com o nome real de quem o executou. Com um login direto como root, tudo aparece como "root", e não dá para saber qual pessoa estava por trás.

## Quem pode usar o sudo e como

As regras ficam em `/etc/sudoers` e, em muitos sistemas, também em arquivos dentro de `/etc/sudoers.d/`. Esse arquivo deve ser editado com `visudo`, que valida a sintaxe antes de salvar (um erro nesse arquivo pode trancar todos os administradores fora do sistema).

```text
# Membros do grupo sudo podem executar qualquer comando (pedindo senha)
%sudo   ALL=(ALL:ALL) ALL

# Esta linha dispensa a senha para o usuário deploy
deploy  ALL=(ALL) NOPASSWD:ALL
```

A opção `NOPASSWD` é a que mais chama atenção. Ela permite que a conta execute comandos como root **sem digitar senha**. Existem usos legítimos (automação), mas é também um excelente alvo para quem quer manter acesso privilegiado silencioso.

Para ver o que o seu usuário pode fazer:

```bash
sudo -l
```

---

# Comandos para investigar identidades

| Comando | O que mostra |
|---|---|
| `whoami` | Qual usuário está executando a sessão atual |
| `id` | UID, GID e todos os grupos do usuário (use `id deploy` para consultar outra conta) |
| `groups` | Lista de grupos do usuário |
| `who` / `w` | Quem está logado **agora** e de onde |
| `last` | Histórico de logins, com origem e duração (lê `/var/log/wtmp`) |
| `lastb` | Tentativas de login que falharam (requer root, lê `/var/log/btmp`) |
| `getent passwd` | Lista contas consultando todas as fontes configuradas (inclusive diretórios corporativos) |

Um exemplo de saída do `id`:

```text
$ id deploy
uid=1001(deploy) gid=1001(deploy) groups=1001(deploy),27(sudo)
```

Em uma linha, você vê a identidade (UID 1001), o grupo primário e que essa conta pertence ao grupo `sudo`, ou seja, consegue virar administrador.

---

# Logs de autenticação: a memória do sistema sobre quem entrou

O Linux registra eventos de autenticação e uso de sudo em arquivos de log. O nome muda conforme a distribuição:

| Distribuição | Arquivo |
|---|---|
| Debian / Ubuntu | `/var/log/auth.log` |
| RHEL / CentOS / Fedora | `/var/log/secure` |

Ou, em sistemas com systemd, consultando o journal que você viu no capítulo anterior (`journalctl`). As linhas mais úteis seguem alguns padrões:

```text
sshd[3190]: Accepted password for deploy from 203.0.113.58 port 51422 ssh2
sshd[3201]: Failed password for root from 198.51.100.7 port 40112 ssh2
sudo: deploy : TTY=pts/0 ; PWD=/home/deploy ; USER=root ; COMMAND=/usr/bin/apt update
useradd[4410]: new user: name=helpdesk, UID=1002, GID=1002
```

Cada uma conta uma história diferente: um login que funcionou, uma tentativa que falhou, um comando executado com privilégio, e uma conta nova sendo criada. O campo `COMMAND=` do `sudo` é particularmente valioso, porque mostra exatamente o que foi executado como root.

---

# Aplicação em um SOC

### Ao investigar acessos

- verificar `who` / `w` para ver sessões ativas e `last` para o histórico recente;
- cruzar origem dos logins (IP) com o que é esperado para aquela conta;
- observar horários fora do padrão de uso daquele usuário.

### Ao auditar privilégios

- procurar contas com UID 0 além do `root`;
- revisar membros dos grupos `sudo`/`wheel` e `docker`;
- inspecionar `/etc/sudoers` e `/etc/sudoers.d/` à procura de `NOPASSWD` ou regras amplas demais;
- verificar contas de serviço com shell interativo (`/bin/bash` em vez de `nologin`).

### Ao reconstruir a linha do tempo

- usar `auth.log` / `secure` para ordenar login, uso de sudo e criação de contas;
- comparar a data de modificação de arquivos sensíveis (`ls -l`, `stat`) com os eventos registrados nos logs.

!!! tip "Isso também tem nome no MITRE ATT&CK"
    Criar contas novas ou aproveitar contas já existentes são técnicas documentadas no framework. As principais são [Create Account (T1136)](https://attack.mitre.org/techniques/T1136/) e [Valid Accounts (T1078)](https://attack.mitre.org/techniques/T1078/). A segunda é especialmente importante: muitas intrusões não usam nenhuma falha técnica sofisticada, apenas **credenciais legítimas** que foram obtidas ou reaproveitadas. Para o sistema, parece um usuário normal fazendo login.

---

# Cenário prático — Descobrindo quem criou o serviço

Voltando à pergunta do início. Primeiro, quem é o dono do arquivo do serviço e quando ele foi criado:

```bash
$ ls -l /etc/systemd/system/system-update-checker.service
-rw-r--r-- 1 root root 164 Sep 24 02:14 /etc/systemd/system/system-update-checker.service
```

O dono é `root`, como esperado, mas a data importa: **24 de setembro, 02h14 da madrugada**. Esse é o ponto de partida da linha do tempo. Agora, o que o log de autenticação diz nesse intervalo?

```bash
$ grep "Sep 24 02:" /var/log/auth.log
Sep 24 02:09:31 srv01 sshd[3190]: Accepted password for deploy from 203.0.113.58 port 51422 ssh2
Sep 24 02:12:05 srv01 sudo: deploy : TTY=pts/0 ; PWD=/home/deploy ; USER=root ; COMMAND=/usr/bin/tee /etc/sudoers.d/90-deploy
Sep 24 02:14:40 srv01 sudo: deploy : TTY=pts/0 ; PWD=/home/deploy ; USER=root ; COMMAND=/usr/bin/tee /etc/systemd/system/system-update-checker.service
Sep 24 02:15:02 srv01 sudo: deploy : TTY=pts/0 ; PWD=/home/deploy ; USER=root ; COMMAND=/usr/bin/systemctl enable system-update-checker.service
```

A sequência fala sozinha:

1. 02h09: alguém fez login via SSH com a conta `deploy`, **a partir de um IP externo**;
2. 02h12: essa sessão usou sudo para escrever em `/etc/sudoers.d/90-deploy`;
3. 02h14: usou sudo para criar o unit file do serviço falso;
4. 02h15: usou sudo para habilitar o serviço no boot.

Confirmando o histórico de sessões e as permissões da conta:

```bash
$ last deploy
deploy   pts/0   203.0.113.58   Thu Sep 24 02:09 - 02:31  (00:22)

$ id deploy
uid=1001(deploy) gid=1001(deploy) groups=1001(deploy),27(sudo)

$ cat /etc/sudoers.d/90-deploy
deploy ALL=(ALL) NOPASSWD:ALL

$ awk -F: '$3 == 0 {print $1}' /etc/passwd
root
```

Dois achados importantes saem daqui. O primeiro é tranquilizador: **não existe uma segunda conta com UID 0**, então o invasor não criou um "root disfarçado". O segundo é preocupante: a conta `deploy` já era um usuário legítimo, mas ganhou **privilégio sudo sem senha** durante essa mesma sessão de 22 minutos, e isso foi feito de madrugada, a partir de um IP que não pertence à empresa.

> Esse é o padrão típico de abuso de conta válida: nenhuma conta nova foi criada, nenhuma falha foi explorada nesse trecho. Alguém simplesmente entrou com uma credencial real.

A pergunta mudou de lugar. Já sabemos **o que** foi feito, **quando**, e **com qual conta**. O que ainda não sabemos é **como o invasor conseguiu a senha da conta `deploy`**, e por que o SSH aceitou um login vindo de fora. Essa resposta está do lado da rede.

---

## Perguntas de investigação

??? question "1. Por que o sistema trabalha com UID e não com o nome do usuário?"
    Porque o nome é apenas um rótulo para humanos. Arquivos e processos guardam o número (UID/GID), e o sistema traduz esse número para um nome consultando `/etc/passwd`. Isso também explica por que uma conta com UID 0, mesmo com outro nome, tem poder de root.

??? question "2. O `/etc/passwd` guarda senhas?"
    Não, apesar do nome. Ele lista as contas e seus atributos. O `x` no segundo campo indica que o hash da senha está em `/etc/shadow`, que é protegido por permissões mais restritas.

??? question "3. Qual a vantagem do sudo sobre fazer login direto como root?"
    Rastreabilidade. O `sudo` registra qual usuário executou qual comando, enquanto o login direto como root mistura todas as ações sob a mesma identidade. O sudo também permite conceder poder limitado, em vez de acesso total.

??? question "4. Por que `NOPASSWD` em uma regra de sudo merece atenção?"
    Porque remove a barreira da senha. Qualquer pessoa ou processo que controle aquela conta passa a executar comandos como root sem nenhuma confirmação adicional. Pode ser legítimo em automações, mas precisa ser justificado e documentado.

??? question "5. Estar fora do grupo sudo significa não ter poder administrativo?"
    Não necessariamente. Grupos como `docker` podem conceder, na prática, acesso equivalente ao de root. Auditar privilégios exige entender o que cada grupo realmente permite.

??? question "6. Qual a diferença entre `last` e os logs de `auth.log`?"
    O `last` mostra um resumo de sessões de login (quem, de onde, por quanto tempo). O `auth.log` / `secure` registra eventos de autenticação com mais detalhe, incluindo tentativas falhas, comandos executados via sudo e criação de contas. Um complementa o outro.

---

# O que não fazer

- não conclua que algo é legítimo só porque o arquivo pertence ao `root`;
- não olhe apenas o grupo `sudo` ao auditar privilégios, e esqueça `docker` e outros grupos poderosos;
- não edite `/etc/sudoers` diretamente com um editor comum, use `visudo`;
- não exponha ou copie o conteúdo de `/etc/shadow` para ambientes inseguros, mesmo durante uma investigação;
- não apague uma conta suspeita antes de coletar evidências (logs, histórico, arquivos da home), pois você destrói a linha do tempo;
- não trate "o login funcionou" como prova de que a pessoa certa entrou.

---

# Erros comuns

### "O arquivo é do root, então foi o administrador que fez"

Não.

Ser dono `root` só indica o nível de privilégio com que o arquivo foi criado, não **quem** estava por trás da ação. Qualquer pessoa que consiga executar comandos como root, via sudo, por exemplo, gera arquivos do root.

### "Se não existe usuário novo, então a conta não foi comprometida"

Não.

O caso deste capítulo mostra o contrário. Nenhuma conta foi criada, mas uma conta legítima foi usada indevidamente. Comprometimento de credenciais não deixa um `useradd` nos logs.

### "O `/etc/passwd` guarda as senhas dos usuários"

Não.

Ele guarda a lista de contas. O hash das senhas fica no `/etc/shadow`, justamente por isso ter permissões mais restritas.

### "Só o root tem poder de administrador"

Não.

Qualquer conta com UID 0, membros de grupos como `sudo`, `wheel` ou `docker`, e contas com regras amplas no `sudoers` podem ter poder equivalente. O que importa é o **poder efetivo**, não o nome da conta.

### "Se o log mostra o usuário deploy, foi o responsável pelo deploy"

Não.

O log registra a **conta** usada, não a **pessoa** por trás. Se a credencial foi roubada ou reaproveitada, o log vai mostrar o nome do usuário legítimo. Por isso o IP de origem, o horário e o comportamento da sessão importam tanto quanto o nome.

---

# Resumo

Neste capítulo, aprendemos que:

- todo processo e todo arquivo possuem um **dono**, representado internamente por um **UID** (e um **GID** para grupos);
- **UID 0** é o que define o poder de root, independentemente do nome da conta;
- contas de serviço (como `www-data`) existem para aplicar o **princípio do menor privilégio**;
- `/etc/passwd` lista as contas, `/etc/shadow` guarda os hashes (com acesso restrito) e `/etc/group` define os grupos e seus membros;
- grupos como `sudo`/`wheel` e `docker` concedem poder administrativo, direta ou indiretamente;
- o `sudo` oferece **rastreabilidade** e é configurado em `/etc/sudoers` e `/etc/sudoers.d/`, com atenção especial a `NOPASSWD`;
- `id`, `who`, `last` e os logs de autenticação (`auth.log` ou `secure`) permitem reconstruir quem fez o quê e quando;
- abusar de **contas válidas** é uma técnica comum e não deixa o rastro óbvio de uma conta nova.

```text
Não pergunte apenas:

"Quem é o dono desse arquivo ou processo?"

Pergunte também:

"Quem estava por trás dessa conta naquele momento?"
"De onde veio essa sessão e isso é esperado?"
"Esse privilégio foi concedido quando, e por quem?"
```

> **Em Linux, quase tudo deixa um dono. Investigar é seguir o dono até chegar em quem realmente agiu.**

---

# Checkpoint

Antes de seguir para o próximo capítulo, confirme se você consegue responder:

- [ ] O que é um UID e por que o UID 0 é especial?
- [ ] Qual a diferença entre root, conta de sistema e conta de usuário comum?
- [ ] O que cada arquivo guarda: `/etc/passwd`, `/etc/shadow` e `/etc/group`?
- [ ] Como procurar contas com UID 0 além do root?
- [ ] Qual a diferença entre `su` e `sudo`, e por que o sudo é mais rastreável?
- [ ] Por que `NOPASSWD` e o grupo `docker` merecem atenção?
- [ ] Onde ficam os logs de autenticação em Debian/Ubuntu e em RHEL?
- [ ] Como usar `last` e `auth.log` para reconstruir uma linha do tempo de acesso?
- [ ] Por que o uso de credenciais válidas é difícil de distinguir de atividade legítima?

---

# Glossário

| Termo | Definição |
|---|---|
| **UID** | Número que identifica um usuário no sistema. O UID 0 corresponde ao root. |
| **GID** | Número que identifica um grupo no sistema. |
| **root** | Conta de administrador total do sistema (UID 0). |
| **Conta de serviço** | Conta criada para rodar um serviço específico, geralmente sem login interativo e com poucos privilégios. |
| **Princípio do menor privilégio** | Princípio de segurança que defende dar a cada identidade apenas o acesso estritamente necessário. |
| **Hash de senha** | Resultado de uma transformação de mão única aplicada à senha, armazenado no lugar do texto original. |
| **sudo** | Comando que executa um comando com os privilégios de outra conta (por padrão, root), registrando quem o executou. |
| **sudoers** | Arquivo (e diretório `sudoers.d`) que define quem pode usar sudo e com quais permissões. |
| **NOPASSWD** | Opção do sudoers que dispensa a senha ao executar comandos privilegiados. |
| **Conta válida (Valid Accounts)** | Técnica em que um atacante usa credenciais legítimas, obtidas de alguma forma, em vez de explorar uma falha técnica. |

---

# Referências

- [man7.org — Página de manual do passwd (formato do arquivo)](https://man7.org/linux/man-pages/man5/passwd.5.html)
- [man7.org — Página de manual do shadow](https://man7.org/linux/man-pages/man5/shadow.5.html)
- [man7.org — Página de manual do sudoers](https://man7.org/linux/man-pages/man5/sudoers.5.html)
- [man7.org — Página de manual do last](https://man7.org/linux/man-pages/man1/last.1.html)
- [MITRE ATT&CK — Create Account (T1136)](https://attack.mitre.org/techniques/T1136/)
- [MITRE ATT&CK — Valid Accounts (T1078)](https://attack.mitre.org/techniques/T1078/)

---

# Próximo capítulo

Sabemos que o invasor entrou com a conta `deploy`, via SSH, a partir de um IP externo. Mas como esse acesso remoto funciona, o que o SSH registra, e que informações de rede o Linux nos oferece para ver quem está conectado ao servidor neste exato momento? No próximo capítulo, vamos estudar **rede no Linux e SSH** — `ip`, `ss`, conexões ativas, o serviço `sshd` e como ligar o que aprendemos no módulo de Redes à investigação dentro do servidor.

[← Capítulo anterior: Serviços e o systemd](006-servicos-e-o-systemd.md){ .md-button }

---

> **Entender antes de decorar.**
