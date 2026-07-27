# 05 — Credenciais de Rede (Wi-Fi, RDP, VPN, Drives)

## Perfis Wi-Fi

**Caminho**: `C:\ProgramData\Microsoft\Wlansvc\Profiles\Interfaces\{InterfaceGUID}\{ProfileGUID}.xml`

- O elemento `<keyMaterial>` contém a senha WPA/WPA2 **criptografada com DPAPI em escopo de máquina**.
- Enumerável sem acesso ao disco:

```cmd
netsh wlan show profiles
netsh wlan show profile name="SSID" key=clear
```

## Credenciais RDP salvas

- Ficam no **Credential Manager** como alvos `TERMSRV/<host>` (ver doc 01).
- `.rdp` files podem conter `password 51:b:<blob DPAPI hex>` quando salvos manualmente — comuns em `Documents`, `Desktop` e pastas compartilhadas.
- Histórico de conexões: `HKCU\Software\Microsoft\Terminal Server Client\Servers` (e `Default\MRU*`) — não tem senha, mas revela alvos.

## Mapeamentos de drive / rede

- Credenciais de compartilhamentos SMB salvos vão para o Credential Manager como `Domain:target=<servidor>` ou `<servidor>`.
- Mapeamentos persistentes: `HKCU\Network\<letra>`.

## VPN

| Tipo | Local |
|---|---|
| Windows RAS (PPTP/L2TP/SSTP) | `%APPDATA%\Microsoft\Network\Connections\Pbk\rasphone.pbk` (usuário)<br>`C:\ProgramData\Microsoft\Network\Connections\Pbk\rasphone.pbk` (todos) |
| Senhas de sessão RAS | LSA Secrets (`RasCredentials`) / DPAPI |
| OpenVPN | `%USERPROFILE%\OpenVPN\config\` — arquivos `.ovpn` podem embutir `auth-user-pass` e chaves |
| WireGuard | `%PROGRAMFILES%\WireGuard\Data\Configurations\*.conf.dpapi` (DPAPI) — **chave privada do túnel** |

## Certificados com chave privada

| Store | Caminho em disco |
|---|---|
| Usuário (CurrentUser\My) | `%APPDATA%\Microsoft\SystemCertificates\My\Certificates\` + chaves em `%APPDATA%\Microsoft\Crypto\RSA\{SID}\` e `%APPDATA%\Microsoft\Crypto\Keys` (CNG) |
| Máquina (LocalMachine\My) | `C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\` e `C:\ProgramData\Microsoft\Crypto\Keys` |

- Chaves privadas de certificados (EFS, smart-card virtual, S/MIME, client auth) vivem nesses diretórios `Crypto\`, protegidas por DPAPI e ACLs — alvo de roubo de certificados (T1552.004).

## Implicações de segurança

- `.rdp` e `.ovpn` esquecidos em pastas compartilhadas equivalem a credenciais deixadas na mesa.
- Auditar `rasphone.pbk` e perfis Wi-Fi em imagens forenses revela rapidamente redes-alvo do usuário.
