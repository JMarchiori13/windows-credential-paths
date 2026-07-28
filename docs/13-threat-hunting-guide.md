# 13 — Threat Hunting com os Caminhos deste Repositório

> Perspectiva defensiva, na prática. Os docs anteriores dizem onde os segredos moram. Este transforma cada caminho em pergunta de hunting: o que procurar, em qual fonte, e o que conta como suspeito. É o manual de uso do repositório para um SOC.

## Como ler este guia

Cada seção segue o mesmo formato: hipótese de caça, fontes de dados, e o que separa o normal do anormal. A maioria das consultas pressupõe Sysmon com config pública (SwiftOnSecurity ou Olaf) e os logs encaminhados para um SIEM.

## Hunts por camada

### DPAPI e Credential Manager (doc 01)

| Hunt | Fonte | Sinal |
|---|---|---|
| Leitura de masterkeys | Sysmon EID 11 ou SACL em `%APPDATA%\Microsoft\Protect\` | Processo fora do sistema lendo `{SID}\{GUID}` ou `Preferred` |
| Enumeração de credenciais | Sysmon EID 1 | `vaultcmd /listcreds`, chamadas de `CredEnumerate` via tooling |
| Acesso a blobs de credencial | EID 11 em `Microsoft\Credentials\` | Qualquer leitura por processo de usuário comum |

O normal é quase zero. Poucos processos legítimos tocam esses diretórios, o que faz deste hunt um dos de maior precisão.

### SAM, LSA e cache (doc 02)

| Hunt | Fonte | Sinal |
|---|---|---|
| Cópia dos hives | EID 11, EID 1 | `reg save HKLM\SAM`, acesso a `config\SAM`, `SYSTEM`, `SECURITY` |
| Shadow copy para extração | EID 1 | `vssadmin create shadow`, `wmic shadowcopy` |
| Dump do lsass | EID 10 | GrantedAccess 0x1010/0x1FFFFF em `lsass.exe` por processo não PPL |
| NTDS.dit | EID 11 no DC | Leitura do arquivo por processo que não é o NTDS |

### Chaves SSH e cloud (docs 03, 12)

| Hunt | Fonte | Sinal |
|---|---|---|
| Leitura de `.ssh` | EID 11 | Processo inesperado lendo `id_rsa`, `id_ed25519` |
| Leitura de `.aws`, `.kube`, `.azure` | EID 11 | Mesmo padrão, pastas de cloud CLI |
| Acesso ao cache MSAL | EID 11 | Leitura de `IdentityCache` por processo fora dos apps Microsoft |

### Navegadores e tokens (doc 04)

| Hunt | Fonte | Sinal |
|---|---|---|
| Extração de `Login Data` | EID 11 | Leitura de `Login Data` + `Local State` na mesma janela de tempo |
| Token do Discord | EID 11 | Acesso a `leveldb` do Discord por processo que não é o Discord |
| Histórico do PowerShell | EID 1, 4104 | Senha como argumento de comando; `ConsoleHost_history.txt` lido por terceiro |

### Rede e movimento (docs 05, 08, 13 do repo irmão)

| Hunt | Fonte | Sinal |
|---|---|---|
| Enumeração Wi-Fi | EID 1 | `netsh wlan show profile` com `key=clear` |
| Credenciais RDP | EID 1, 4624 | `cmdkey /list`, logon tipo 10 fora do baseline |
| Pass-the-Hash | EID 4624 | Logon tipo 9 (NewCredentials), sempre raro |

## Canary files: o hunt que caça sozinho

A técnica de maior custo-benefício do repositório. Plante arquivos falsos nos caminhos da fase 1 (`.ssh\id_rsa`, `.aws\credentials`, `sitemanager.xml`, `Login Data` de um perfil fantasma) e coloque SACL de auditoria de leitura em cada um.

Qualquer leitura é incidente. Não existe processo legítimo com motivo para abrir esses arquivos. O canary transforma o hunt ativo em alarme passivo de precisão quase perfeita.

## Priorização

Ordene os hunts pelo retorno sobre o esforço:

1. Canary files: implementação de um dia, precisão quase perfeita.
2. Dump de lsass (EID 10) e reg save de hives: poucos falsos positivos, impacto alto.
3. Leitura de `.ssh` e pastas de cloud: barato, pega a fase 1 da coleta.
4. Login Data de navegador com Local State: assinatura clássica de infostealer.
5. Baseline de 4624 por tipo: mais trabalhoso, mas é o que enxerga o lateral maduro.

## Referências

- Sysmon (documentação de eventos), SigmaHQ, MITRE ATT&CK
- Docs 01 a 12 deste repositório, e o doc 07 para a ordem ofensiva que este guia inverte
