# 07 — Priorização Ofensiva: Ordem de Coleta de Credenciais

> Perspectiva de red team. Os outros documentos respondem "onde cada credencial está". Este responde outra pergunta: o que um operador coleta primeiro, com qual privilégio e por quê. Para a defesa, essa ordem define a prioridade dos SACLs, dos canary files e das regras de detecção (doc 06).

## Fase 1: primeira coleta (usuário comum, sem elevação)

Tudo o que o próprio usuário lê sem ser admin. É o alvo típico de infostealer e do primeiro estágio de um engajamento.

| Caminho | Técnica ATT&CK | Por que vem primeiro |
|---|---|---|
| `%USERPROFILE%\.ssh\` | T1552.004 | Chave privada em texto claro é acesso direto a servidor Linux |
| `.aws\credentials`, `.azure\`, `.kube\config` | T1552.001 | Acesso à cloud inteira, geralmente com privilégio largo |
| `.git-credentials`, `_netrc` | T1552.001 | Texto claro, abre repositório e código-fonte |
| `.docker\config.json` | T1552.001 | Auth em Base64 de registries |
| `Login Data` + `Local State` dos navegadores Chromium | T1555.003 | Senhas e cookies de tudo; o DPAPI é do próprio usuário |
| `logins.json` + `key4.db` (Gecko) | T1555.003 | Sem senha mestre, decriptável offline |
| `%APPDATA%\discord\Local Storage\leveldb\` | T1528 | Conta inteira sem senha nem 2FA, até revogação manual |
| `sitemanager.xml`, WinSCP, mRemoteNG, MobaXterm | T1552 | Credenciais de servidores: mapa pronto para lateral |
| Arquivos `.rdp` em Desktop/Documents | T1552 | Podem trazer blob DPAPI de senha |
| `Microsoft\Credentials\` e Vault | T1555.004 | `TERMSRV/*` abrem com a masterkey do próprio usuário |

O custo para o atacante é baixo. O custo para a defesa não notar é alto: leitura de arquivo pequeno raramente dispara alerta sem SACL ou Sysmon dedicado.

## Fase 2: pós-elevação (admin local ou SYSTEM)

Com privilégio, o alvo muda. Agora interessa os outros usuários da máquina e a máquina como trampolim de domínio.

| Caminho | Técnica ATT&CK | Por que é priorizado |
|---|---|---|
| Memória do `lsass.exe` | T1003.001 | Hashes e tickets de todas as sessões ativas, incluindo admin de domínio que logou na máquina |
| `SAM` + `SYSTEM` via VSS | T1003.002 | Hashes locais; reuso de senha de admin local entre máquinas é comum |
| Hive `SECURITY` | T1003.004 | `DPAPI_SYSTEM`, senhas de serviço, autologon |
| `Microsoft\Protect\{SID}\` de outros usuários | T1555.004 | Com a senha ou hash do usuário, todos os blobs DPAPI dele abrem offline |
| `HKLM\SECURITY\Cache` (`NL$*`) | T1003.005 | Hashes MSCash v2 dos últimos logons de domínio, quebra offline |
| `Crypto\RSA\MachineKeys\` | T1552.004 | Chaves privadas de certificado da máquina |
| Perfis Wi-Fi em `Wlansvc` | T1552 | Senhas de rede sem fio, úteis para persistência física |
| Em DC: `NTDS.dit` + SYSTEM | T1003.003 | Todos os hashes do AD. Fim de jogo |

## Fase 3: movimento lateral

A credencial coletada vira acesso a outro host. O foco sai da coleta e vai para o alcance.

| Fonte | Técnica ATT&CK | Uso |
|---|---|---|
| Credenciais RDP salvas (`TERMSRV/*`) | T1021.001 | Logon interativo em servidor, sem malware |
| Sessões WinSCP, mRemoteNG, MobaXterm | T1021.004 | Inventário de hosts e credenciais no mesmo arquivo |
| Tokens AWS, Azure, kubectl | T1528 | Movimento lateral para a cloud, fora do alcance do EDR de endpoint |
| Tickets Kerberos do lsass | T1550.003 | Pass-the-Ticket para serviços de domínio |
| Hashes NTLM | T1550.002 | Pass-the-Hash em SMB, WMI, WinRM |
| Cache de domínio em máquina de admin | T1003.005 | Crack offline e credencial de domínio válida |

## O que isso muda na defesa

1. Ordene SACLs e canary files pela Fase 1. É onde o custo-benefício do atacante é melhor, e onde a detecção precoce corta a cadeia antes de escalar.
2. Canary de alta fidelidade: `.ssh\id_rsa`, `.aws\credentials` e `sitemanager.xml` falsos, com auditoria de leitura (doc 06).
3. Quebre a Fase 2 com Credential Guard, RunAsPPL e LAPS. Juntos, eles eliminam os três primeiros itens da tabela.
4. Monitore a Fase 3: logon RDP e WinRM com conta local, uso de ticket fora de horário, autenticação de cloud vinda de IP novo.

## Referências

- MITRE ATT&CK: T1003, T1552, T1555, T1528, T1550, T1021
- Relatórios públicos de infostealers (RedLine, Vidar, Lumma) e suas listas de alvos
- Docs 01 a 06 deste repositório
