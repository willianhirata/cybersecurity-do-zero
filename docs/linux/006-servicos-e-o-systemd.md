---
title: Capítulo 006 — Serviços e o systemd
description: Entenda a diferença entre um processo pontual e um serviço de longa duração, como o systemd gerencia isso, e como serviços maliciosos são usados como técnica de persistência.
---

# Capítulo 006 — Serviços e o systemd

> **Entender antes de decorar.**

---

| Informação | Detalhes |
|---|---|
| **Módulo** | 03 — Linux |
| **Nível** | Iniciante |
| **Tempo estimado** | 15 a 20 minutos |
| **Pré-requisito** | [Capítulo 005 — Processos: o que está rodando no sistema](005-processos-o-que-esta-rodando-no-sistema.md) |

---

## Objetivo deste capítulo

Ao final deste capítulo, você será capaz de:

- diferenciar um processo pontual de um serviço de longa duração;
- entender o papel do systemd como sistema de inicialização moderno;
- usar `systemctl` para verificar, iniciar, parar, habilitar e desabilitar serviços;
- diferenciar iniciar um serviço agora (`start`) de configurá-lo para iniciar automaticamente no boot (`enable`);
- usar `journalctl` para investigar logs de um serviço específico;
- reconhecer como serviços maliciosos são usados como técnica de persistência.

---

## Como sempre, vamos começar utilizando nossa imaginação.

O servidor foi reiniciado ontem à noite — uma manutenção de rotina. Hoje de manhã, você verifica novamente: o `update.sh`, aquele script suspeito filho do nginx que você já vinha investigando, **está rodando de novo**.

```text
Opção A
"Deve ser coincidência. Alguém deve ter executado ele manualmente de novo."

Opção B
"Alguém configurou algo para que ele rode automaticamente, mesmo depois de um reboot."
```

Processos comuns não sobrevivem a um reboot — eles simplesmente somem quando o sistema desliga. Para algo voltar a rodar sozinho depois de reiniciar, precisa haver um mecanismo de inicialização automática por trás. É exatamente isso que vamos investigar neste capítulo.

> **Se algo malicioso sobrevive a um reboot, alguém configurou deliberadamente para isso acontecer.**

---

# Processos pontuais x serviços de longa duração

Nem todo processo é igual em propósito. Um comando como `ls` roda, termina, e desaparece em frações de segundo. Já programas como `nginx` ou `sshd` são pensados para rodar **continuamente**, o tempo todo, sobrevivendo entre reinicializações — esses são chamados de **serviços** (ou, historicamente, **daemons** — muitos, por convenção, têm nomes terminados em "d").

```text
Processo pontual:  executa uma tarefa e termina (ex: ls, cat, um script rodado manualmente)
Serviço/daemon:    roda continuamente, reiniciado automaticamente se cair, ativo desde o boot
```

---

# O que é o systemd?

O **systemd** é o sistema de inicialização (init system) usado pela maioria das distribuições Linux modernas. É o processo **PID 1** — o primeiro a iniciar quando o sistema liga, responsável por inicializar e gerenciar todos os demais serviços a partir daí.

Cada serviço gerenciado pelo systemd é descrito por um **unit file** — um arquivo de configuração (geralmente com extensão `.service`) que define como aquele serviço deve ser iniciado, reiniciado em caso de falha, e em que ordem em relação a outros serviços.

---

# systemctl — controlando serviços

| Comando | O que faz |
|---|---|
| `systemctl status nginx` | Mostra o status atual do serviço |
| `systemctl start nginx` | Inicia o serviço agora |
| `systemctl stop nginx` | Para o serviço agora |
| `systemctl restart nginx` | Reinicia o serviço |
| `systemctl enable nginx` | Configura o serviço para iniciar automaticamente no próximo boot |
| `systemctl disable nginx` | Remove essa inicialização automática |

!!! tip "start/stop não é o mesmo que enable/disable"
    Essa é a confusão mais comum entre iniciantes: `start` e `stop` afetam o **agora** — se o serviço está rodando neste exato momento. `enable` e `disable` afetam o **futuro** — se ele vai iniciar sozinho na próxima vez que o sistema ligar. Um serviço pode estar rodando agora (`active`) sem estar habilitado para o boot (`enabled`), e vice-versa.

---

# Interpretando a saída do systemctl status

```text
● nginx.service - A high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
     Active: active (running) since Mon 2026-09-28 06:00:12 UTC
   Main PID: 1382 (nginx)
      Tasks: 3
     Memory: 4.2M
     CGroup: /system.slice/nginx.service
             ├─1382 nginx: master process
             └─4521 /tmp/update.sh
```

