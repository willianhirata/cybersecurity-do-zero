---
title: LAB 003 — Simulação e Investigação de Phishing
description: Simulação controlada de phishing com link, download inofensivo e análise dos eventos no Elastic.
---

# LAB 003 — Simulação e Investigação de Phishing

> **Entender antes de decorar.**

| Informação | Detalhes |
|---|---|
| **Categoria** | Laboratório |
| **Nível** | Iniciante |
| **Tempo estimado** | 10 a 15 minutos |
| **Ambiente** | Windows 10 + Sysmon + Elastic |
| **Capítulo relacionado** | [Capítulo 012 — Engenharia Social e Phishing](../fundamentos/012-engenharia-social-e-phishing.md) |

## Objetivo

Simular um phishing de forma controlada e depois comprovar a interação do usuário por meio da telemetria coletada no Elastic.

```text
Phishing
   ↓
Clique no link
   ↓
Página falsa
   ↓
Download
   ↓
Sysmon
   ↓
Elastic
```

Nenhum malware real será utilizado.

# Parte 1 — Simulação do phishing

## 1. Abrir o e-mail simulado

Na VM Windows, abra o HTML utilizado para representar o e-mail:

```powershell
Start-Process "C:\SOC-Lab\Phishing-003\email-simulado.html"
```

A mensagem contém um falso aviso do RH solicitando acesso urgente a um documento.

Clique em:

```text
Acessar documento
```

### Evidência 1 — E-mail de phishing

![E-mail de phishing com botão Acessar documento](assets/lab-003-phishing/01-email-phishing.png)

## 2. Acessar a página falsa

Após clicar no link, o navegador acessará:

```text
http://10.0.2.2:8080/index.html
```

A página apresenta o botão:

```text
Visualizar documento
```

### Evidência 2 — Página falsa

![Página falsa com botão Visualizar documento](assets/lab-003-phishing/02-pagina-visualizar-documento.png)

## 3. Realizar o download

Clique em:

```text
Visualizar documento
```

O navegador fará o download do arquivo inofensivo:

```text
documento.txt
```

### Evidência 3 — Download no navegador

![Download do documento no navegador](assets/lab-003-phishing/03-download-navegador.png)

# Parte 2 — Análise no Elastic

Agora assumimos o papel do analista SOC.

O objetivo é verificar se a telemetria confirma que o usuário acessou a infraestrutura do phishing e realizou o download do arquivo.

## 4. Procurar a conexão de rede

No Elastic Discover, filtre:

```kql
host.name: "labsoc1"
and event.code: "3"
and process.name: "chrome.exe"
and destination.ip: "10.0.2.2"
and destination.port: 8080
```

Campos importantes:

```text
@timestamp
event.code
process.name
destination.ip
destination.port
host.name
```

### Evidência 4 — Conexão com a infraestrutura do phishing

![Evento de rede do Chrome para 10.0.2.2 na porta 8080](assets/lab-003-phishing/04-conexao-rede-evento-3.png)

## 5. Procurar a criação do arquivo

Agora filtre eventos de criação de arquivo:

```kql
host.name: "labsoc1"
and event.code: "11"
and process.name: "chrome.exe"
```

Durante o download, o Sysmon registrou arquivos temporários e o seguinte artefato:

```text
documento.txt:Zone.Identifier
```

Esse evento ajuda a comprovar que o arquivo foi criado no endpoint durante a atividade do navegador.

### Evidência 5 — Evento de criação do arquivo

![Evento 11 mostrando a criação do arquivo baixado](assets/lab-003-phishing/05-download-evento-11.png)

## 6. Montar a timeline

Para deixar apenas os eventos importantes:

```kql
host.name: "labsoc1"
and (
    (
        event.code: "1"
        and process.name: "chrome.exe"
    )
    or
    (
        event.code: "3"
        and process.name: "chrome.exe"
        and destination.ip: "10.0.2.2"
        and destination.port: 8080
    )
    or
    (
        event.code: "11"
        and process.name: "chrome.exe"
    )
)
```

Ordene por `@timestamp`.

A sequência esperada é:

```text
chrome.exe iniciado
      ↓
conexão com 10.0.2.2:8080
      ↓
download iniciado
      ↓
arquivo criado
```

### Evidência 6 — Timeline completa

![Timeline dos eventos do phishing no Elastic](assets/lab-003-phishing/06-timeline-completa.png)

# Conclusão

A simulação permitiu comprovar tecnicamente a interação com o phishing.

```text
E-mail
   ↓
Clique
   ↓
Chrome
   ↓
10.0.2.2:8080
   ↓
Download
   ↓
documento.txt
```

Durante a investigação identificamos:

- processo responsável;
- endereço IP e porta de destino;
- eventos de rede;
- eventos de criação de arquivo;
- sequência temporal da atividade.

O cenário pode ser relacionado a:

```text
T1566.002 — Spearphishing Link
T1204.001 — Malicious Link
```

O principal aprendizado é que não basta dizer que uma mensagem “parece phishing”.

Precisamos buscar evidências que mostrem **o que aconteceu depois da interação do usuário**.

[← Voltar ao Capítulo 012 — Engenharia Social e Phishing](../fundamentos/012-engenharia-social-e-phishing.md){ .md-button }

> **Entender antes de decorar.**
