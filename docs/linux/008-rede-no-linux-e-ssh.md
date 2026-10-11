---
title: Capítulo 008 — Rede no Linux e SSH
description: Aprenda a enxergar a rede de dentro de um servidor Linux com ip e ss, entenda como o SSH funciona e como configurá-lo com segurança, e veja como acessos remotos, tentativas de login e chaves autorizadas aparecem em uma investigação.
---

# Capítulo 008 — Rede no Linux e SSH

> **Entender antes de decorar.**

---

| Informação | Detalhes |
|---|---|
| **Módulo** | 03 — Linux |
| **Nível** | Iniciante |
| **Tempo estimado** | 15 a 20 minutos |
| **Pré-requisito** | [Capítulo 007 — Usuários e Grupos](007-usuarios-e-grupos.md) |

---

## Objetivo deste capítulo

Ao final deste capítulo, você será capaz de:

- consultar a configuração de rede de um servidor Linux com `ip addr` e `ip route`;
- listar portas em escuta e conexões ativas com `ss` e associá-las a processos;
- explicar como o SSH funciona e a diferença entre autenticação por senha e por chave;
- identificar no `sshd_config` as opções que mais impactam a segurança;
- reconhecer nos logs a diferença entre ruído de internet, tentativa de força bruta e uso de credencial válida;
- encontrar a persistência por chaves SSH em `authorized_keys`;
- relacionar o que aprendemos no módulo de Redes com o que se vê de dentro do servidor.

---

## Como sempre, vamos começar utilizando nossa imaginação.

A investigação do servidor avançou bastante.

Você sabe que existe um serviço falso garantindo que o `update.sh` volte a rodar. Sabe que quem o criou entrou pela conta `deploy`, via SSH, a partir do IP `203.0.113.58`, às 02h09 da madrugada, e que essa sessão deu a si mesma `sudo` sem senha.

O que você ainda não sabe é como essa pessoa chegou até a porta de entrada.

```text
Opção A
"O SSH estava aberto porque sempre foi assim. Alguém descobriu a senha, azar.
Vou trocar a senha do deploy e seguir."

Opção B
"Preciso entender o que o servidor expõe para a rede, como o SSH está configurado
e o que mais essa sessão deixou para trás antes de encerrar a investigação."
```

Trocar a senha encerra uma porta. Mas se essa sessão deixou outra chave embaixo do tapete, o invasor volta sem precisar de senha nenhuma.

> **Servidor comprometido não é só sobre o que foi feito dentro dele. É também sobre por onde se entra e se continua entrando.**

---

# Enxergando a rede de dentro do servidor

No módulo de Redes, você aprendeu endereços IP, máscaras, portas e rotas olhando "de fora", pelo ponto de vista de um pacote. Agora o ponto de vista muda: você está **dentro** da máquina e precisa descobrir como ela se enxerga na rede.

## ip: interfaces, endereços e rotas

O comando `ip` substituiu os antigos `ifconfig` e `route`, que ainda aparecem em tutoriais, mas estão desatualizados em muitas distribuições.

| Comando | O que mostra |
|---|---|
| `ip addr` (ou `ip a`) | Interfaces de rede e seus endereços IP |
| `ip route` (ou `ip r`) | Tabela de rotas, incluindo o gateway padrão |
| `ip link` | Estado das interfaces (ligadas ou desligadas, endereço MAC) |

Um exemplo:

```text
$ ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 52:54:00:3a:91:c2 brd ff:ff:ff:ff:ff:ff
    inet 10.0.20.15/24 brd 10.0.20.255 scope global eth0

$ ip route
default via 10.0.20.1 dev eth0
10.0.20.0/24 dev eth0 proto kernel scope link src 10.0.20.15
```

Tudo isso é conteúdo que você já conhece: o servidor tem o IP privado `10.0.20.15`, está na rede `10.0.20.0/24` (veja o capítulo 004 de Redes) e usa `10.0.20.1` como gateway (capítulo 009 de Redes). A diferença é que agora você enxerga isso no próprio sistema.

!!! tip "Por que isso importa em uma investigação"
    Saber o IP, a sub-rede e o gateway do servidor permite responder perguntas como: esse IP de origem está dentro ou fora da rede da empresa? Esse tráfego deveria passar pelo firewall? O servidor tem mais de uma interface (o que indicaria um caminho entre duas redes)? Um host com duas interfaces é um ponto de passagem em potencial.

