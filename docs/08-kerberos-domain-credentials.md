# 08 — Kerberos e Credenciais de Domínio

> Perspectiva dupla. Os docs anteriores cobriram segredos locais. Este cobre o que muda quando a máquina entra num domínio Active Directory: tickets Kerberos em memória, a base do AD em disco e alguns esqueletos históricos que ainda assombram redes antigas.

## Tickets Kerberos em memória

O Kerberos não guarda senha na estação. Guarda tickets.

| Item | Onde fica | Vida útil |
|---|---|---|
| TGT (Ticket Granting Ticket) | Memória do `lsass.exe`, na sessão de logon | Padrão de 10 horas, renovável |
| TGS (tickets de serviço) | Mesma região, um por serviço acessado | Padrão de 10 horas |
| Chaves de sessão | Junto aos tickets | Enquanto o ticket valer |

Não existe cache de tickets em disco no Windows, diferente do `krb5cc` do Linux. Quem quer tickets vai na memória do lsass, o que devolve o problema ao doc 02: Credential Guard e RunAsPPL são as barreiras.

Um ponto que confunde muita gente: ticket roubado vale sem a senha. Quem tem o TGT de alguém se autentica como essa pessoa até o ticket expirar (T1550.003, Pass-the-Ticket). Trocar a senha não invalida tickets já emitidos.

## krbtgt, golden e silver tickets

O TGT é criptografado com a chave da conta `krbtgt`, o KDC do domínio. Quem tem o hash dela consegue forjar TGT de qualquer conta, com qualquer privilégio, validade arbitrária. É o golden ticket (T1558.001). A variação silver (T1558.002) forja TGS com a chave de uma conta de serviço ou de máquina, sem passar pelo KDC.

Conceitualmente, os dois dependem do mesmo passo anterior: extrair hashes do AD, o que quase sempre significa o `NTDS.dit` (T1003.003) ou um DCSync contra o DC. Por isso a defesa contra golden ticket não começa no ticket. Começa em proteger o DC e a base:

- Rotação da senha da `krbtgt` duas vezes seguidas depois de qualquer suspeita de comprometimento de domínio. Duas, porque o histórico mantém a senha anterior válida.
- Tiering administrativo: conta de admin de domínio nunca loga em estação comum, então o hash dela nunca toca a memória de uma máquina comprometida.
- Protected Users: grupo que força AES, proíbe NTLM e encurta a vida do TGT para 4 horas.

## RODC

O Read-Only Domain Controller existe para filiais inseguras: ele mantém só um subconjunto de senhas, o PRP (Password Replication Policy). Comprometer um RODC expõe apenas as contas cacheadas nele, e não o domínio inteiro. Em rede antiga ou mal configurada, é comum encontrar o PRP aberto demais, com contas privilegiadas cacheadas onde não deviam.

## GPP e o cpassword (legado que insiste em aparecer)

Group Policy Preferences permitia distribuir senha de admin local via XML no SYSVOL, criptografada com uma chave AES que a Microsoft publicou em 2012. Qualquer usuário do domínio lê o SYSVOL, então qualquer um decriptava. O MS14-025 em 2014 matou a criação de novas senhas assim, mas o patch não apagou os XML antigos.

Resultado: em redes com histórico, ainda se encontra `Groups.xml` e `ScheduledTasks.xml` com `cpassword` em `\\<dominio>\SYSVOL\<dominio>\Policies\`. É T1552.006, e a checagem é barata: procurar a string `cpassword` nos XMLs do SYSVOL.

## DPAPI-NG (CNG DPAPI)

O DPAPI clássico (doc 01) deriva a chave da senha. O DPAPI-NG, usado para segredos compartilhados em domínio, protege o conteúdo com chave de grupo: qualquer membro do grupo decripta. É o mecanismo por trás de PFX protegidos para grupo e de alguns cenários de certificado corporativo. Os arquivos continuam em disco nos mesmos caminhos de `Protect\`, mas a semântica de acesso muda: comprometer uma conta do grupo compromete o segredo de todos.

## Resumo defensivo

| Ameaça | Controle |
|---|---|
| Roubo de ticket (T1550.003) | Credential Guard, Protected Users, tickets curtos |
| Golden/silver (T1558) | Proteger o DC, rotação dupla da krbtgt, tiering |
| NTDS.dit (T1003.003) | DC físico ou VM blindada, backup criptografado, alerta em acesso ao arquivo |
| cpassword em SYSVOL (T1552.006) | Varredura por `cpassword`, remoção dos XMLs, LAPS como substituto |
| PRP aberto em RODC | Revisar quais contas são cacheáveis por filial |

## Referências

- MITRE ATT&CK: T1550.003, T1558.001, T1558.002, T1003.003, T1552.006
- Microsoft: documentação de Kerberos, Protected Users, MS14-025, RODC
- Relacionado neste repositório: doc 02 (LSASS, NTDS), doc 06 (hardening), doc 07 (priorização)
