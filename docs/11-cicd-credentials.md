# 11 — Credenciais em CI/CD

> Perspectiva dupla. O doc 04 cobriu a estação do desenvolvedor. Este cobre a fábrica: os pipelines que constroem e publicam software, e a quantidade assustadora de segredo que circula por eles.

## Por que CI/CD virou alvo de primeira linha

Um pipeline precisa de credencial para quase tudo: clonar código, assinar binário, publicar pacote, subir para cloud, reiniciar serviço. Soma-se a isso que o pipeline executa código de terceiros o tempo todo, em pull request, dependência e plugin. O resultado é um sistema altamente privilegiado que roda conteúdo semicoonfiável por definição.

Nos últimos anos, alguns dos maiores incidentes de supply chain passaram exatamente por aí.

## Onde os segredos moram

| Sistema | Local dos segredos | Observação |
|---|---|---|
| GitHub Actions | Secrets do repo/org, `GITHUB_TOKEN` automático | O token automático tem escopo configurável; o padrão é mais largo do que devia |
| GitLab CI | CI/CD Variables | Variável sem "masked" e sem "protected" vaza em log e roda em qualquer branch |
| Azure DevOps | Variable groups, service connections | Service connection costuma ser SPN com papel largo na subscription |
| Jenkins | Credentials store, `JENKINS_HOME/credentials.xml` | Criptografado com chave mestra no próprio servidor |
| Arquivos de configuração | `.github/workflows/`, `.gitlab-ci.yml`, `azure-pipelines.yml` | Segredo hardcoded em YAML é o erro clássico |
| Logs de build | Saída do pipeline | Segredo impresso em log persiste para sempre e para todos |

## O self-hosted runner, o parente pobre da segurança

Runner hospedado pelo fornecedor é efêmero e isolado. Runner self-hosted é uma máquina sua, que roda código de PR, e que frequentemente tem acesso à rede interna, ao registry e à cloud. Um PR malicioso num repo público com runner self-hosted é execução de código dentro da empresa, patrocinada pelo próprio pipeline.

Regra prática: runner self-hosted nunca roda PR de fork, nunca tem acesso desnecessário à rede interna, e é tão efêmero quanto possível.

## O token automático

O `GITHUB_TOKEN` e equivalentes existem para o pipeline interagir com o próprio repositório. O problema é o padrão: em muitos setups, ele escreve no repo, aprova PR e lê secrets. Um step comprometido com esse token modifica o código-fonte e os próprios workflows, o que é persistência dentro do pipeline. A correção é escopo mínimo por job, declarado no YAML.

## Assinatura de código e de pacote

O pipeline que assina binários guarda a chave de assinatura, ou acessa um serviço de assinatura. Essa chave é o selo de confiança de toda a distribuição do software. Comprometê-la não é roubar uma credencial; é fabricar produto legítimo falso. Merece HSM ou serviço de assinatura remota, nunca arquivo no runner.

## Vetores conceituais e sinais

| Vetor | Conceito | Sinal defensivo |
|---|---|---|
| Segredo em YAML/log | Hardcoded ou impresso em build | Varredura de secrets no repo e nos logs (gitleaks, trufflehog) |
| PR malicioso em runner self-hosted | Execução na rede interna via pipeline | Política: fork não roda, runner efêmero, egress restrito |
| Token automático abusado | Step comprometido altera repo e workflows | Escopo mínimo, revisão de workflow em PR, alerta em push de workflow |
| Dependência envenenada | Pacote malicioso executa em install/build | Lockfile, verificação de integridade, egress de build restrito |
| Service connection larga | SPN do pipeline é dono da subscription | Papel mínimo por conexão, alerta em uso fora do pipeline |

## Contrapartida defensiva

1. Segredo só em store de secrets, com masking e proteção por branch/ambiente. Nunca em YAML, nunca em variável comum.
2. Escopo mínimo para todo token automático, declarado por job.
3. Runner self-hosted isolado, efêmero, sem PR de fork, com egress filtrado.
4. Varredura de secrets contínua no histórico do repo e nos logs de build.
5. Assinatura em HSM ou serviço remoto, fora do runner.
6. Revisar mudança de workflow com o mesmo rigor de mudança de código, porque workflow é código com credencial.

## Referências

- MITRE ATT&CK: T1552.001, T1195 (Supply Chain Compromise), T1072 (ferramentas de deploy)
- Documentação GitHub Actions, GitLab CI, Azure DevOps sobre hardening de pipeline
- Relacionado neste repositório: doc 04 (estação de dev, `.git-credentials`), doc 07 (fase 1 da coleta)