## ss: portas em escuta e conexões ativas

Se o `ip` mostra como o servidor está **conectado**, o `ss` mostra o que ele está **fazendo** na rede. Ele lista sockets: portas aguardando conexões e conexões em andamento. Substituiu o antigo `netstat`.

```bash
ss -tulpn
```

| Flag | Significado |
|---|---|
| `-t` | Conexões TCP |
| `-u` | Conexões UDP |
| `-l` | Apenas sockets em escuta (*listening*) |
| `-p` | Mostra o processo dono do socket (precisa de root para ver todos) |
| `-n` | Mostra números (IPs e portas) em vez de resolver nomes |

Saída típica:

```text
$ sudo ss -tulpn
Netid State  Local Address:Port  Peer Address:Port  Process
tcp   LISTEN 0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=812,fd=3))
tcp   LISTEN 0.0.0.0:80          0.0.0.0:*          users:(("nginx",pid=1382,fd=6))
tcp   LISTEN 127.0.0.1:3306      0.0.0.0:*          users:(("mysqld",pid=905,fd=21))
```

Leia esse resultado como uma lista de **portas de entrada**:

- `0.0.0.0:22`: o SSH aceita conexões em **todas** as interfaces, ou seja, de qualquer origem que consiga alcançar o servidor;
- `0.0.0.0:80`: o nginx está disponível para todos, o que é esperado em um servidor web;
- `127.0.0.1:3306`: o MySQL escuta **apenas localmente**, e portanto não é acessível pela rede. É assim que deve ser para um banco de dados.

O endereço onde um serviço escuta é uma decisão de segurança. `0.0.0.0` significa "qualquer interface", enquanto `127.0.0.1` significa "só esta máquina".

Para ver as conexões estabelecidas:

```bash
ss -tnp state established
```

Ali aparecem as sessões em curso: quem está conectado ao servidor e para onde o próprio servidor está se conectando. Conexões de **saída** feitas por processos estranhos são exatamente o tipo de comportamento (Comando e Controle) que discutimos no capítulo 001 de Redes.

---

# SSH: o acesso remoto padrão do Linux

O **SSH** (Secure Shell) é o protocolo usado para administrar servidores Linux remotamente. Ele abre um terminal em outra máquina por um canal **criptografado**, por padrão na porta **TCP 22**. Em termos de Redes, é um serviço de camada de aplicação com criptografia em trânsito, o mesmo princípio que você viu em VPNs e TLS.

Ele substituiu protocolos antigos como o Telnet, que enviavam tudo em texto puro. Isso resolve o risco de alguém no meio do caminho ler o que trafega (sniffing, como no capítulo 012 de Redes). Mas **não resolve** um problema diferente: se alguém tem credenciais válidas, o canal criptografado protege o atacante tanto quanto protege você.

## Dois jeitos de se autenticar

| Método | Como funciona | Pontos fortes | Pontos fracos |
|---|---|---|---|
| **Senha** | O usuário digita uma senha, que o servidor valida | Simples de usar | Pode ser adivinhada, reaproveitada de outro vazamento ou capturada em phishing |
| **Chave pública** | O usuário possui um par de chaves. A chave **pública** fica no servidor, a **privada** fica só com ele | Não há senha para adivinhar ou reutilizar | Quem rouba a chave privada ganha acesso |

Na autenticação por chave, o servidor guarda as chaves públicas autorizadas em um arquivo dentro da home de cada usuário:

```text
/home/deploy/.ssh/authorized_keys
```

Cada linha desse arquivo é uma chave que **tem permissão de entrar como aquele usuário**, sem pedir senha. Guarde isso, porque esse arquivo vai voltar mais tarde na investigação.

## Configurando o SSH com segurança: sshd_config

O comportamento do servidor SSH é definido em `/etc/ssh/sshd_config`. As opções que mais importam para segurança:

| Opção | Valor mais seguro | O que controla |
|---|---|---|
| `PermitRootLogin` | `no` | Se o root pode entrar diretamente por SSH |
| `PasswordAuthentication` | `no` | Se login por senha é aceito (forçando o uso de chaves) |
| `PubkeyAuthentication` | `yes` | Se login por chave pública é aceito |
| `AllowUsers` | lista de contas | Restringe quais usuários podem usar SSH |
| `Port` | variável | Porta em que o SSH escuta (mudar a porta reduz ruído, mas **não é uma proteção real**) |

