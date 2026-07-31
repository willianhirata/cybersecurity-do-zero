<img width="1919" height="873" alt="image" src="https://github.com/user-attachments/assets/b0f4b931-23b8-4b96-a89b-b5576b67ee4b" />---
title: Caso SOC #001 — Investigação de execução do PowerShell
description: Análise prática de logs do Windows utilizando Sysmon, Elastic Defend e Elastic Security para investigar uma execução do PowerShell.
---

# Caso SOC #001 — Investigação de execução do PowerShell

> **Entender antes de decorar.**

---

| Informação | Detalhes |
|---|---|
| **Projeto** | Cybersecurity do Zero |
| **Categoria** | Laboratório SOC |
| **Nível** | Iniciante |
| **Ambiente** | Windows 10, Sysmon, Elastic Agent, Elastic Defend e Elastic Cloud |
| **Objetivo** | Investigar uma execução do PowerShell e determinar se a atividade é legítima ou suspeita |

---

## Introdução

Depois de preparar a máquina virtual, instalar o Elastic Agent, configurar o Elastic Defend e integrar os eventos do Sysmon ao Elastic, chegou o momento de botar a mão na massa e começar a praticar.

Neste primeiro caso, vamos investigar uma execução do **PowerShell** identificada em uma estação Windows.

O PowerShell é uma ferramenta legítima e muito utilizada por administradores. Ao mesmo tempo, também pode ser empregado por atacantes para executar comandos, coletar informações, baixar arquivos e realizar outras ações.

Por isso, encontrar `powershell.exe` nos logs não significa automaticamente que ocorreu um ataque.

> **O processo é apenas o ponto de partida. O contexto é o que permite entender o evento.**

---

## Estrutura do laboratório

O laboratório utilizado nesta análise possui a seguinte estrutura:

```text
Windows 10 — Máquina Virtual
        │
        ├── Sysmon
        │     └── Geração de eventos detalhados do Windows
        │
        ├── Elastic Defend
        │     └── Telemetria e proteção do endpoint
        │
        └── Elastic Agent
              └── Envio dos eventos para o Elastic Cloud
                         │
                         ├── Fleet
                         ├── Elasticsearch
                         ├── Kibana
                         └── Elastic Security
```

O **Sysmon** e o **Elastic Defend** podem registrar informações relacionadas à mesma atividade. Isso não representa necessariamente um erro ou uma duplicação indevida.

Cada fonte possui sua própria forma de observar e estruturar o evento.

---

## Cenário da investigação



Durante o monitoramento do endpoint, foi identificada a execução do PowerShell na máquina do laboratório.

## Imagem 2 — Execução do comando na máquina virtual





![Execução controlada do PowerShell](assets/caso-soc-001/02-execucao-controlada.png)

> **Comentário da análise:**  
> A atividade foi gerada de forma controlada. Como sabemos exatamente qual comando foi executado, podemos verificar se a telemetria coletada representa corretamente o comportamento observado na máquina.

Anote o horário aproximado da execução. Essa informação ajudará a limitar o período da pesquisa no Elastic.

---

## Triagem inicial no Discover

No Kibana, acesse:

```text
Analytics → Discover
```

Ajuste o período para algo próximo ao horário do teste, como:

```text
Last 15 minutes
```

Uma consulta inicial pode procurar pelos processos envolvidos:

```text
event.category: process and
process.name: ("powershell.exe" or "whoami.exe" or "hostname.exe")
```

Adicione, quando disponíveis, as seguintes colunas:

```text
@timestamp
host.name
user.name
process.name
process.parent.name
process.command_line
event.code
data_stream.dataset
```

Esses campos ajudam a responder as perguntas principais da investigação.

---

## Imagem 3 — Resultado da triagem inicial

<!--
Sugestão de arquivo:
assets/caso-soc-001/03-triagem-no-discover.png

Inclua um print do Discover mostrando:
- período selecionado;
- consulta utilizada;
- lista de eventos;
- colunas principais;
- eventos relacionados ao PowerShell.
-->

![Triagem inicial no Elastic Discover](assets/caso-soc-001/03-triagem-no-discover.png)

> **Comentário da análise:**  
> Comecei delimitando o período e os processos envolvidos. Essa triagem reduz o volume de eventos e ajuda a localizar rapidamente a atividade que será investigada.

Neste momento, ainda não devemos concluir que a atividade é suspeita. Primeiro precisamos abrir os eventos e analisar seus detalhes.

