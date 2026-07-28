# 01 — DPAPI e Credential Manager

## O que é o DPAPI

O Data Protection API é o mecanismo do Windows para criptografar segredos sem que a aplicação precise gerenciar chaves. Quase toda credencial salva no sistema passa por ele, de senhas de RDP a cookies de navegador.

São dois escopos. O de usuário deriva a chave da senha de logon da pessoa. O de máquina (`CRYPTPROTECT_LOCAL_MACHINE`) vincula a chave à conta do computador no LSA.

## Masterkeys

| Escopo | Caminho |
|---|---|
| Usuário | `%APPDATA%\Microsoft\Protect\{SID}\{GUID}` |
| Chave preferida do usuário | `%APPDATA%\Microsoft\Protect\{SID}\Preferred` |
| Máquina | `C:\Windows\System32\Microsoft\Protect\S-1-5-18\` e subpasta `User\` |

Cada arquivo `{GUID}` guarda uma masterkey de 64 bytes, criptografada com AES-256-SHA512 nas versões atuais e 3DES-SHA1 nas antigas. O arquivo `Preferred` aponta qual está ativa, e a rotação acontece a cada 90 dias, aproximadamente.

Dois detalhes que importam muito na prática. A masterkey do usuário deriva de `SHA1(senha)` mais o SID, então quem tem a senha da pessoa, ou um backup das masterkeys somado a ela, decripta offline todos os blobs DPAPI daquele usuário. E a masterkey da máquina é protegida pelo segredo LSA `DPAPI_SYSTEM`, assunto do doc 02.

## Credential Manager em disco

| Local | Caminho |
|---|---|
| Credenciais do usuário | `%APPDATA%\Microsoft\Credentials\` |
| Credenciais locais | `%LOCALAPPDATA%\Microsoft\Credentials\` |
| Credenciais do sistema | `C:\Windows\System32\config\systemprofile\AppData\Local\Microsoft\Credentials\` |

Cada arquivo é um Credential Blob criptografado com DPAPI, com alvo (por exemplo `TERMSRV/servidor01`), tipo, persistência e o segredo em si. Dá para enumerar sem ler os arquivos, pela API `CredEnumerate()` ou pelo `vaultcmd /listcreds`.

## Windows Vault

| Local | Caminho |
|---|---|
| Vault do usuário | `%APPDATA%\Microsoft\Vault\{GUID}\` |
| Vault local | `%LOCALAPPDATA%\Microsoft\Vault\` |
| Vault do sistema | `C:\ProgramData\Microsoft\Vault\` |

Dentro ficam o `Policy.vpol`, com o esquema e a chave do vault, e os arquivos `.vcrd`, um por credencial. É o repositório usado pelo Internet Explorer, pelo Edge legado e por alguns aplicativos UWP.

## Implicações de segurança

Ferramentas como mimikatz (`dpapi::cred`, `dpapi::vault`) e SharpDPAPI exploram exatamente esses caminhos. A lógica é direta: masterkey mais blob resulta em segredo em texto claro.

A defesa passa por Credential Guard, por não salvar credenciais de RDP e por restringir logon interativo de contas privilegiadas em estações comuns.
