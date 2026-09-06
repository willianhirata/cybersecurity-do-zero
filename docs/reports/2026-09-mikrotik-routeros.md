<p align="center"> <img src="https://willianhirata.github.io/cybersecurity-do-zero/assets/logo.png" alt="Cybersecurity do Zero" width="120"> </p>

# MikroTrick — Bypass de Autenticação SSH em Massa no MikroTik RouterOS

**Data da divulgação:** 05 de setembro de 2026
**Descoberto e coordenado por:** CERT Polska
**Severidade:** Crítica (CVSS até 9.2)
**Status:** Exploração ativa confirmada desde 02/09/2026

---

## Resumo

O CERT Polska identificou seis vulnerabilidades no MikroTik RouterOS. Duas delas, combinadas, permitem que um atacante assuma o controle administrativo total de um dispositivo **sem qualquer autenticação**, desde que o serviço SSH esteja exposto à internet. Essa cadeia recebeu o nome **MikroTrick**.

Já há confirmação de exploração ativa em ambiente real, com ataques documentados desde pelo menos 2 de setembro de 2026 — um dia *antes* da MikroTik publicar as correções, o que levanta a suspeita de que se trata de um 0-day genuíno (ou que os atacantes anteciparam o lançamento do patch e agiram rapidamente). Nem o CERT Polska nem a cobertura da imprensa especializada confirmaram publicamente qual dos dois cenários é o correto.

## Linha do Tempo

| Data | Evento |
|---|---|
| 02/09/2026 | Primeiros indícios de exploração ativa em fóruns e logs de usuários |
| 03/09/2026 | MikroTik libera correções em todos os canais (7.25beta3, 7.24.2, 7.23.4, 6.49.21) e, pela primeira vez na história da empresa, envia notificação push pelo aplicativo mobile alertando os usuários |
| 04/09/2026 | MikroTik libera a versão 7.23.5 (long-term), corrigindo uma regressão de DHCPv6 introduzida na 7.23.4 e mantendo a correção de segurança |
| 05/09/2026 | CERT Polska publica o alerta público e a página técnica com as seis CVEs; o pesquisador Costin Raiu publica uma análise técnica independente no Medium |
| 06/09/2026 | Cobertura ampla na imprensa especializada (The Hacker News, Security Affairs, entre outros) |

## A Cadeia "MikroTrick"

A cadeia combina duas falhas distintas no mecanismo de login SSH do RouterOS:

- **CVE-2026-67276** — permite autenticar-se como um usuário autorizado sem possuir a chave privada correspondente.
- **CVE-2026-86060** — permite escalar privilégios até obter controle administrativo total durante o próprio processo de login SSH.

Juntas, elas dão a um atacante não autenticado acesso administrativo completo a qualquer RouterOS com SSH acessível pela internet. O CERT Polska optou por não detalhar publicamente o mecanismo exato de encadeamento entre as duas falhas — nem esta publicação, nem a cobertura da imprensa especializada que a analisou, expõem esse detalhe, justamente para não facilitar a automação do ataque por quem ainda não o tenha reproduzido.

## As Seis Vulnerabilidades