---

## Validação do evento de criação do processo

Para visualizar eventos de criação de processo registrados pelo Sysmon, utilize:

```text
data_stream.dataset: "windows.sysmon_operational" and
event.code: "1" and
process.name: "powershell.exe"
```

O **Event ID 1 do Sysmon** representa a criação de um processo.

Ao abrir o evento, procure campos como:

```text
@timestamp
host.name
user.name
process.name
process.executable
process.command_line
process.parent.name
process.parent.executable
process.pid
process.parent.pid
process.hash.sha256
```

Nem todos os campos estarão necessariamente presentes em todas as fontes de dados.

---

## Imagem 4 — Detalhes do evento do PowerShell

<!--
Sugestão de arquivo:
assets/caso-soc-001/04-detalhes-powershell.png

Inclua um print do evento expandido mostrando principalmente:
- process.name;
- process.command_line;
- process.parent.name;
- user.name;
- host.name;
- event.code;
- data_stream.dataset.
-->

![Detalhes do evento de criação do PowerShell](assets/caso-soc-001/04-detalhes-powershell.png)

> **Comentário da análise:**  
> O campo `process.command_line` mostra exatamente como o PowerShell foi iniciado. Já o `process.parent.name` ajuda a entender de onde a execução partiu. Neste caso, o processo pai esperado é o `cmd.exe`, pois o comando foi iniciado manualmente pelo Prompt de Comando.

### Evidências observadas

Preencha esta tabela com os valores encontrados no seu ambiente:

| Evidência | Valor encontrado |
|---|---|
| **Horário** | `PREENCHER` |
| **Host** | `PREENCHER` |
| **Usuário** | `PREENCHER` |
| **Processo** | `powershell.exe` |
| **Processo pai** | `PREENCHER` |
| **Linha de comando** | `PREENCHER` |
| **Event ID** | `1` |
| **Fonte dos dados** | `PREENCHER` |

---

## Análise dos processos relacionados

Depois de identificar o evento principal, precisamos procurar atividades relacionadas.

Pesquise pelos processos executados durante o teste:

```text
event.category: process and
process.name: ("whoami.exe" or "hostname.exe")
```

Também podemos procurar processos cujo pai seja o PowerShell:

```text
process.parent.name: "powershell.exe"
```

A presença de `whoami.exe` e `hostname.exe` pode indicar uma atividade de descoberta.

Essas ferramentas podem ser utilizadas:

- por administradores;
- por scripts legítimos;
- por equipes de suporte;
- por ferramentas de inventário;
- por atacantes tentando entender o ambiente comprometido.

> **O comportamento precisa ser interpretado junto com o usuário, a linha de comando, o processo pai, o horário e as demais evidências.**

---

## Imagem 5 — Processos iniciados e correlação dos eventos

<!--
Sugestão de arquivo:
assets/caso-soc-001/05-processos-relacionados.png

Inclua um print mostrando:
- whoami.exe;
- hostname.exe;
- processo pai;
- horários próximos;
- relação temporal com powershell.exe.

Você também pode incluir uma árvore de processos, caso ela esteja disponível no Elastic Security.
-->

![Processos relacionados à execução do PowerShell](assets/caso-soc-001/05-processos-relacionados.png)

> **Comentário da análise:**  
> Os eventos ocorreram em sequência e estão relacionados pelo processo pai e pelo horário. Essa correlação ajuda a reconstruir o que aconteceu, em vez de analisar cada evento de forma isolada.

---

## Construindo uma linha do tempo

Organize os eventos encontrados em ordem cronológica:

| Ordem | Horário | Processo | Processo pai | Ação observada |
|---:|---|---|---|---|
| 1 | `PREENCHER` | `cmd.exe` | `PREENCHER` | Prompt de Comando aberto |
| 2 | `PREENCHER` | `powershell.exe` | `cmd.exe` | PowerShell iniciado |
| 3 | `PREENCHER` | `whoami.exe` | `powershell.exe` | Identificação do usuário |
| 4 | `PREENCHER` | `hostname.exe` | `powershell.exe` | Identificação da máquina |
| 5 | `PREENCHER` | `powershell.exe` | `cmd.exe` | Consulta dos processos ativos |

Uma linha do tempo ajuda a transformar eventos separados em uma narrativa compreensível.

---

## Imagem 6 — Resumo da investigação

