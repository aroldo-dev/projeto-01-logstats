# Projeto 01 — `logstats`

CLI em Rust, **zero dependências**, que lê um log estruturado e emite um relatório
determinístico: contagens por nível, ranking de módulos por erro e percentis de latência.

**Nível:** básico · **Estimativa total:** 13–18h em 5 etapas
**Papéis:** eu sou produto + QA (spec e plano de testes). Você é o dev.

---

## Comece por aqui

1. Leia a [`SPEC.md`](SPEC.md) inteira uma vez, sem tentar implementar nada. É o
   contrato — os testes validam ela, não a sua implementação.
2. Volte e leia só [`etapas/etapa-1-parser.md`](etapas/etapa-1-parser.md).
3. Implemente a Etapa 1. Quando o *Definition of Done* dela estiver fechado, me
   manda o que está no fim do arquivo. Eu reviso e libero a Etapa 2.

**Uma etapa por vez.** Não adiante a próxima antes da revisão — metade do valor
está em ver o que eu vou apontar antes de você repetir o mesmo padrão cinco vezes.

---

## As 5 etapas

Cada etapa entrega um binário que **roda de ponta a ponta**. Você nunca fica horas
sem nada funcionando.

| # | Etapa | O que passa a funcionar | Testes | ~h |
|---|---|---|---|---|
| 1 | [Parser de linha + eco](etapas/etapa-1-parser.md) | classifica cada linha: válida / inválida (motivo) / em branco | U01–U23, D01–D02 | 3–4 |
| 2 | [Agregação e relatório texto](etapas/etapa-2-agregacao.md) | o relatório da SPEC §7.2, byte a byte | U24–U34, U49–U51, I01, I12–I13, D03, E01 | 3–4 |
| 3 | [Robustez, stderr e exit codes](etapas/etapa-3-robustez.md) | CRLF, UTF-8 inválido, arquivo vazio, stdin, exits 0/2/3 | U17–U19, I07–I11, E01, E03–E04, E08–E10 | 2–3 |
| 4 | [CLI completa e filtros](etapas/etapa-4-cli.md) | todas as flags da SPEC §5, filtros, exit 1 | U35–U48, I02–I06, E05–E07, E11–E13 | 3–4 |
| 5 | [JSON, performance e gates](etapas/etapa-5-json-e-gates.md) | `--format json`, streaming provado, Q01–Q06 | U52–U54, E02, E14, D04, Q01–Q06, P01–P04 | 2–3 |

O escopo final é exatamente o da SPEC — nada foi cortado, só ordenado.

---

## Montando o repositório

```bash
cargo new logstats
cd logstats
mkdir -p tests/fixtures

# fixtures e .gitattributes
cp "D:/Repos/rust/projeto-01-logstats/fixtures/"* tests/fixtures/
cp "D:/Repos/rust/projeto-01-logstats/.gitattributes" .
```

O `.gitattributes` **não é opcional**: sem ele o git no Windows normaliza os finais
de linha, o `crlf.log` perde os `\r\n` e o teste I10 vira falso-positivo — passa
sem testar nada.

Confira depois de copiar:

```bash
python -c "print(open('tests/fixtures/crlf.log','rb').read()[:60])"
# tem que aparecer \r\n
```

`expected/` não precisa ser copiado: use `include_str!` ou cole inline no assert.

---

## Fixtures

Todas foram verificadas contra uma implementação de referência da SPEC — os
números abaixo são o que o seu `logstats` tem que produzir.

| Arquivo | Cobre | total / válidas / inválidas / branco | exit | Etapa |
|---|---|---|---|---|
| `sample.log` | caso canônico | 20 / 16 / 3 / 1 | 0 | 1, 2, 3 |
| `empty.log` | 0 bytes | 0 / 0 / 0 / 0 | 0 | 3 |
| `all_invalid.log` | 5 inválidas, motivos variados | 5 / 0 / 5 / 0 | **3** | 3, 4 |
| `crlf.log` | terminadores `\r\n` | 5 / 4 / 0 / 1 | 0 | 3 |
| `no_trailing_newline.log` | última linha sem `\n` | 3 / 3 / 0 / 0 | 0 | 3 |
| `utf8_mixed.log` | acento, emoji e **bytes UTF-8 inválidos** na linha 2 | 4 / 3 / 1 / 0 | 0 | 3 |
| `lo g ção.log` | caminho com espaço e acento | 2 / 2 / 0 / 0 | 0 | 4 |

Motivos esperados, por linha:

| Fixture | stderr |
|---|---|
| `sample.log` | `line 8: too_few_tokens`, `line 11: bad_level`, `line 15: bad_duration` |
| `all_invalid.log` | `line 1: too_few_tokens`, `line 2: bad_timestamp`, `line 3: bad_level`, `line 4: bad_module`, `line 5: bad_duration` |
| `utf8_mixed.log` | `line 2: invalid_utf8` — e as linhas 3 e 4 **continuam sendo processadas** |

## Saídas esperadas

`expected/*.txt` assume que o binário foi invocado com o caminho literal
`tests/fixtures/sample.log` — os campos `file:` (texto) e `"file"` (json) refletem
exatamente a string passada como argumento. Rodando de outro diretório, ajuste
essa linha no assert.

| Arquivo | Comando | Etapa |
|---|---|---|
| `sample.stdout.txt` | `logstats tests/fixtures/sample.log` | 2 |
| `sample.stderr.txt` | idem, canal stderr | 3 |
| `sample.json.txt` | `logstats --format json tests/fixtures/sample.log` | 5 |

Exit code esperado nos três: `0`.

---

## Gates que valem desde a Etapa 1

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
cargo test --doc
```

Rode os quatro no fim de cada etapa, não só no fim do projeto.
