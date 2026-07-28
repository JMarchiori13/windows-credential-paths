# 05 — Credenciais de Rede (Wi-Fi, RDP, VPN, Drives)

## Perfis Wi-Fi

**Caminho**: `C:\ProgramData\Microsoft\Wlansvc\Profiles\Interfaces\{InterfaceGUID}\{ProfileGUID}.xml`

O elemento `<keyMaterial>` guarda a senha WPA/WPA2, criptografada com DPAPI em escopo de máquina. Dá para enumerar sem tocar no disco:

```cmd
netsh wlan show profiles
netsh wlan show profile name="SSID" key=clear
```

## Credenciais RDP salvas

Ficam no Credential Manager como alvos `TERMSRV/<host>` (doc 01). Arquivos `.rdp` salvos manualmente podem trazer `password 51:b:<blob DPAPI hex>`, e aparecem bastante em Documents, Desktop e pastas compartilhadas. O histórico de conexões fica em `HKCU\Software\Microsoft\Terminal Server Client\Servers`; não tem senha, mas entrega a lista de alvos do usuário.

## Mapeamentos de drive

Credenciais de compartilhamento SMB salvas vão para o Credential Manager como `Domain:target=<servidor>`. Os mapeamentos persistentes ficam em `HKCU\Network\<letra>`.

## VPN

| Tipo | Local |
|---|---|
| Windows RAS (PPTP, L2TP, SSTP) | `%APPDATA%\Microsoft\Network\Connections\Pbk\rasphone.pbk` (usuário)<br>`C:\ProgramData\Microsoft\Network\Connections\Pbk\rasphone.pbk` (todos) |
| Senhas de sessão RAS | LSA Secrets (`RasCredentials`) e DPAPI |
| OpenVPN | `%USERPROFILE%\OpenVPN\config\`; arquivos `.ovpn` podem embutir `auth-user-pass` e chaves |
| WireGuard | `%PROGRAMFILES%\WireGuard\Data\Configurations\*.conf.dpapi`, com a chave privada do túnel |

## Certificados com chave privada

| Store | Caminho em disco |
|---|---|
| Usuário (CurrentUser\My) | `%APPDATA%\Microsoft\SystemCertificates\My\Certificates\`, com chaves em `%APPDATA%\Microsoft\Crypto\RSA\{SID}\` e `%APPDATA%\Microsoft\Crypto\Keys` (CNG) |
| Máquina (LocalMachine\My) | `C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\` e `C:\ProgramData\Microsoft\Crypto\Keys` |

Chaves privadas de certificado (EFS, S/MIME, autenticação de cliente) vivem nesses diretórios `Crypto\`, protegidas por DPAPI e ACLs. São o alvo de T1552.004.

## Implicações de segurança

Um `.rdp` ou `.ovpn` esquecido em pasta compartilhada equivale a deixar credencial em cima da mesa. Em análise forense, `rasphone.pbk` e os perfis Wi-Fi revelam rápido para quais redes o usuário se conecta.