<!--
Sugestão de arquivo:
assets/caso-soc-001/06-resumo-da-investigacao.png

Crie uma imagem final contendo:
- host;
- usuário;
- processo analisado;
- processo pai;
- linha de comando;
- processos relacionados;
- classificação;
- conclusão.
-->

![Resumo da investigação](assets/caso-soc-001/06-resumo-da-investigacao.png)

> **Comentário da análise:**  
> A investigação não encontrou evidências adicionais de comprometimento. A linha de comando, o processo pai, o usuário e o contexto do laboratório indicam que a execução foi legítima e controlada.

---

## Conclusão do analista

Com base nas evidências analisadas:

| Pergunta | Resposta |
|---|---|
| **Quando aconteceu?** | `PREENCHER` |
| **Em qual máquina?** | `PREENCHER` |
| **Qual usuário executou?** | `PREENCHER` |
| **Quem iniciou o PowerShell?** | `PREENCHER` |
| **Qual comando foi executado?** | `PREENCHER` |
| **Quais processos foram iniciados?** | `PREENCHER` |
| **Existem outras evidências suspeitas?** | Não identificadas neste exercício |
| **Classificação** | Atividade legítima de laboratório |

### Parecer

```text
Foi identificada a execução do powershell.exe na estação do laboratório.

A análise da linha de comando, do usuário, do processo pai e dos processos
relacionados demonstrou que a atividade foi iniciada manualmente durante
um teste controlado.

Os comandos executados realizaram consultas básicas de identificação do
usuário, da máquina e dos processos ativos.

Não foram encontradas, neste exercício, evidências adicionais que indiquem
persistência, download de arquivos, comunicação maliciosa ou comprometimento
do endpoint.

Classificação final: atividade legítima de laboratório.
```

---

## O que aprendemos

Neste primeiro caso, aprendemos que:

- um processo legítimo também pode aparecer em atividades maliciosas;
- o nome do processo, sozinho, não permite concluir se houve um ataque;
- a linha de comando revela como o processo foi utilizado;
- o processo pai ajuda a identificar a origem da execução;
- usuário, host e horário fornecem contexto;
- processos filhos ajudam a reconstruir a sequência;
- eventos isolados precisam ser correlacionados;
- uma investigação deve terminar com uma conclusão baseada em evidências.

> **Um alerta inicia a investigação. Ele não determina sozinho o resultado.**

---

## Consultas utilizadas

### Processos envolvidos

```text
event.category: process and
process.name: ("powershell.exe" or "whoami.exe" or "hostname.exe")
```

### PowerShell no Sysmon

```text
data_stream.dataset: "windows.sysmon_operational" and
event.code: "1" and
process.name: "powershell.exe"
```

### Processos iniciados pelo PowerShell

```text
process.parent.name: "powershell.exe"
```

### Consulta mais ampla por linha de comando

```text
process.command_line: *powershell*
```

---

## Checklist das imagens

Antes de publicar, confirme:

- [ ] A imagem 1 apresenta o cenário e o objetivo.
- [ ] A imagem 2 mostra a atividade controlada.
- [ ] A imagem 3 mostra a triagem no Discover.
- [ ] A imagem 4 destaca os campos mais importantes.
- [ ] A imagem 5 mostra os processos relacionados.
- [ ] A imagem 6 apresenta a conclusão.
- [ ] Nenhuma senha, token ou informação pessoal aparece nos prints.
- [ ] Os campos importantes estão legíveis.
- [ ] Os comentários explicam o raciocínio, e não apenas o que existe na tela.
- [ ] A conclusão está baseada nas evidências observadas.

---

## Estrutura sugerida de arquivos

```text
docs/
├── laboratorios/
│   └── caso-soc-001-powershell.md
└── assets/
    └── caso-soc-001/
        ├── 01-apresentacao-do-caso.png
        ├── 02-execucao-controlada.png
        ├── 03-triagem-no-discover.png
        ├── 04-detalhes-powershell.png
        ├── 05-processos-relacionados.png
        └── 06-resumo-da-investigacao.png
```

---

## Próximo caso

No próximo laboratório, podemos investigar uma execução mais incomum do PowerShell e comparar:

- atividade administrativa legítima;
- comportamento suspeito;
- sinais que podem justificar a criação de um alerta;
- evidências necessárias antes de classificar um incidente.

**Entender antes de decorar.**
