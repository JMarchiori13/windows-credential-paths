# 02 — SAM, LSA Secrets e Cache de Domínio

## SAM (Security Account Manager)

**Caminho**: `C:\Windows\System32\config\SAM`, carregado em `HKLM\SAM`.

Guarda os hashes NTLM das contas locais, com histórico. As permissões são duras: só SYSTEM lê, e nem administrador chega perto sem elevação. Em cima disso, o conteúdo é criptografado com a SYSKEY, derivada do hive `SYSTEM`. Com o Windows rodando, o arquivo está travado; o acesso em disco depende de sistema offline ou de volume shadow copy.

Vizinhos na mesma pasta `config\`:

| Arquivo | Conteúdo |
|---|---|
| `SAM` | Hashes de contas locais |
| `SYSTEM` | Bootkey/SYSKEY, configuração de serviços |
| `SECURITY` | LSA Secrets, cache de domínio |
| `NTDS.dit` (só em DCs) | Base completa do AD, em `C:\Windows\NTDS\` |

## LSA Secrets

**Local**: `HKLM\SECURITY\Policy\Secrets`, no hive `SECURITY`.

Segredos criptografados com a bootkey e chaves do LSA. Os mais relevantes:

- `DPAPI_SYSTEM`, que protege as masterkeys DPAPI de escopo máquina.
- `NL$KM`, a chave do cache de credenciais de domínio.
- `DefaultPassword`, a senha de autologon, quando `AutoAdminLogon=1` no Winlogon.
- `_SC_<serviço>`, senhas de contas de serviço configuradas com usuário comum.
- `RasCredentials`, segredos de VPN e dados de tarefas agendadas que guardam credencial.

## Cache de credenciais de domínio (MSCash)

**Local**: `HKLM\SECURITY\Cache`, valores `NL$1` a `NL$10`.

Guarda os últimos dez logons de domínio como hash MSCash v2 (PBKDF2-SHA1, 10240 iterações), o que permite logon sem o DC por perto. A quantidade é configurável em `CachedLogonsCount` no Winlogon, e zero desativa. O formato é lento de quebrar, mas sai offline com o hive e a bootkey.

## Windows Hello, PIN e biometria

Circula por aí a ideia de que o PIN fica salvo no Windows. Não fica. O PIN nunca é armazenado; ele só destrava uma chave privada, e o mesmo vale para a biometria. É por isso que o Hello resiste a credential dumping de um jeito que senha nenhuma resiste.

### Caminhos

| Item | Caminho | Conteúdo |
|---|---|---|
| Container NGC | `C:\Windows\ServiceProfiles\LocalService\AppData\Local\Microsoft\Ngc\` | Chaves privadas por usuário, referenciadas por GUID |
| Banco biométrico | `C:\Windows\System32\WinBioDatabase\*.DAT` | Templates biométricos, que não são imagens reversíveis |
| Plugins biométricos | `C:\Windows\System32\WinBioPlugins\` | Drivers e adapters de sensores |
| Política de PIN | `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Authentication\LogonUI\NgcPin` | Regras de complexidade |
| Política Hello for Business | `HKLM\SOFTWARE\Policies\Microsoft\PassportForWork` | Configuração corporativa via GPO/MDM |

### Como a proteção funciona

No provisionamento, o Windows gera um par de chaves por usuário e dispositivo. Com TPM 2.0, a chave privada nasce dentro do chip e não sai de lá, nem para SYSTEM; o PIN ou a biometria só autorizam o uso dela. Sem TPM, a chave fica no container NGC protegida por DPAPI, e mesmo assim o PIN continua sem existir em disco em formato algum. A autenticação é desafio-resposta com a chave privada, sem segredo compartilhado com o servidor. Não existe hash de PIN para roubar.

### Implicações

Do lado ofensivo, os ataques reais contra o Hello miram modificar o processo de autenticação (T1556): injeção no WinBio, spoofing de sensor, abuso do fallback para senha. Roubar o PIN não é opção, porque não há o que roubar. O vetor que sobra é container NGC sem TPM somado a senha comprometida, que devolve o problema ao modelo DPAPI do doc 01.

Do lado defensivo: exigir TPM 2.0, política de PIN com complexidade, restringir fallback para senha e monitorar acessos ao `WinBioDatabase` e ao `Ngc\` por processos fora do sistema.

## LSASS em memória

Não é caminho em disco, mas não dava para deixar de fora. O `lsass.exe` mantém em memória hashes NTLM de sessões ativas, tickets Kerberos e, quando o WDigest está ligado (`UseLogonCredential = 1`), credenciais em texto claro. Dumps do processo são o vetor clássico de credential dumping (T1003.001).

## Implicações de segurança

- Monitorar acesso aos hives `SAM`, `SECURITY`, `SYSTEM` e ao `NTDS.dit`.
- Credential Guard ligado, para isolar os segredos no LSAiso.
- `CachedLogonsCount` baixo em laptops de gente privilegiada.
- gMSA para serviços, em vez de conta com senha guardada.
- Migrar logon interativo para Windows Hello for Business com TPM, o que elimina hash NTLM reutilizável das sessões.
