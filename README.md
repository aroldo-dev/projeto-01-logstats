# Projeto 01 — `logstats`

Ferramenta de linha de comando que lê um log estruturado e emite um relatório
determinístico: contagens por nível, ranking de módulos com mais erros e
percentis de latência.

**Nível:** básico · **Estimativa de referência:** 13–18h em 5 entregas

---

## Papéis

| Papel | Quem | Responsável por |
|---|---|---|
| **Produto** | Claude | *O quê* e *para quem*: problema, histórias, regras de negócio, critérios de aceite |
| **QA** | Claude | *Como verificar*: plano de testes, massa de teste, saídas esperadas, aceite de cada entrega |
| **Dev** | você | *Como construir*: arquitetura, código, testes automatizados, escolhas técnicas |
| **Tutor** | Claude, no chat, quando você pedir | Dúvidas de Rust e revisão de código — regras em [`CLAUDE.md`](CLAUDE.md) |

Os documentos deste repositório são de Produto e QA. De propósito, eles **não**
trazem nomes de funções, tipos, módulos ou arquivos de código, nem dicas de
implementação: essas decisões são suas. Se travar, pergunte ao tutor.

---

## Comece por aqui

1. Leia a [`SPEC.md`](SPEC.md) inteira, uma vez, sem implementar nada. É o contrato:
   o QA valida o comportamento descrito lá, não uma implementação.
2. Volte e leia só a [Entrega 1](etapas/etapa-1-classificacao.md).
3. Construa a Entrega 1, feche a *Definition of Done* e envie o **pacote de entrega**
   ([SPEC §11](SPEC.md#11-fluxo-de-entrega-e-aceite)). O QA aprova ou devolve bugs;
   com a aprovação, Produto libera a próxima.

**Uma entrega por vez.** O retorno de cada entrega evita que você repita o mesmo
defeito nas quatro seguintes.

---

## As 5 entregas

Cada entrega deixa o produto **funcionando de ponta a ponta**, com mais capacidade
que a anterior.

| # | Entrega | O que o usuário ganha | Casos de teste | ~h |
|---|---|---|---|---|
| 1 | [Classificação de linhas](etapas/etapa-1-classificacao.md) | vê cada linha classificada: válida, inválida (com motivo) ou em branco | U01–U23, U55 | 3–4 |
| 2 | [Relatório em texto](etapas/etapa-2-relatorio.md) | o relatório completo: níveis, ranking de erros, latência | U24–U34, U49–U51, I01, I12–I13, E01 | 3–4 |
| 3 | [Entradas problemáticas, avisos e códigos de saída](etapas/etapa-3-robustez.md) | arquivos do Windows, corrompidos ou vazios; stdin; avisos; códigos 0/2/3 | U17–U19, I07–I11, E01, E03–E04, E08–E10 | 2–3 |
| 4 | [Opções de linha de comando e filtros](etapas/etapa-4-opcoes-e-filtros.md) | filtros por nível, módulo e tempo; ajuda; versão; código 1 | U35–U48, I02–I06, E05–E07, E11–E13, E15 | 3–4 |
| 5 | [Saída JSON, requisitos não funcionais e aceite final](etapas/etapa-5-json-e-aceite-final.md) | JSON para a pipeline; arquivos enormes sem estourar memória | U52–U54, E02, E14, P01–P04, Q01–Q05 | 2–3 |

O escopo final é exatamente o da SPEC: nada foi cortado, só ordenado.

---

## Massa de teste (fornecida pelo QA)

Em `tests/fixtures/`. Os números abaixo foram conferidos contra a SPEC e são o
que o `logstats` tem que produzir.

| Arquivo | Cobre | total / válidas / inválidas / em branco | código | Entrega |
|---|---|---|---|---|
| `sample.log` | caso canônico | 20 / 16 / 3 / 1 | 0 | 1, 2, 3 |
| `empty.log` | 0 bytes | 0 / 0 / 0 / 0 | 0 | 3 |
| `all_invalid.log` | 5 inválidas, motivos variados | 5 / 0 / 5 / 0 | **3** | 3, 4 |
| `crlf.log` | terminadores CRLF | 5 / 4 / 0 / 1 | 0 | 3 |
| `no_trailing_newline.log` | última linha sem quebra de linha | 3 / 3 / 0 / 0 | 0 | 3 |
| `utf8_mixed.log` | acento, emoji e **bytes UTF-8 inválidos** na linha 2 | 4 / 3 / 1 / 0 | 0 | 3 |
| `lo g ção.log` | caminho com espaço e acento | 2 / 2 / 0 / 0 | 0 | 4 |

Motivos esperados, por linha:

| Arquivo | stderr |
|---|---|
| `sample.log` | `line 8: too_few_tokens`, `line 11: bad_level`, `line 15: bad_duration` |
| `all_invalid.log` | `line 1: too_few_tokens`, `line 2: bad_timestamp`, `line 3: bad_level`, `line 4: bad_module`, `line 5: bad_duration` |
| `utf8_mixed.log` | `line 2: invalid_utf8` — e as linhas 3 e 4 **continuam sendo processadas** |

## Saídas esperadas (oráculos)

Em `expected/`. Consideram que o programa foi chamado com o caminho literal
`tests/fixtures/sample.log`: a linha `file:` (texto) e o campo `"file"` (JSON)
mostram exatamente o caminho informado.

| Arquivo | Comando | Entrega |
|---|---|---|
| `sample.stdout.txt` | `logstats tests/fixtures/sample.log` | 2 |
| `sample.stderr.txt` | o mesmo, canal stderr | 3 |
| `sample.json.txt` | `logstats --format json tests/fixtures/sample.log` | 5 |

Código de saída esperado nos três: `0`.

## Integridade da massa de teste

A massa de teste precisa chegar **byte a byte** como o QA a entregou. Se o controle
de versão converter os finais de linha, `crlf.log` perde os CRLF e o I10 passa sem
testar nada; e as saídas esperadas deixam de bater byte a byte.

Depois de clonar, confira: `crlf.log` com terminadores CRLF, e os arquivos de
`expected/` só com LF.
