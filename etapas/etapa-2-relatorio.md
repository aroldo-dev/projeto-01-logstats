# Entrega 2 — Relatório em texto

**Estimativa de referência:** 3–4h · **Seções da SPEC:** §5.1, §6.2

---

## Por que esta entrega

É aqui que o produto passa a responder a pergunta do usuário: *o que quebrou e
onde está lento?*

## Histórias

> **Como** engenheiro de plantão, **quero** ver quantas linhas há de cada nível,
> **para** ter noção do volume de problemas.

> **Como** engenheiro de plantão, **quero** saber quais módulos concentram mais
> erros, **para** saber por onde começar a investigar.

> **Como** engenheiro de plantão, **quero** ver média e percentis de latência,
> **para** saber se o problema é lentidão.

---

## Escopo

- `logstats <ARQUIVO>` imprime o relatório de SPEC §5.1 — para `sample.log`,
  **byte a byte igual** a SPEC §6.2.
- A listagem da Entrega 1 deixa de existir.
- Sem opções ainda: vale o padrão (`--top 5`, formato texto).

## Fora do escopo

Avisos no stderr, códigos 1/2/3, opções de linha de comando, stdin, JSON.

## Regras de negócio que entram

SPEC §5.1 inteira. Os pontos em que o QA mais encontra defeito:

- **Percentil é *nearest-rank*, sem interpolação.** Uma fórmula de percentil
  "de planilha" dá números diferentes; U30 e U31 pegam.
- **A média arredonda metades para longe do zero.** U33 (`[1, 1, 2]` → `1.3`) é
  o caso de borda.
- **Ranking:** mais erros primeiro; no empate, nome de A→Z.
- `by_level` mostra os 5 níveis sempre, mesmo zerados.
- Ranking vazio → `  (none)`. Sem amostras → `  (no samples)`.
- Alinhamento calculado **por bloco**, não pelo relatório inteiro.

Requisitos não funcionais que já valem:

- **RNF5 (determinismo):** o QA vai rodar a mesma execução várias vezes e
  comparar. Saída que muda de ordem entre execuções é defeito, mesmo que "às
  vezes passe".
- **RNF2 (memória limitada):** só é medido na Entrega 5 (E14), mas vale a partir
  daqui. Se esta entrega não atender, o retrabalho aparece lá.

---

## Critérios de aceite

- [ ] O stdout de `logstats tests/fixtures/sample.log` é idêntico, byte a byte, a
      `expected/sample.stdout.txt` — comparado com uma ferramenta de diff, não a olho
- [ ] O código de saída é 0
- [ ] 10 execuções seguidas produzem 10 saídas idênticas
- [ ] Os números conferem com a conta de SPEC §6.2 (soma 3662, n = 12, média `305.2`)

## Casos de teste (SPEC §7)

| Grupo | IDs | Nível |
|---|---|---|
| Estatísticas | **U24–U34** | unitário |
| Formatação da saída | **U49–U51** | unitário |
| Fluxo completo | **I01, I12, I13** | integração |
| Caso canônico | **E01** — só stdout e código de saída; o stderr entra na Entrega 3 | ponta a ponta |

Sobre os testes de integração: eles exercitam o fluxo **sem passar pela linha de
comando e sem arquivos em disco** (SPEC §7). É um requisito de testabilidade do
QA; como tornar isso possível é decisão de design sua.

---

## Definition of Done

- [ ] Critérios de aceite atendidos
- [ ] Casos desta entrega automatizados e verdes
- [ ] Casos da Entrega 1 continuam verdes
- [ ] Q01 e Q02 atendidos

## Pacote de entrega para o QA

1. Saída completa da suíte de testes
2. Saída da verificação de formatação e do linter
3. Resultado do diff entre o stdout e `expected/sample.stdout.txt` (tem que sair vazio)
4. Evidência das 10 execuções idênticas
5. Limitações conhecidas, se houver
