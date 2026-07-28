# 12 — Segredos em Cloud Híbrida: MSAL, Managed Identities e o Token Broker

> Perspectiva dupla. A fronteira de credenciais do Windows não termina no disco local. Quase toda estação corporativa moderna conversa com Entra ID o tempo todo, e os tokens dessa conversa vivem em lugares que nem todo time de segurança conhece.

## O cache MSAL

A Microsoft Authentication Library é a camada que quase todo app moderno (Office, Teams, Edge, Azure CLI) usa para autenticar no Entra ID. Ela mantém um cache local de tokens de acesso e, mais importante, de **refresh tokens**.

| Item | Onde fica | Observação |
|---|---|---|
| Cache MSAL dos apps | `%LOCALAPPDATA%\Microsoft\IdentityCache\` e variações por app | Binário, protegido por DPAPI do usuário |
| Cache do Azure CLI | `%USERPROFILE%\.azure\msal_token_cache.bin` | DPAPI do usuário |
| Tokens do Office | Credential Manager, alvos `MicrosoftOffice16_Data:ADAL:*` | DPAPI |

O refresh token é o prêmio. Token de acesso expira em cerca de uma hora; refresh token vale por dias ou semanas e fabrica tokens de acesso novos sob demanda, sem senha e, dependendo da política condicional, sem novo MFA. Roubo de refresh token (T1528) é um dos vetores de account takeover corporativo mais eficazes da atualidade.

## Primary Refresh Token (PRT)

O PRT é o irmão mais poderoso: o token de SSO do Windows em si, emitido no momento em que o usuário loga numa máquina Entra-joined ou hybrid-joined. Com ele, o usuário acessa recursos de cloud sem digitar nada. O PRT é protegido por chave vinculada ao TPM quando há TPM, e vive sob a guarda do `lsass` e do CloudAP plugin.

A defesa do PRT é basicamente a defesa do lsass: Credential Guard, RunAsPPL, TPM. Quando essas camadas caem, o PRT entra na mesa do atacante, e com ele o SSO inteiro do usuário.

## Managed identities e o endpoint IMDS

Em VMs e serviços Azure, a managed identity elimina senha de aplicação: o recurso pede token diretamente à plataforma, via endpoint de metadados (`169.254.169.254`). Não há segredo para vazar em arquivo. O que há é um endpoint que responde token a qualquer processo local que pergunte direito.

A consequência prática: SSRF e execução local em VM com managed identity valem token de cloud. A defesa é escopo mínimo na identidade e, quando possível, o modo de endpoint com header obrigatório e política de acesso restrita.

## O que muda na coleta (ofensivo)

| Alvo | Esforço | Rendimento |
|---|---|---|
| Cache MSAL do usuário | Leitura de perfil + DPAPI do próprio usuário | Refresh tokens de cloud corporativa |
| PRT | Requer SYSTEM, TPM no caminho | SSO completo do usuário |
| IMDS em VM | Execução local ou SSRF | Token da identidade gerenciada |
| Service connections do pipeline | Ver doc 11 | SPN com papel em subscription |

O padrão se repete: a fronteira endpoint-cloud é onde a coleta de credencial moderna mais rende, porque o token de cloud ignora toda a segurança de endpoint tradicional. O EDR não vê login de cloud feito de outro continente com um refresh token legítimo.

## A defesa na fronteira

1. Conditional Access com avaliação contínua (CAE), para revogação quase em tempo real de token comprometido.
2. Token binding onde disponível, que amarra o token ao dispositivo e mata o replay de outra máquina.
3. Alerta em logon impossível: mesmo refresh token de dois lugares distantes, IP novo, dispositivo novo.
4. Managed identity com escopo mínimo, e preferência por identidade atribuída pelo sistema, que morre junto com o recurso.
5. Inventário de quais apps mantêm cache MSAL em cada estação, tratado como o equivalente moderno do doc 04.

## Referências

- Microsoft: documentação de MSAL, PRT, CloudAP, managed identities, CAE
- MITRE ATT&CK: T1528 (Steal Application Access Token), T1550.001, T1557
- Relacionado neste repositório: doc 02 (lsass, Credential Guard), doc 08 (Kerberos), doc 11 (CI/CD)