| CVE | CWE | Resumo |
|---|---|---|
| [CVE-2026-67276](https://www.cve.org/CVERecord?id=CVE-2026-67276) | CWE-347 (verificação imprópria de assinatura criptográfica) | Bypass de autenticação SSH via chave RSA incompleta (CVSS 9.2) |
| [CVE-2026-86060](https://www.cve.org/CVERecord?id=CVE-2026-86060) | CWE-88 (injeção de argumento) | Escalonamento de privilégio via username malformado no login SSH (CVSS 9.2) |
| [CVE-2026-67277](https://www.cve.org/CVERecord?id=CVE-2026-67277) | CWE-306 (ausência de autenticação para função crítica) | Vazamento de memória do kernel / DoS via bandwidth-test (CVSS 8.8) |
| [CVE-2026-67278](https://www.cve.org/CVERecord?id=CVE-2026-67278) | CWE-347 | Falsificação de certificado X.509 / impersonação de servidor TLS |
| [CVE-2026-67279](https://www.cve.org/CVERecord?id=CVE-2026-67279) | CWE-841 (fluxo de trabalho comportamental impróprio) | Execução de comando via SSH sem autenticação, através de um rekey |
| [CVE-2026-67281](https://www.cve.org/CVERecord?id=CVE-2026-67281) | CWE-824 (acesso a ponteiro não inicializado) | Leitura arbitrária de arquivos via WebFig (`/jsproxy`) |

*Todas afetam: RouterOS 6.x anterior a 6.49.21; 7.0 a 7.23.x anterior a 7.23.4; e 7.24 anterior a 7.24.2.*

### CVE-2026-67276 — Bypass de autenticação SSH

O RouterOS validava apenas o tipo e o módulo da chave pública RSA usada na autenticação SSH, sem conferir o expoente. Como a verificação da assinatura depende da chave enviada pelo próprio cliente, bastava conhecer o módulo de um usuário autorizado para montar uma chave falsa com expoente 1, forjar uma assinatura válida e abrir uma sessão SSH como esse usuário — sem jamais ter tido acesso à chave privada real.

### CVE-2026-86060 — Escalonamento de privilégio via username

Um nome de usuário SSH iniciado por um caractere não permitido não era tratado corretamente pelo mecanismo de login, permitindo alterar a máscara de política interna do sistema e escalar privilégios até uma sessão com controle administrativo completo.

### CVE-2026-67277 — Vazamento de memória e DoS via bandwidth-test

O serviço de teste de banda aceitava uma conexão "relacionada" antes mesmo de a sessão principal concluir a autenticação. Um cliente não autenticado podia iniciar um teste UDP e, com a opção `random-data` desativada, receber de volta um trecho de memória não inicializada do buffer de pacotes do kernel. Um erro paralelo de validação de tamanho (underflow de inteiro) também permitia gerar pacotes fragmentados anormalmente grandes, podendo derrubar e reiniciar o kernel do RouterOS.

### CVE-2026-67278 — Falsificação de certificado X.509

A validação de certificados X.509 aceitava assinaturas RSA/PKCS#1 v1.5 malformadas. Como o repositório de confiança do sistema inclui uma CA raiz com expoente público *e=3*, um atacante capaz de interceptar ou redirecionar uma conexão TLS de saída do RouterOS podia usar apenas o certificado público dessa raiz — sem a chave privada — para forjar um intermediário confiável e emitir certificados para qualquer domínio, se passando por um servidor legítimo.

### CVE-2026-67279 — Execução de comando sem autenticação via SSH

Mesmo sem o usuário jamais ter sido autenticado, o servidor SSH avançava para a fase de protocolo de conexão após uma renegociação de chaves (rekey) solicitada pelo cliente. Isso permitia abrir um canal de sessão e enviar um comando de execução que o servidor chegava a processar — possibilitando criar, sobrescrever e reconstruir arquivos no sistema de arquivos gerenciado do RouterOS, incluindo arquivos de suporte com dados de configuração e diagnóstico.

### CVE-2026-67281 — Leitura arbitrária de arquivos via WebFig

A interface WebFig possui uma falha de leitura de arquivo não autenticada no caminho `/jsproxy`: uma sessão recém-criada mantinha um ponteiro de identidade ("principal") não inicializado, usado para autorizar acesso a arquivos. Um atacante conseguia preparar o alocador de memória para que esse ponteiro apontasse para uma identidade com privilégios suficientes e, então, usar componentes de diretório-pai em uma URI codificada para escapar do namespace de arquivos do WebFig — expondo arquivos de propriedade do root, incluindo armazenamentos de configuração com credenciais.

## Indicadores de Comprometimento (IoCs)

**Entradas de log:**

```
login failure for user -2 from <ip> via ssh
user <name> added by ssh:-2@<ip>
```

**Outros sinais:**

- Conta de usuário com altos privilégios chamada **"ops"**
- Qualquer entrada em `/system history` associada a `ssh:-2@<ip>` seguida de uma ação de configuração (criação/alteração de usuários, chaves SSH, scripts, tarefas agendadas, regras de firewall, proxies, túneis ou configurações de captura de pacotes) deve ser tratada como comprometimento confirmado, a menos que tenha origem em um teste de segurança autorizado
- A ausência desses registros **não garante** que o dispositivo esteja seguro — logs podem ter sido rotacionados ou apagados

**IPs associados aos ataques observados** (segundo CERT Polska e análise independente de Costin Raiu):

| IP | Papel observado |
|---|---|
| `82.192.72.4` | Origem da maior parte dos ataques confirmados desde 02/09/2026; hospedava um binário BusyBox (build MIPS de 2010) e os arquivos `ftpsrv.py`, `launch.sh` e `serve.py` |
| `103.102.31.18` | Associado a tentativas de exploração da mesma cadeia |

**Hashes SHA-256 dos arquivos identificados na infraestrutura de ataque:**

```
ftpsrv.py  6e95f70fdbabb57881b3f5b2c8465d4b17ba901100704efb1278bb3386e6729d
launch.sh  972b474b896f9fac3cd6b5b8476b410b8f39fbedee8a3b0c745d6e3b328d7dcd
serve.py   6dca83338d60467b65b7789d4d59754e40a7aaa36f40ea2da57538367ac9b89e
```

> Na data da divulgação, a maioria desses arquivos não tinha detecção em serviços como o VirusTotal — reforço de que assinatura de antivírus não é suficiente para detectar esse comprometimento.

## Como Verificar se um Dispositivo foi Afetado

1. Atualize o RouterOS para uma versão corrigida (ver seção de recomendações).
2. Após atualizar, verifique o log em busca da mensagem crítica de dispositivo "Flagged".
3. Rode `/system/device-mode/print` e confira o status do marcador "Flagged" — consulte a [documentação oficial](https://manual.mikrotik.com/docs/system-information-and-utilities/device-mode#flagged-status) para os próximos passos caso ele esteja ativo.
4. **Mesmo sem o marcador ativo**, inspecione manualmente a configuração em busca de usuários, scripts, tarefas agendadas, proxies ou túneis desconhecidos — o mecanismo "Flagged" detecta apenas alguns padrões conhecidos de comprometimento, não todos.

## Pesquisa Apoiada por IA

Um detalhe curioso dessa divulgação: as seis vulnerabilidades foram descobertas pelo próprio CERT Polska com apoio de modelos de linguagem (GPT-5.5-cyber e GPT-5.6-sol), dentro do programa de colaboração GTAC da OpenAI. Os agentes de IA automatizaram parte da criação de laboratórios, comparação de versões e análise de binários e RFCs, mas toda hipótese ainda precisou ser confirmada manualmente em um sistema real, com testes de controle negativo e validação de impacto pelos pesquisadores humanos.

Depois da divulgação, o pesquisador Costin Raiu testou se diferentes modelos de IA conseguiriam reproduzir o exploit a partir das informações públicas: um deles recusou por motivos de segurança, e os demais tentaram, mas nenhum chegou a uma implementação funcional — o que sugere uma janela de tempo (ainda que curta) antes que uma prova de conceito pública apareça.

## Recomendações

| Faixa afetada (segundo CERT Polska) | Correção inicial | Observação |
|---|---|---|
| 6.0.0 até anterior a 6.49.21 | 6.49.21 | Versão de segurança do RouterOS 6 |
| 7.0.0 até anterior a 7.23.4 | 7.23.4 | Prefira a **7.23.5**, que corrige uma regressão de DHCPv6 introduzida na 7.23.4 |
| 7.24 até anterior a 7.24.2 | 7.24.2 | Versão de segurança do canal Stable |
| Canal de desenvolvimento | 7.25beta3 | Correção no canal Development |

Se a atualização não puder ser aplicada imediatamente:

- Desative ou restrinja o acesso externo aos serviços expostos — especialmente SSH, WWW/WWW-SSL e bandwidth-test — permitindo apenas redes de gerência confiáveis.
- Evite iniciar conexões TLS a partir do dispositivo não corrigido e evite usar os clientes SSH nativos do RouterOS (`/system ssh` e `/system ssh-exec`) em redes não confiáveis.
- Essas são mitigações temporárias — não substituem a atualização.

Se houver indício de comprometimento:

1. Isole o dispositivo da rede e preserve logs e configuração **antes** de qualquer reset.
2. Restaure para configuração de fábrica e reconstrua a partir de um backup confiável e verificado — nunca restaure diretamente um backup de um dispositivo potencialmente comprometido.
3. Troque todas as senhas, chaves e demais segredos em uso.
4. Não limpe o marcador "Flagged" antes de concluir a análise e preservar as evidências.

## Por Que Isso Importa (Blue Team)

Esse caso é um bom lembrete de por que monitorar logs de autenticação SSH — mesmo em equipamentos de borda como roteadores — é parte essencial de uma estratégia de detecção. Um usuário de login inválido tão simples quanto `-2` seria fácil de ignorar em meio ao ruído normal de tentativas de força bruta, mas nesse caso era exatamente o indício que diferenciava um ataque direcionado de spam automatizado. Também reforça um princípio incômodo do gerenciamento de patches: a publicação de uma correção pode, ela mesma, ser o gatilho que acelera a exploração em massa, já que atacantes frequentemente comparam versões corrigidas com as anteriores para reconstruir a falha original.

## Referências

- [CERT Polska — Alerta principal](https://cert.pl/en/posts/2026/09/vulnerabilities-in-mikrotik-routeros-actively-exploited/)
- [CERT Polska — Página técnica com as 6 CVEs](https://cert.pl/en/posts/2026/09/mikrotik-routeros-cve/)
- [MikroTik — Boletim oficial de segurança (setembro de 2026)](https://mikrotik.com/supportsec/september-2026-vulnerability/)
- [MikroTik — Documentação do status "Flagged"](https://manual.mikrotik.com/docs/system-information-and-utilities/device-mode#flagged-status)
- [The Hacker News — Cobertura do caso](https://thehackernews.com/2026/09/attackers-hijack-mikrotik-routers.html)
- [Security Affairs — Análise de IoCs e cronologia (Pierluigi Paganini)](https://securityaffairs.com/198538/security/your-mikrotik-router-may-already-be-compromised-look-for-ssh-user-2.html)
- [Costin Raiu — Análise técnica independente (Medium)](https://medium.com/@costin.raiu/mikrotik-ssh-0day-exploitation-in-the-wild-20da4587d9b2)
