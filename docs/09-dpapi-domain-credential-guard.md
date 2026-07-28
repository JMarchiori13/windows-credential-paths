# 09 — DPAPI de Domínio e Credential Guard

> Perspectiva dupla. O doc 01 explicou o DPAPI com a chave derivada da senha do usuário. Em domínio, a história tem um capítulo a mais: existe uma chave-mestra do AD capaz de abrir as masterkeys de qualquer usuário do domínio. E o Credential Guard é a resposta da Microsoft para o cenário em que essa cadeia inteira é atacada.

## A chave de backup DPAPI do domínio

Quando um usuário de domínio precisa recuperar um blob DPAPI depois de trocar a senha, o Windows usa um par de chaves RSA chamado de chave de backup DPAPI do domínio. O funcionamento, em resumo:

1. Toda masterkey DPAPI de usuário de domínio é protegida duas vezes: pela senha do usuário e pela chave pública de backup do domínio.
2. A chave privada correspondente fica guardada no AD, como segredo dos controladores de domínio.
3. Com ela, qualquer masterkey de qualquer usuário do domínio pode ser recuperada, de qualquer época.

Essa chave não rotaciona sozinha. Ela nasce com o domínio e, na prática, vive para sempre.

A implicação é direta: comprometer o AD não significa só hashes de senha. Significa a capacidade de decriptar todo blob DPAPI de todo usuário do domínio, incluindo credenciais salvas, chaves privadas de certificado e cookies de navegador protegidos por DPAPI. É um dos segredos de maior valor agregado que existem numa rede Windows, e raramente aparece em planos de rotação.

## Credential Guard: o que muda

O Credential Guard usa virtualização (VSM) para isolar os segredos de logon num processo separado, o LSAiso, que roda num mundo que nem o kernel do sistema operacional principal enxerga.

| Sem Credential Guard | Com Credential Guard |
|---|---|
| Hashes NTLM e tickets Kerberos vivem na memória do lsass | Segredos de logon vivem no LSAiso, fora do alcance do lsass |
| Dump do lsass entrega tudo (T1003.001) | Dump do lsass entrega muito pouco: tickets parciais e metadados |
| Pass-the-Hash trivial com hash extraído | Hash não sai do mundo isolado |

### O que o Credential Guard não faz

Aqui mora a confusão mais comum. O Credential Guard protege segredos de autenticação de domínio em memória. Ele não protege:

- Masterkeys DPAPI em disco, nem a chave de backup do domínio no AD.
- Credenciais salvas no Credential Manager, que seguem o modelo do doc 01.
- Tokens de aplicação e cookies, que vivem no espaço do usuário.
- Contas locais, que não passam pelo LSAiso.

Ou seja: um atacante com SYSTEM numa máquina com Credential Guard ainda lê arquivos, ainda acessa o SAM, ainda usa os blobs DPAPI do próprio usuário logado. O que ele perde é a mina de hashes e tickets da memória.

## A cadeia completa, ofensivamente

Juntando os docs em uma narrativa que aparece em engajamentos reais:

1. Fase 1 do doc 07: blobs DPAPI do usuário atual, navegadores, `.ssh`, tokens de aplicação. Credential Guard não muda nada aqui.
2. Elevação: SAM local, LSA Secrets, MachineKeys. Credential Guard não muda nada aqui também.
3. Memória do lsass: com Credential Guard, a colheita cai drasticamente. Sem ele, é a fase mais rica.
4. Domínio: NTDS.dit, e com ele hashes do AD inteiro mais a chave de backup DPAPI. Nesse ponto, todo blob DPAPI de todo usuário abre, passado e futuro.

A defesa quebra essa cadeia em elos diferentes: Credential Guard no elo 3, tiering e Protected Users no elo 4, e atenção especial à chave de backup DPAPI, que não tem rotação automática e quase nunca entra em plano de resposta.

## LSAiso, TPM e atestado

O LSAiso conversa com o TPM para selar segredos de longo prazo, e o Credential Guard suporta atestado de integridade: o DC pode verificar se o LSAiso está íntegro antes de confiar nele. Em engajamento, tentar mexer no LSAiso é barulhento e exige violar a cadeia de boot segura, o que o coloca fora do alcance da maioria dos cenários de pós-exploração comuns.

## Resumo defensivo

| Ameaça | Controle |
|---|---|
| Dump de credenciais da memória (T1003.001) | Credential Guard + RunAsPPL |
| Roubo de tickets (T1550.003) | Credential Guard, Protected Users, tickets curtos |
| Chave de backup DPAPI do domínio | Tratar como segredo de nível krbtgt: inventário, acesso auditado, rotação planejada após incidente |
| NTDS.dit + backup key juntos | Tiering, DC blindado, backup criptografado, alerta em acesso |
| Falsa sensação de segurança | Credential Guard não cobre DPAPI em disco, SAM, nem tokens de aplicação |

## Referências

- Microsoft: documentação de Credential Guard, DPAPI backup key, VSM e LSAiso
- MITRE ATT&CK: T1003.001, T1003.003, T1550.003, T1555.004
- Relacionado neste repositório: doc 01 (DPAPI), doc 02 (LSASS), doc 07 (priorização), doc 08 (Kerberos)
