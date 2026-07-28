# 03 — Chaves SSH no Windows

## OpenSSH client (usuário)

**Caminho padrão**: `%USERPROFILE%\.ssh\`

| Arquivo | Conteúdo | Proteção |
|---|---|---|
| `id_ed25519`, `id_rsa`, `id_ecdsa` | Chave privada | Nenhuma por padrão; passphrase opcional |
| `*.pub` | Chave pública | — |
| `known_hosts` | Fingerprints de hosts | — |
| `config` | Configuração do client, pode referenciar chaves | — |

O formato é o OpenSSH (`-----BEGIN OPENSSH PRIVATE KEY-----`). Sem passphrase, a chave privada é texto claro no disco, e qualquer processo rodando como o usuário lê.

## OpenSSH como serviço

| Item | Caminho |
|---|---|
| Host keys do sshd | `C:\ProgramData\ssh\ssh_host_*_key` |
| `sshd_config` | `C:\ProgramData\ssh\sshd_config` |
| `authorized_keys` (usuário comum) | `%USERPROFILE%\.ssh\authorized_keys` |
| `administrators_authorized_keys` | `C:\ProgramData\ssh\administrators_authorized_keys` |

Um detalhe que pega muita gente: para membros do grupo Administradores, o sshd do Windows lê o `administrators_authorized_keys`, com ACL restrita a SYSTEM e Administrators, e ignora o arquivo do perfil do usuário.

## ssh-agent

O serviço vem desabilitado por padrão (`HKLM\SYSTEM\CurrentControlSet\Services\ssh-agent`). As chaves adicionadas com `ssh-add` ficam registradas em `HKCU\Software\OpenSSH\Agent\Keys`, criptografadas com o DPAPI do usuário, e o socket é o pipe `\\.\pipe\openssh-ssh-agent`.

## PuTTY e Pageant

| Item | Local |
|---|---|
| Chaves `.ppk` | Onde o usuário salvou; Desktop e Documents são os campeões |
| Sessões | `HKCU\Software\SimonTatham\PuTTY\Sessions\<sessão>` |
| Host keys | `HKCU\Software\SimonTatham\PuTTY\SshHostKeys` |
| Credenciais de proxy/sessão | Dentro da sessão no registro, quando configuradas |

Arquivo `.ppk` sem passphrase também é texto claro. E o Pageant mantém as chaves decriptadas em memória, alvo de dump de processo.

## Permissões NTFS

```powershell
icacls "$env:USERPROFILE\.ssh" /inheritance:r /grant:r "$env:USERNAME:F"
```

O OpenSSH do Windows recusa chave privada com ACL permissiva demais, mesmo comportamento do OpenSSH no Linux.

## Implicações de segurança

Chave sem passphrase esquecida em Desktop ou Documents é achado frequente em resposta a incidentes. O diretório `.ssh` inteiro cabe em poucos KB, então leituras incomuns nesses caminhos merecem Sysmon EID 11 e SACL. O caminho mais seguro é passphrase com agente, ou chave em TPM e FIDO2 quando o cenário permite.
