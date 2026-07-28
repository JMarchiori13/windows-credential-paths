# 15 — Playbook DFIR: Resposta a Incidente com este Repositório

> Perspectiva defensiva, operacional. Quando o alerta toca, ninguém lê documentação conceitual. Este playbook condensa os 14 docs anteriores em procedimento: o que coletar, em que ordem, e o que procurar em cada fase de um incidente de comprometimento de credenciais.

## Fase 0: antes do incidente

O playbook só funciona se o terreno estiver preparado. O mínimo:

- Sysmon com EID 1, 3, 8, 10, 11, 13 encaminhado (doc 06).
- Script Block Logging (4104) ativo.
- Canary files plantados nos caminhos da fase 1 (doc 13).
- Baseline de Autoruns e de logon administrativo (docs 12 e 13 do repo irmão).
- Inventário de onde ficam PAM, DC e CA (docs 09, 10, 14).

## Fase 1: triagem (primeira hora)

| Pergunta | Onde responder | Doc |
|---|---|---|
| O que disparou? | Alerta, EID correspondente, linha de comando | 06 |
| Houve acesso a credencial? | EID 11 nos caminhos da fase 1 do doc 07 | 07, 13 |
| Houve dump de lsass ou hive? | EID 10 em lsass, reg save, shadow copy | 02 |
| O que saiu da máquina? | EID 3, proxy, razão upload/download | 14 do repo irmão |

Decisão da triagem: credencial tocada sim/não. Se sim, o incidente é de credencial até prova do contrário, e a fase 2 vira prioridade sobre imagem forense completa.

## Fase 2: contenção de credencial (primeiras 24h)

Ordem importa. Trocar senha na sequência errada deixa o atacante refazer o estrago com o que ainda tem na mão.

1. **Isolar o host sem desligar**, se possível. Memória ainda tem evidência.
2. **Inventariar o que foi lido** nos caminhos afetados: `.ssh`, cloud CLIs, navegadores, Credential Manager, MSAL.
3. **Revogar tokens primeiro**: sessões de Discord/Slack/Teams, refresh tokens (revogação no Entra), tokens de cloud CLI. Token vive independente de senha.
4. **Trocar senhas tocadas**: usuário, depois contas de serviço encontradas, depois admin local (LAPS ajuda aqui).
5. **Se lsass ou hive foram tocados**: assumir hashes da memória e locais comprometidos. Logons interativos de admin naquela máquina entram no escopo.
6. **Se DC ou NTDS foram tocados**: incidente de domínio. Rotação dupla da krbtgt, revisão da chave de backup DPAPI, e o plano de resposta de PAM (doc 10).
7. **Se CA ou templates foram tocados**: inventário de certificados emitidos no período, revogação dos suspeitos (doc 14).

## Fase 3: erradicação

| Item | Ação |
|---|---|
| Persistência | Autoruns contra o baseline; revisar serviços, tasks, Run keys, IFEO, WMI |
| Implantes em memória | Scanner (PE-sieve/Moneta) antes de reimagem, para documentar |
| Reimagem | Hosts com credencial privilegiada comprometida voltam de imagem limpa, não de limpeza manual |
| Verificação pós-limpeza | Canary files novos, monitoramento reforçado por 30 dias |

## Fase 4: lições

- Qual elo da cadeia funcionou para o atacante, e qual hunt do doc 13 teria pego antes?
- Qual canary deveria existir e não existia?
- O que entra no baseline novo?

## Erros clássicos de resposta

| Erro | Consequência |
|---|---|
| Trocar senha antes de revogar tokens | Atacante continua dentro com refresh token |
| Reimagem rápida demais | Evidência de memória e persistência perdidas |
| Esquecer OST/PST exfiltrados | Caixa de e-mail continua lida depois do incidente "resolvido" |
| Não revisar emissões de certificado | Persistência por certificado sobrevive a todos os resets |
| Não rotacionar krbtgt duas vezes | Golden ticket continua válido |

## Referências

- Todos os docs deste repositório, em especial 06, 07 e 13
- NIST SP 800-61 (handling de incidentes), MITRE ATT&CK
