# 01 — DPAPI e Credential Manager

## O que é o DPAPI

O **Data Protection API (DPAPI)** é o mecanismo nativo do Windows para criptografar segredos em nível de usuário ou de máquina, sem que a aplicação precise gerenciar chaves. Quase todas as credenciais salvas no sistema passam por ele.

- **Escopo de usuário**: protegido por uma *masterkey* derivada da senha de logon do usuário.
- **Escopo de máquina** (`CRYPTPROTECT_LOCAL_MACHINE`): protegido por uma *masterkey* vinculada à conta da máquina no LSA.

## Masterkeys

| Escopo | Caminho |
|---|---|
| Usuário | `%APPDATA%\Microsoft\Protect\{SID}\{GUID}` |
| Usuário (chave preferida) | `%APPDATA%\Microsoft\Protect\{SID}\Preferred` |
| Máquina | `C:\Windows\System32\Microsoft\Protect\S-1-5-18\`<br>`C:\Windows\System32\Microsoft\Protect\S-1-5-18\User\` |

Detalhes técnicos:

- Cada arquivo `{GUID}` é uma masterkey de 64 bytes criptografada (AES-256-SHA512 nas versões modernas; 3DES-SHA1 em legado).
- O arquivo `Preferred` indica qual masterkey está ativa; masterkeys são rotacionadas a cada ~90 dias.
- A masterkey do **usuário** é derivada de `SHA1(senha)` + SID. Por isso, quem conhece a senha do usuário (ou um backup das masterkeys + senha) pode decriptar todos os blobs DPAPI daquele usuário offline.
- A masterkey da **máquina** é protegida pelo segredo LSA `DPAPI_SYSTEM` (ver doc 02).

## Credential Manager (arquivos)

| Local | Caminho |
|---|---|
| Credenciais do usuário | `%APPDATA%\Microsoft\Credentials\` |
| Credenciais locais | `%LOCALAPPDATA%\Microsoft\Credentials\` |
| Credenciais do sistema | `C:\Windows\System32\config\systemprofile\AppData\Local\Microsoft\Credentials\` |

- Cada arquivo é um **Credential Blob** criptografado com DPAPI, contendo alvo (ex.: `TERMSRV/servidor01`), tipo, persistência e o segredo.
- Enumerável via API `CredEnumerate()` ou `vaultcmd /listcreds`.

## Windows Vault

| Local | Caminho |
|---|---|
| Vault do usuário | `%APPDATA%\Microsoft\Vault\{GUID}\` |
| Vault local | `%LOCALAPPDATA%\Microsoft\Vault\` |
| Vault do sistema | `C:\ProgramData\Microsoft\Vault\` |

- Diretórios contêm `Policy.vpol` (schema/chave do vault) e arquivos `.vcrd` (credenciais individuais).
- Usado por Internet Explorer/Edge legado, credenciais web e alguns apps UWP.

## Implicações de segurança

- Ferramentas como **mimikatz** (`dpapi::cred`, `dpapi::vault`) e **SharpDPAPI** exploram exatamente esses caminhos: masterkey + blob = segredo em texto claro.
- Reduzir risco: habilitar **Credential Guard**, não salvar credenciais RDP, restringir logons interativos de contas privilegiadas em estações comuns.