Repare no último detalhe: o **CGroup** mostra a árvore de processos daquele serviço — e ali está o `update.sh` (PID 4521, o mesmo do capítulo anterior), listado como parte do grupo de controle do serviço nginx. Essa é mais uma confirmação de que ele está sendo gerado como filho direto do serviço web.

---

# journalctl — os logs centralizados do systemd

O systemd mantém seus próprios registros, acessíveis via `journalctl`:

```bash
journalctl -u nginx              # logs específicos do serviço nginx
journalctl -u nginx -f           # acompanha em tempo real (como o tail -f do capítulo 003)
journalctl -u nginx --since "1 hour ago"   # logs da última hora
```

Isso complementa (não substitui) os arquivos tradicionais em `/var/log` — alguns serviços registram exclusivamente no journal, outros também mantêm arquivos próprios.

---

# Persistência via serviços maliciosos

Voltando à pergunta do início do capítulo: como o `update.sh` voltou a rodar sozinho depois do reboot? Uma das explicações mais prováveis é que alguém criou um **serviço systemd customizado**, configurado com `enable`, para iniciar esse script automaticamente toda vez que o sistema liga.

```bash
systemctl list-unit-files --state=enabled
```

Esse comando lista todos os serviços habilitados para iniciar no boot — e um nome genérico ou disfarçado nessa lista (algo como `network-helper.service` ou `system-update.service`) merece investigação, especialmente se apontar para um script fora dos locais padronizados do sistema, como `/tmp`.

!!! tip "Isso tem nome no MITRE ATT&CK"
    Essa técnica — criar ou modificar um serviço do sistema para garantir persistência — é catalogada como uma técnica específica dentro do framework MITRE ATT&CK, que já mencionamos no módulo de Redes. Reconhecer esse padrão não é só intuição: é uma técnica documentada e amplamente observada em investigações reais.

Serviços customizados costumam ter seus unit files em `/etc/systemd/system/` — mais um diretório, além dos já vistos no capítulo 002, que merece atenção em qualquer auditoria de persistência.

---

# Aplicação em um SOC

### Ao investigar persistência

- listar serviços habilitados (`systemctl list-unit-files --state=enabled`) e comparar com o que é esperado naquele servidor;
- verificar unit files recém-criados ou modificados em `/etc/systemd/system/`.

### Ao correlacionar com processos

- usar `systemctl status` para confirmar se um PID suspeito, identificado no capítulo anterior, pertence a algum serviço registrado — e se esse serviço faz sentido existir ali.

### Ao revisar logs

- usar `journalctl -u <serviço> --since` para reconstruir a linha do tempo de quando um serviço específico começou a se comportar de forma anômala.

---

# Cenário prático — Resolvendo a persistência

Investigando a lista de serviços habilitados:

```bash
$ systemctl list-unit-files --state=enabled | grep -i update
system-update-checker.service    enabled
```

