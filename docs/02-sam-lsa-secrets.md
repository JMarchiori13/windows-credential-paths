# 02 — SAM, LSA Secrets e Cache de Domínio

## SAM (Security Account Manager)

**Caminho**: `C:\Windows\System32\config\SAM`
Hive carregado em: `HKLM\SAM`

- Armazena os **hashes NTLM** (e histórico) das contas **locais**.
- Permissões: apenas `SYSTEM` — nem administradores leem diretamente sem elevação.
- Criptografado com **SYSKEY** (bootkey), derivado das chaves do hive `SYSTEM`.
- Em disco, só é acessível com o sistema offline ou via volume shadow copy (`vssadmin`).

Arquivos relacionados (mesma pasta `config\`):

| Arquivo | Conteúdo |
|---|---|
| `SAM` | Hashes de contas locais |
| `SYSTEM` | Bootkey/SYSKEY, configurações de serviço |
| `SECURITY` | LSA Secrets, cache de domínio |
| `NTDS.dit` (DCs) | Base AD completa — `C:\Windows\NTDS\NTDS.dit` |

## LSA Secrets

**Local**: `HKLM\SECURITY\Policy\Secrets` (hive `SECURITY`)

Armazena segredos criptografados com a bootkey + chaves LSA:

- `DPAPI_SYSTEM` — protege as masterkeys DPAPI de escopo máquina.
- `NL$KM` — chave usada no cache de credenciais de domínio.
- `DefaultPassword` — senha de **autologon** (quando `AutoAdminLogon=1` em `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`).
- `_SC_<serviço>` — senhas de contas de **serviços** configurados com conta de usuário.
- `RasCredentials`, segredos de VPN, dados de tarefas agendadas com credencial armazenada (`S-1-5-18` tasks).

## Cache de credenciais de domínio (MSCash)

**Local**: `HKLM\SECURITY\Cache` — valores `NL$1` a `NL$10`

- Guarda os últimos **10 logons de domínio** (por padrão) como hash **MSCash v2** (PBKDF2-SHA1, 10240 iterações), permitindo logon offline.
- Configurável via `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\CachedLogonsCount` (0 desativa).
- Formato lento de quebrar, mas extraível offline com acesso ao hive + bootkey.

## LSASS (memória)

Embora não seja "caminho em disco", o processo **`lsass.exe`** mantém em memória:

- Hashes NTLM de sessões ativas
- Tickets Kerberos (TGT/TGS)
- Credenciais em texto claro quando WDigest está habilitado (`HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest\UseLogonCredential = 1`)

Dumps (`lsass.dmp`, comsvcs.dll MiniDump, ProcDump) são o vetor clássico de credential dumping (MITRE ATT&CK T1003.001).

## Implicações de segurança

- Monitorar acesso aos hives `SAM`/`SECURITY`/`SYSTEM` e ao `NTDS.dit`.
- Habilitar **Credential Guard** (isola segredos no LSAiso via VSM).
- Definir `CachedLogonsCount` baixo em laptops de usuários privilegiados.
- Usar **gMSA** para serviços em vez de contas com senha armazenada.
