# 14 — AD CS: Certificados como Credencial de Domínio (ESC1–ESC8)

> Perspectiva dupla. O Active Directory Certificate Services era aquele serviço que ninguém olhava, até a pesquisa Specterops (Certified Pre-Owned, 2021) mostrar que ele é um segundo sistema de autenticação do domínio, com credenciais próprias e quase nenhuma auditoria. Este documento cobre os caminhos e os pontos fracos conceituais.

## Por que AD CS importa

Certificado de cliente no AD CS autentica como a conta. Quem tem um certificado válido de usuário ou de máquina se autentica no domínio sem senha, e o certificado vale enquanto valer, independentemente de troca de senha. É uma credencial paralela que a maioria das equipes não inventaria, não monitora e não revoga.

## Onde os segredos vivem

| Item | Caminho | Observação |
|---|---|---|
| Certificados do usuário | `CurrentUser\My` e chaves em `%APPDATA%\Microsoft\Crypto\` (doc 05) | DPAPI do usuário |
| Certificados de máquina | `LocalMachine\My` e `C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\` | Alvo de roubo para persistência como máquina |
| Chave privada da CA | No servidor da CA, protegida por DPAPI de máquina ou HSM | Comprometer a CA é comprometer a confiança do domínio |
| Banco da CA | Configuração e templates no AD (`CN=Configuration,CN=...\Services\Public Key Services`) | Legível por qualquer usuário do domínio |
| Certificados emitidos | Solicitação via RPC/DCOM ou web enrollment (`/certsrv`) | Vetor das ESCs |

## As classes de abuso (conceitual)

O catálogo ESC vem da pesquisa original. Em resumo honesto:

| Classe | O problema | Resultado |
|---|---|---|
| ESC1 | Template permite que o solicitante defina o nome alternativo (SAN) e tenha uso de autenticação | Qualquer um emite certificado se passando por qualquer conta, incluindo admin |
| ESC2 | Template com uso irrestrito (any purpose) | Certificado coringa |
| ESC3 | Template de enrollment agent usado para emitir em nome de outros | Escalação indireta |
| ESC4 | ACL fraca no template permite modificá-lo | O atacante transforma o template num ESC1 |
| ESC6 | Flag de edição de SAN ligada na própria CA | Toda emissão pode mentir o nome |
| ESC7 | ACL fraca na CA (ManageCA/ManageCertificates) | Controle da aprovação de certificados |
| ESC8 | Web enrollment exposto a relay NTLM | Coerção + relay vira certificado de máquina do DC |

## Roubo de certificado existente

Além de abusar de templates, há o caminho direto: roubar o certificado com chave privada que já existe no host (docs 01 e 05 deste repositório). Um certificado de usuário roubado é persistência elegante: não aparece em reset de senha, sobrevive a troca de senha e autentica como a vítima até expirar. Revogação de certificado é processo que muita equipe nunca executou na vida.

## Defesa

1. Inventariar templates: SAN editável, uso any-purpose, enrollment agent e ACLs fracas são os quatro achados que importam.
2. Web enrollment exposto ou desnecessário merece revisão imediata, por causa do ESC8.
3. Monitorar emissões: evento 4886/4887 da CA (solicitação/emissão) fora do fluxo normal.
4. Tratar certificado como credencial: inventário, validade, revogação testada, e alerta em autenticação por certificado fora do padrão.
5. Proteger a chave da CA como segredo de tier 0, preferencialmente em HSM.

## Referências

- Specterops, "Certified Pre-Owned" (2021), a pesquisa fundadora do tema
- MITRE ATT&CK: T1552.004, T1649 (certificados roubados/forjados)
- Relacionado neste repositório: doc 05 (MachineKeys), doc 08 (Kerberos), doc 09 (tier 0)
