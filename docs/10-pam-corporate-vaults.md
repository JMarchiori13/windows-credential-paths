# 10 — Credenciais em Gerenciadores de Senha Corporativos (PAM)

> Perspectiva dupla. KeePass e Bitwarden pessoais já apareceram no doc 04. Este documento cobre o outro mundo: os cofres corporativos (PAM) que guardam as senhas mais poderosas da empresa, e o que acontece quando o cofre, e não a senha, vira o alvo.

## Por que o PAM muda o jogo

Num ambiente maduro, senhas de admin, de serviço e de banco não ficam mais em planilha nem em memória de sysadmin. Ficam num cofre central que controla checkout, rotação e auditoria. Isso é ótimo para a defesa e cria um efeito colateral óbvio: todo o valor da rede concentrado num único sistema.

Para o atacante, o PAM é simultaneamente o alvo mais valioso e o mais defendido. Para a defesa, é o sistema cujo comprometimento precisa ser tratado como cenário de pior caso.

## Os principais players

| Produto | Arquitetura em resumo | Onde os segredos vivem |
|---|---|---|
| CyberArk (EPV) | Vault central + componentes (PVWA, CPM, PSM) | Vault criptografado em disco no servidor dedicado; chaves do vault em HSM ou arquivo de chave separado |
| Delinea Secret Server (ex-Thycotic) | Aplicação web + banco SQL Server | Segredos criptografados no banco; chave mestra no servidor, protegida por DPAPI da máquina ou HSM |
| HashiCorp Vault | Serviço com storage backend | Dados criptografados no backend; unseal keys ou auto-unseal via KMS/cloud |
| BeyondTrust Password Safe | Appliance + console | Banco interno criptografado, chaves no appliance |

## Os pontos de atenção, um por um

### O vault e a chave mestra

Todo PAM criptografa os segredos em repouso. A pergunta útil não é "o vault é criptografado?" e sim "onde está a chave que abre o vault?". Respostas comuns: arquivo de chave no próprio servidor, DPAPI da máquina, HSM, ou KMS de cloud. As duas primeiras aparecem em auditoria com frequência preocupante.

### As contas de serviço do próprio PAM

O CPM do CyberArk rota senhas usando uma conta privilegiada. O Secret Server fala com o AD e com bancos usando contas de serviço. Essas contas são, por necessidade, poderosas, e vivem nos mesmos lugares de qualquer conta de serviço: memória de processo, configuração, LSA Secrets em alguns cenários. Comprometer o servidor do PAM costuma render as chaves do reino sem abrir o cofre.

### O checkout auditado

A maioria dos PAM entrega o segredo em texto claro para o usuário ou aplicação autorizada no momento do checkout. Nesse instante, a proteção do cofre não existe mais: a senha está na memória do cliente, na variável de ambiente, no pipeline de CI. Muita rede bem protegida já foi comprometida pelo Jenkins que tinha credencial de checkout, e não pelo vault.

### Sessões PSM

O PSM do CyberArk grava sessões privilegiadas. Essas gravações são ouro para auditoria e, se o acesso a elas for frouxo, ouro para o atacante também: é o replay de cada senha digitada.

## Ataques conceituais e sinais

| Vetor | Conceito | Sinal defensivo |
|---|---|---|
| Servidor do PAM comprometido | Chave mestra, contas de serviço e sessões ficam expostas | Servidor de PAM é ativo crítico: EDR, acesso restrito, alerta em qualquer logon incomum |
| Abuso de API de checkout | Usar credencial de aplicação legítima para puxar segredos em massa | Volume de checkout anômalo, fora de horário, de host novo |
| Roubo de conta de serviço do PAM | A conta que rotaciona senhas tem privilégio sobre todas | Mesma telemetria de conta de serviço privilegiada (docs 02 e 08) |
| Acesso às gravações de sessão | Replay de sessões privilegiadas | Auditoria de acesso às próprias gravações |

## A regra de ouro da defesa

O PAM precisa ser tratado com o mesmo rigor do DC. Na prática:

1. Servidor do vault no tier 0, com as mesmas restrições de logon do controlador de domínio.
2. Chave mestra em HSM ou KMS, nunca em arquivo ao lado do vault.
3. Contas de serviço do PAM como gMSA onde suportado, com monitoramento dedicado.
4. Alerta em padrão de checkout: volume, horário, origem. O cofre auditável só ajuda se alguém olha a auditoria.
5. Plano de resposta específico para comprometimento do PAM, porque trocar todas as senhas da empresa é exatamente o cenário para o qual ele existe, e ninguém quer descobrir isso durante o incidente.

## Referências

- Documentação CyberArk (EPV, CPM, PSM), Delinea Secret Server, HashiCorp Vault
- MITRE ATT&CK: T1555 (Credentials from Password Stores), T1552.001
- Relacionado neste repositório: doc 02 (contas de serviço), doc 04 (gerenciadores pessoais), doc 09 (tier 0)
