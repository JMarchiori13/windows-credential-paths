# 03 — Chaves SSH no Windows

## OpenSSH Client (usuário)

**Caminho padrão**: `%USERPROFILE%\.ssh\`

| Arquivo | Conteúdo | Proteção |
|---|---|---|
| `id_ed25519` / `id_rsa` / `id_ecdsa` | **Chave privada** | Nenhuma por padrão; passphrase opcional |
| `id_ed25519.pub` / `id_rsa.pub` | Chave pública | — |
| `known_hosts` | Fingerprints de hosts | — |
| `config` | Config do client (pode referenciar chaves) | — |

- Formato OpenSSH (`-----BEGIN OPENSSH PRIVATE KEY-----`). Sem passphrase, a chave privada está **em texto claro no disco** — qualquer processo rodando como o usuário a lê.

## OpenSSH Windows — serviço e administrador

| Item | Caminho |
|---|---|
| Host keys do servidor sshd | `C:\ProgramData\ssh\ssh_host_*_key` |
| `sshd_config` | `C:\ProgramData\ssh\sshd_config` |
| `authorized_keys` (usuário comum) | `%USERPROFILE%\.ssh\authorized_keys` |
| `administrators_authorized_keys` | `C:\ProgramData\ssh\administrators_authorized_keys` |

Nota: para membros do grupo **Administrators**, o sshd do Windows lê `administrators_authorized_keys` (ACL restrita a SYSTEM/Administrators), **não** o arquivo do perfil.

## ssh-agent (serviço Windows)

- Serviço: `ssh-agent` (`HKLM\SYSTEM\CurrentControlSet\Services\ssh-agent`) — desabilitado por padrão.
- Chaves adicionadas via `ssh-add` ficam registradas em `HKCU\Software\OpenSSH\Agent\Keys`, criptografadas com **DPAPI do usuário**.
- Socket: pipe nomeado `\\.\pipe\openssh-ssh-agent`.

## PuTTY / Pageant

| Item | Local |
|---|---|
| Chaves privadas (arquivos `.ppk`) | onde o usuário salvar — comuns: `Desktop`, `Documents`, `%USERPROFILE%` |
| Sessões salvas (registro) | `HKCU\Software\SimonTatham\PuTTY\Sessions\<sessão>` |
| Host keys | `HKCU\Software\SimonTatham\PuTTY\SshHostKeys` |
| Proxy credentials / senhas de sessão | dentro da sessão no registro, quando configuradas |

- Arquivos `.ppk` sem passphrase também são texto claro.
- **Pageant** mantém chaves descriptografadas **em memória** — alvo de dump de processo.

## Permissões NTFS esperadas (hardening)

```powershell
# .ssh deve ser acessível apenas pelo dono
icacls "$env:USERPROFILE\.ssh" /inheritance:r /grant:r "$env:USERNAME:F"
```

O OpenSSH do Windows **recusa** chaves privadas com ACL permissiva demais (mesmo comportamento do OpenSSH no Linux com permissões de arquivo).

## Implicações de segurança

- Chaves sem passphrase em `Desktop`/`Documents` são achado frequente em resposta a incidentes.
- Exfiltração de `.ssh` inteiro cabe em poucos KB — monitorar leituras incomuns nesses caminhos (Sysmon Event ID 11, SACLs).
- Preferir chaves com passphrase + agente, ou armazenar em TPM/`solokeys`/Hello for Business quando possível.
