# Contribuindo com o Windows Credential & SSH Key Paths

Obrigado pelo interesse em contribuir! Este repositório é uma referência **viva** de caminhos de credenciais no Windows — novos apps, versões e mecanismos surgem o tempo todo, e a comunidade é bem-vinda para manter tudo atualizado.

## Como contribuir

### 1. Sugerindo um novo caminho ou aplicação (Issue)

Abra uma [issue](../../issues/new) com:

- **Nome da aplicação/recurso** e versão testada
- **Caminho(s) completo(s)** — use as convenções do projeto (`%APPDATA%`, `%LOCALAPPDATA%`, etc.)
- **Formato de armazenamento** — arquivo, registro, SQLite, JSON, hive
- **Mecanismo de proteção** — DPAPI (usuário/máquina), senha mestre, Base64, texto claro, App-Bound, etc.
- **Fonte/evidência** — documentação oficial, análise própria, artigo técnico (com link)

### 2. Enviando alterações (Pull Request)

1. Faça fork e crie um branch descritivo: `docs/add-<app>` ou `fix/corrige-<doc>`
2. Siga o formato dos documentos existentes:
   - Tabelas com colunas **Aplicação | Caminho | Proteção**
   - Variáveis de ambiente em vez de caminhos absolutos com nome de usuário
   - Seção de **implicações de segurança** quando relevante (ângulo defensivo)
3. Commits em português, no estilo convencional: `docs: adiciona X`, `fix: corrige caminho Y`
4. Descreva no PR o que foi adicionado/corrigido e a fonte da informação

### 3. Corrigindo erros

Caminhos mudam entre versões (ex.: Edge legado → Chromium, Chrome 127+ App-Bound). Ao corrigir:

- Indique a **versão do Windows/app** em que verificou
- Se o comportamento varia por versão, documente **ambos** em vez de substituir

## Regras de conteúdo

✅ **Aceito:**
- Caminhos de armazenamento de credenciais, chaves, tokens e certificados
- Mecanismos de proteção e suas fraquezas conhecidas (documentadas publicamente)
- Detecção, hardening, auditoria e resposta a incidentes
- Referências MITRE ATT&CK, Sigma, Sysmon

❌ **Não aceito:**
- Exploits prontos, PoCs de extração ou ferramentas de credential dumping
- Instruções passo a passo para obter credenciais de terceiros
- Dados reais de credenciais, mesmo "de exemplo"
- Conteúdo sem fonte ou não verificável

> A linha é: **documentar onde as coisas estão e como defendê-las** — não como roubá-las. Este material existe para Blue Team, DFIR e hardening.

## Código de conduta

Seja respeitoso e técnico. Discussões sobre ética e legalidade são bem-vindas; conteúdo que viole o aviso legal do README será removido.

## Dúvidas?

Abra uma issue com a tag `question` — respondemos assim que possível.