Para ver o que está de fato valendo, incluindo valores padrão que não aparecem no arquivo:

```bash
sudo sshd -T | grep -E "permitrootlogin|passwordauthentication|allowusers"
```

!!! tip "Mudar a porta do SSH não é segurança"
    Trocar a porta de 22 para outra reduz o volume de tentativas automatizadas nos logs, e isso é um benefício real de organização. Mas qualquer varredura de portas (como no capítulo 012 de Redes) encontra o serviço em segundos. A proteção de verdade vem de **desligar senhas, restringir usuários e limitar a exposição à rede**.

---

# O que a internet faz com uma porta 22 aberta

Qualquer servidor com SSH exposto à internet recebe tentativas de login automatizadas o tempo todo, de IPs do mundo inteiro, testando usuários e senhas comuns. Isso é ruído de fundo, e quem analisa logs aprende a conviver com ele.

No `auth.log` (capítulo 007), as linhas típicas são:

```text
sshd[2210]: Invalid user admin from 192.0.2.44 port 40022
sshd[2210]: Failed password for invalid user admin from 192.0.2.44 port 40022 ssh2
sshd[2231]: Failed password for root from 198.51.100.7 port 40112 ssh2
```

A pergunta que separa ruído de incidente não é "houve falhas?" (sempre há). É:

| Padrão no log | O que costuma indicar |
|---|---|
| Muitas falhas, usuários variados (`admin`, `ubuntu`, `test`), vindas de IPs diferentes | Varredura automatizada da internet. Ruído de fundo |
| Muitas falhas **contra uma mesma conta real**, de um mesmo IP | Tentativa de adivinhação de senha (força bruta) |
| Falhas seguidas de **um** `Accepted` para a mesma conta | Força bruta que funcionou |
| **Um único `Accepted`**, de um IP desconhecido, sem falhas antes | Credencial já conhecida. Foi obtida em outro lugar |

Essas distinções importam porque apontam para **causas diferentes**, e portanto para respostas diferentes. Força bruta se resolve com senha forte e bloqueio de tentativas. Credencial já conhecida significa que o problema está em outro lugar: vazamento, reaproveitamento de senha ou phishing (capítulo 012 de Fundamentos).

## Limitando a exposição

Algumas medidas simples, que combinam com o que você já viu:

- **Desativar login por senha** e exigir chaves;
- **Firewall** limitando o SSH a IPs conhecidos (como as regras do capítulo 009 de Redes e do Lab-004). Em muitas distribuições, ferramentas como `ufw` ou `firewalld` facilitam isso;
- **Bloqueio automático** após várias falhas, com ferramentas como o `fail2ban`, que lê os logs e bloqueia o IP temporariamente;
- **Autenticação multifator** ou acesso apenas via VPN (capítulo 010 de Redes).

---

# Aplicação em um SOC

### Ao avaliar a superfície exposta

- usar `ss -tulpn` para listar as portas em escuta e confirmar se cada uma tem justificativa;
- conferir o endereço de escuta: um serviço interno em `0.0.0.0` merece questionamento;
- comparar o resultado com o que o firewall deveria estar permitindo.

### Ao investigar acessos SSH

- no `auth.log` / `secure`, procurar `Accepted` e checar IP de origem, horário e conta;
- diferenciar ruído de força bruta e de uso de credencial válida antes de concluir qualquer coisa;
- consultar o `sshd_config` para entender quais caminhos de entrada estavam abertos.

### Ao caçar persistência pós-acesso

- inspecionar `authorized_keys` de todas as contas com acesso interativo;
- verificar a data de modificação desses arquivos e comparar com a linha do tempo da intrusão;
- olhar conexões de saída (`ss -tnp state established`) em busca de processos que falam com a internet sem razão.