```bash
$ cat /etc/systemd/system/system-update-checker.service
[Unit]
Description=System Update Checker

[Service]
ExecStart=/tmp/update.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

> Um serviço com nome genérico e convincente ("System Update Checker") foi criado especificamente para rodar `/tmp/update.sh`.

> `Restart=always` garante que, mesmo que o script seja encerrado manualmente, o systemd o reinicia automaticamente.

> `WantedBy=multi-user.target` é o que garante que esse serviço inicia todas as vezes que o sistema liga — explicando exatamente por que o script reapareceu depois do reboot.

Esse é um exemplo típico de persistência via serviço systemd — e agora você sabe exatamente onde procurar e como reconhecer.

---

## Perguntas de investigação

??? question "1. Qual a diferença entre iniciar um serviço com start e habilitá-lo com enable?"
    `start` inicia o serviço imediatamente, na sessão atual. `enable` configura o serviço para iniciar automaticamente sempre que o sistema for ligado — são independentes um do outro.

??? question "2. O que o systemd faz como PID 1?"
    É o primeiro processo iniciado pelo sistema, responsável por inicializar e gerenciar todos os demais serviços a partir daí, seguindo as definições dos unit files.

??? question "3. Por que um atacante criaria um serviço systemd customizado como técnica de persistência?"
    Porque isso garante que seu script ou ferramenta continue rodando automaticamente mesmo após o sistema ser reiniciado, sem precisar de execução manual repetida.

??? question "4. Qual comando mostra os logs específicos de um único serviço?"
    `journalctl -u nome_do_servico`, que filtra os registros do journal apenas para aquele serviço específico.

??? question "5. Onde costumam ficar os unit files de serviços customizados?"
    Em `/etc/systemd/system/`, diferente dos unit files padrão do sistema, que ficam em `/lib/systemd/system/` (ou `/usr/lib/systemd/system/`, dependendo da distribuição).

---

# O que não fazer

- não assuma que um serviço `active` (rodando agora) está necessariamente `enabled` (configurado pra iniciar no boot), ou vice-versa;
- não trate `journalctl` como redundante em relação a `/var/log` — alguns serviços registram exclusivamente ali;
- não ignore `/etc/systemd/system/` numa auditoria de persistência;
- não descarte um nome de serviço "convincente" sem verificar o que ele realmente executa.

---

# Erros comuns

### "Se o serviço está rodando, ele com certeza está habilitado para o boot"

Não.

São configurações independentes. Um serviço pode ter sido iniciado manualmente (`start`) sem nunca ter sido habilitado (`enable`) para sobreviver a um reboot — e o inverso também é possível.

### "journalctl é só mais um jeito de ver os mesmos logs de /var/log"

Não.

O journal do systemd é um sistema de log próprio, e alguns serviços registram exclusivamente ali, sem gerar arquivo correspondente em `/var/log`.

### "Só preciso verificar processos, não preciso verificar serviços"

Não.

Um processo malicioso pode desaparecer com um reboot — mas se ele estiver amarrado a um serviço habilitado, ele volta sozinho, e só investigar processos, sem olhar para os serviços por trás, deixa passar a causa raiz da persistência.

---

# Resumo

Neste capítulo, aprendemos que:

- **serviços** (ou daemons) são processos de longa duração, diferente de processos pontuais que terminam rapidamente;
- o **systemd**, como PID 1, gerencia a inicialização e o ciclo de vida desses serviços através de **unit files**;
- `systemctl start/stop` afeta o agora; `systemctl enable/disable` afeta o comportamento no próximo boot;
- `journalctl` fornece logs centralizados, específicos por serviço, incluindo acompanhamento em tempo real;
- criar um serviço systemd customizado é uma técnica documentada de **persistência**, permitindo que um script malicioso sobreviva a reinicializações;
- unit files customizados costumam ficar em `/etc/systemd/system/`.

```text
Não pergunte apenas:

"Esse processo está rodando agora?"

Pergunte também:

"Existe algum serviço garantindo que ele volte a rodar sozinho?"
"Esse serviço está habilitado para o boot, e isso faz sentido?"
"O nome desse serviço é convincente demais para ser coincidência?"
```

> **Matar o processo resolve o agora. Encontrar o serviço por trás dele resolve a causa raiz.**

---

# Checkpoint

Antes de seguir para o próximo capítulo, confirme se você consegue responder:

- [ ] Qual a diferença entre um processo pontual e um serviço?
- [ ] Qual o papel do systemd como PID 1?
- [ ] Qual a diferença entre `start`/`stop` e `enable`/`disable`?
- [ ] Como consultar logs de um serviço específico com `journalctl`?
- [ ] Como um serviço systemd customizado pode ser usado para persistência?
- [ ] Onde costumam ficar os unit files de serviços customizados?

---

# Glossário

| Termo | Definição |
|---|---|
| **Serviço (daemon)** | Processo de longa duração, projetado para rodar continuamente. |
| **systemd** | Sistema de inicialização moderno usado pela maioria das distribuições Linux, rodando como PID 1. |
| **Unit file** | Arquivo de configuração que descreve como um serviço deve ser gerenciado pelo systemd. |
| **systemctl** | Comando usado para controlar serviços gerenciados pelo systemd. |
| **journalctl** | Comando usado para consultar os logs centralizados do systemd. |
| **Persistência** | Técnica usada para garantir que um acesso ou processo malicioso sobreviva a reinicializações do sistema. |

---

# Referências

- [man7.org — Página de manual do systemctl](https://man7.org/linux/man-pages/man1/systemctl.1.html)
- [man7.org — Página de manual do journalctl](https://man7.org/linux/man-pages/man1/journalctl.1.html)
- [man7.org — Página de manual do systemd.service](https://man7.org/linux/man-pages/man5/systemd.service.5.html)

---

# Próximo capítulo

No próximo capítulo, vamos entender **usuários e grupos** — como o Linux gerencia identidades, onde essas informações ficam armazenadas, e o papel do `sudo` na concessão de privilégios.

[← Capítulo anterior: Processos](005-processos-o-que-esta-rodando-no-sistema.md){ .md-button }

---

> **Entender antes de decorar.**
