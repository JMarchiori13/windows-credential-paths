# Contribuindo com o Windows Credential & SSH Key Paths

Obrigado pelo interesse em contribuir! Este repositório é uma referência **viva** de caminhos de credenciais no Windows — novos apps, versões e mecanismos surgem o tempo todo, e a comunidade é bem-vinda para manter tudo atualizado.

O projeto trabalha com **dupla perspectiva**: 🔴 **Red Team** (onde coletar credenciais em engajamentos autorizados, priorização tática, técnicas ATT&CK) e 🔵 **Blue Team** (detecção, hardening, DFIR). Contribuições de ambos os lados são bem-vindas — o ideal é que cada caminho documentado sirva aos dois.

## Como contribuir

### 1. Sugerindo um novo caminho ou aplicação (Issue)

Abra uma [issue](../../issues/new) com:

- **Nome da aplicação/recurso** e versão testada
- **Caminho(s) completo(s)** — use as convenções do projeto (`%APPDATA%`, `%LOCALAPPDATA%`, etc.)
- **Formato de armazenamento** — arquivo, registro, SQLite, JSON, hive
- **Mecanismo de proteção** — DPAPI (usuário/máquina), senha mestre, Base64, texto claro, App-Bound, etc.
- **Fonte/evidência** — documentação oficial, análise própria, artigo técnico (com link)
- **Perspectiva** — relevância ofensiva (privilégio necessário, fase do engajamento) e/ou defensiva (detecção, hardening)

### 2. Enviando alterações (Pull Request)

1. Faça fork e crie um branch descritivo: `docs/add-<app>` ou `fix/corrige-<doc>`
2. Siga o formato dos documentos existentes:
   - Tabelas com colunas **Aplicação | Caminho | Proteção**
   - Variáveis de ambiente em vez de caminhos absolutos com nome de usuário
   - Seção de **implicações de segurança** — idealmente cobrindo os dois ângulos:
     - 🔴 ofensivo: privilégio necessário, ferramenta/técnica ATT&CK associada, fase do engajamento
     - 🔵 defensivo: evento de auditoria, regra de detecção, mitigação
3. Commits em português, no estilo convencional: `docs: adiciona X`, `fix: corrige caminho Y`
4. Descreva no PR o que foi adicionado/corrigido e a fonte da informação

### 3. Corrigindo erros

Caminhos mudam entre versões (ex.: Edge legado → Chromium, Chrome 127+ App-Bound). Ao corrigir:

- Indique a **versão do Windows/app** em que verificou
- Se o comportamento varia por versão, documente **ambos** em vez de substituir

## Código de conduta

Seja respeitoso e técnico. Discussões sobre ética e legalidade são bem-vindas.

## Dúvidas?

Abra uma issue com a tag `question` — respondemos assim que possível.