!!! tip "Mais técnicas catalogadas no MITRE ATT&CK"
    Este capítulo toca em três técnicas conhecidas: [External Remote Services (T1133)](https://attack.mitre.org/techniques/T1133/), que é o uso de serviços de acesso remoto expostos como porta de entrada; [Brute Force (T1110)](https://attack.mitre.org/techniques/T1110/), que é a tentativa de adivinhar credenciais; e [SSH Authorized Keys (T1098.004)](https://attack.mitre.org/techniques/T1098/004/), que é a adição de chaves para manter acesso. Reconhecer onde cada uma deixa rastro é o que transforma o framework em ferramenta de investigação.

---

# Cenário prático — Como o invasor chegou e o que deixou

Primeiro, o que o servidor expõe hoje:

```bash
$ sudo ss -tulpn | grep sshd
tcp   LISTEN 0.0.0.0:22    0.0.0.0:*    users:(("sshd",pid=812,fd=3))
tcp   LISTEN [::]:22       [::]:*       users:(("sshd",pid=812,fd=4))

$ sudo sshd -T | grep -E "permitrootlogin|passwordauthentication"
permitrootlogin no
passwordauthentication yes
```

O SSH está aberto para o mundo, em IPv4 e IPv6, e **aceita senha**. O `permitrootlogin no` é uma boa notícia: o invasor teve que usar uma conta comum e depois escalar via sudo, que é o que vimos nos logs. Esse foi o primeiro achado de configuração.

Agora, a pergunta central: houve força bruta contra a conta `deploy`?

```bash
$ grep "Failed password" /var/log/auth.log | grep "deploy" | wc -l
0

$ grep "203.0.113.58" /var/log/auth.log
Sep 24 02:09:31 srv01 sshd[3190]: Accepted password for deploy from 203.0.113.58 port 51422 ssh2
```

Zero falhas para a conta `deploy`, e o IP `203.0.113.58` aparece **uma única vez**, justamente no login bem-sucedido. Pelo quadro da seção anterior, isso não se parece com força bruta. Se alguém tivesse adivinhado a senha, haveria um rastro de tentativas antes. Em vez disso, houve um único login, correto de primeira.

Existe ruído no log, como em qualquer servidor exposto:

```bash
$ grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -3
  412 198.51.100.7
  187 192.0.2.44
   96 192.0.2.91
```

Esses IPs tentaram usuários genéricos (`root`, `admin`, `ubuntu`), nunca `deploy`, e não têm nenhum `Accepted`. É a varredura de fundo da internet, que não teve sucesso. Foi só barulho.

A conclusão parcial é importante: a senha do `deploy` **já era conhecida** antes de chegar ao servidor. Pode ter sido reutilizada de outro serviço que vazou, capturada em um phishing ou obtida de alguma outra forma. Os logs deste servidor sozinhos não respondem como, mas mostram o que **não** aconteceu.

Antes de encerrar, a pergunta da abertura: o que mais essa sessão deixou? Olhando para a chave SSH:

```bash
$ ls -l /home/deploy/.ssh/
-rw------- 1 deploy deploy  398 Sep 24 02:19 authorized_keys

$ cat /home/deploy/.ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ops-maintenance@laptop
```

O arquivo foi modificado às **02h19**, no meio da sessão do invasor (02h09 a 02h31), e contém uma chave com um comentário genérico e convincente (`ops-maintenance@laptop`). Ninguém na equipe reconhece essa chave. Esse comando não aparece nos registros de `sudo`, porque o usuário `deploy` pode escrever na própria home sem privilégios elevados.

Isso muda tudo. Trocar a senha do `deploy` **não resolveria nada**: o invasor continuaria entrando pela chave, sem senha e sem deixar falhas no log.

Por fim, uma última verificação de comportamento: o que o servidor está contactando agora?

```bash
$ sudo ss -tnp state established | grep -v sshd
ESTAB 10.0.20.15:49418  198.51.100.23:443  users:(("update.sh",pid=4521,fd=3))
```

O `update.sh` (PID 4521, o mesmo processo do capítulo 005) mantém uma conexão de saída com `198.51.100.23`, na porta 443. Tráfego em 443 parece HTTPS comum, e é por isso que é um destino frequente de Comando e Controle. Esse é o ponto onde o capítulo 001 de Redes volta à cena.

O quadro da invasão agora tem quatro peças:

```text
1. Entrada:       credencial válida do deploy, via SSH com senha habilitada
2. Elevação:      sudo NOPASSWD criado pela própria sessão
3. Persistência:  serviço systemd falso + chave SSH adicionada
4. Comunicação:   update.sh conectado a um IP externo na porta 443
```

---

## Perguntas de investigação

??? question "1. Qual a diferença entre `ip` e `ss`?"
    O `ip` mostra a configuração de rede da máquina: interfaces, endereços e rotas. O `ss` mostra a atividade na rede: portas em escuta e conexões em andamento, com o processo responsável por cada uma.

??? question "2. O que significa um serviço escutando em `0.0.0.0` versus `127.0.0.1`?"
    `0.0.0.0` significa que o serviço aceita conexões em todas as interfaces, ou seja, de qualquer origem que alcance o servidor. `127.0.0.1` significa que só aceita conexões da própria máquina. Serviços internos, como bancos de dados, normalmente devem escutar apenas localmente.

??? question "3. Por que mudar a porta do SSH não é uma medida de segurança real?"
    Porque uma varredura de portas encontra o serviço independentemente do número. Mudar a porta reduz o ruído nos logs, mas não impede um atacante que procure ativamente. A proteção real vem de desativar senhas, restringir usuários e limitar quem alcança a porta.

??? question "4. Como diferenciar ruído de internet de um ataque real nos logs do SSH?"
    Pelo padrão: muitas falhas com usuários genéricos vindas de vários IPs são varredura automatizada. Falhas repetidas contra uma conta real a partir de um IP indicam tentativa de adivinhação. Um único login correto, sem falhas antes, sugere que o atacante já tinha a credencial.

??? question "5. Por que o arquivo `authorized_keys` é um alvo de persistência?"
    Porque qualquer chave listada ali permite login como aquele usuário, sem senha. Se o invasor adiciona a própria chave, mantém o acesso mesmo depois de a senha ser trocada, e as entradas no log aparecem como "Accepted publickey", sem a marca de tentativas falhas.

??? question "6. Por que a adição da chave não apareceu nos logs do sudo?"
    Porque o usuário `deploy` tem permissão de escrita na própria pasta `.ssh`. Para editar um arquivo da própria home, não é preciso elevar privilégios, então nenhum comando `sudo` foi registrado. Essa é uma lição importante: o `sudo` log não conta a história completa.

---

# O que não fazer

- não trate a troca de senha como resposta suficiente a um acesso indevido, sem antes revisar `authorized_keys` e outros pontos de persistência;
- não conclua que "houve força bruta" só porque há falhas no log, já que todo servidor exposto as recebe;
- não confie na ideia de que mudar a porta protege o SSH;
- não deixe `PasswordAuthentication yes` e `PermitRootLogin yes` ativos em servidores expostos sem uma razão forte;
- não edite o `sshd_config` e reinicie o serviço sem manter uma sessão aberta para testar, pois um erro pode te trancar fora do servidor;
- não desconecte ou isole o servidor sem antes registrar conexões, processos e logs, para preservar evidências.

---

# Erros comuns

### "SSH é criptografado, então é seguro"

Não.

A criptografia protege o conteúdo contra quem observa o tráfego. Ela não impede o login de quem tem uma credencial válida, e não distingue o administrador do invasor.

### "Se não houve força bruta, a conta não foi comprometida"

Não.

Um único login correto, sem falhas antes, é justamente o padrão de quem já possuía a credencial. A ausência de tentativas falhas não é prova de legitimidade.

### "Basta trocar a senha da conta comprometida"

Não.

Se houve acesso, é preciso verificar chaves em `authorized_keys`, serviços, tarefas agendadas e regras de `sudo`. Trocar apenas a senha fecha uma porta e deixa outras abertas.

### "O ss e o ip só mostram o estado atual, não servem para investigação"

Não.

Mostram uma fotografia do momento, mas ela vale muito: portas abertas inesperadas, conexões de saída para IPs estranhos e processos associados a elas são evidências diretas, e devem ser registradas antes de qualquer contenção.

### "Muitas falhas de login significam que fomos atacados com sucesso"

Não.

Falhas mostram tentativas, não sucesso. O que determina um incidente é o que aconteceu depois do primeiro `Accepted`.

---

# Resumo

Neste capítulo, aprendemos que:

- `ip addr` e `ip route` mostram como o servidor está configurado na rede, e `ss -tulpn` mostra quais portas estão em escuta e quais processos as mantêm;
- o endereço de escuta (`0.0.0.0` x `127.0.0.1`) é uma decisão de segurança sobre quem pode alcançar o serviço;
- o **SSH** oferece acesso remoto criptografado, com autenticação por senha ou por chave pública;
- as opções do `sshd_config` (`PermitRootLogin`, `PasswordAuthentication`, `AllowUsers`) definem boa parte da segurança do acesso remoto;
- é preciso diferenciar **ruído de internet**, **força bruta** e **uso de credencial válida** pelos padrões no `auth.log`;
- o arquivo `authorized_keys` é um ponto clássico de **persistência**, e sua alteração não passa pelo `sudo`;
- conexões de saída de processos suspeitos (`ss -tnp`) conectam a investigação no servidor ao conceito de Comando e Controle do módulo de Redes.

```text
Não pergunte apenas:

"Como o invasor entrou?"

Pergunte também:

"O que mais essa sessão deixou para trás?"
"Que outras portas de entrada existem além da que eu encontrei?"
"O servidor está falando com quem, agora?"
```

> **Fechar a porta por onde ele entrou não adianta se ele já deixou uma cópia da chave.**

---

# Checkpoint

Antes de seguir para o próximo capítulo, confirme se você consegue responder:

- [ ] Para que servem `ip addr` e `ip route`?
- [ ] O que `ss -tulpn` mostra e o que cada flag significa?
- [ ] Qual a diferença entre um serviço escutando em `0.0.0.0` e em `127.0.0.1`?
- [ ] Como o SSH autentica por senha e por chave pública?
- [ ] Quais opções do `sshd_config` mais afetam a segurança?
- [ ] Como diferenciar ruído de internet, força bruta e uso de credencial válida?
- [ ] Por que `authorized_keys` é um ponto de persistência?
- [ ] Por que trocar a senha não basta depois de um acesso indevido?
- [ ] Como encontrar conexões de saída suspeitas?

---

# Glossário

| Termo | Definição |
|---|---|
| **SSH** | Protocolo para acesso remoto seguro a sistemas, com comunicação criptografada, normalmente na porta TCP 22. |
| **sshd** | Serviço (daemon) servidor do SSH, que aceita as conexões. |
| **Socket** | Ponto de comunicação de rede definido por um endereço IP e uma porta. |
| **Listening** | Estado de uma porta que está aguardando conexões. |
| **Chave pública / privada** | Par de chaves usado na autenticação. A pública é compartilhada, a privada é mantida em segredo. |
| **authorized_keys** | Arquivo que lista as chaves públicas autorizadas a entrar como determinado usuário. |
| **Força bruta** | Tentativa repetida de adivinhar credenciais testando muitas combinações. |
| **Gateway padrão** | Roteador para onde o servidor envia o tráfego destinado a redes que ele não conhece. |
| **fail2ban** | Ferramenta que lê logs e bloqueia temporariamente IPs com muitas falhas de autenticação. |

---

# Referências

- [man7.org — Página de manual do ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [man7.org — Página de manual do ip](https://man7.org/linux/man-pages/man8/ip.8.html)
- [OpenBSD — Página de manual do sshd_config](https://man.openbsd.org/sshd_config)
- [OpenSSH — Site oficial](https://www.openssh.com/)
- [MITRE ATT&CK — External Remote Services (T1133)](https://attack.mitre.org/techniques/T1133/)
- [MITRE ATT&CK — Brute Force (T1110)](https://attack.mitre.org/techniques/T1110/)
- [MITRE ATT&CK — SSH Authorized Keys (T1098.004)](https://attack.mitre.org/techniques/T1098/004/)

---

# Próximo capítulo

Até aqui, encontramos três mecanismos de continuidade deixados pelo invasor: um serviço systemd, uma regra de sudo e uma chave SSH. Mas existe outro mecanismo clássico que ainda não verificamos, e é um dos mais usados porque é discreto e antigo: as tarefas agendadas. No próximo capítulo, vamos estudar o **cron e o agendamento de tarefas**, onde ele guarda suas configurações, e como fechar o checklist de persistência de um servidor Linux.

[← Capítulo anterior: Usuários e Grupos](007-usuarios-e-grupos.md){ .md-button }

---

> **Entender antes de decorar.**
