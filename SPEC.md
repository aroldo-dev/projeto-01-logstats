# Projeto 01 — `logstats`

**Nível:** Básico (primeiro projeto)
**Estimativa:** 13–18h, divididas em 5 etapas (ver §0)
**Papel:** você é o dev. Eu sou produto + QA. A spec abaixo é contrato — a suíte de testes valida ela, não a sua implementação.

---

## 0. Como este documento é usado

Este arquivo é o **contrato final** do projeto: o que o `logstats` tem que fazer
quando estiver pronto. Ele não é a ordem de implementação.

A execução é por etapas, em `etapas/`. Cada etapa recorta um pedaço desta spec,
entrega um binário que roda de ponta a ponta, e tem o seu próprio subconjunto do
plano de testes da §9 e a sua própria *Definition of Done*:

| # | Etapa | Recorta desta spec | Testes |
|---|---|---|---|
| 1 | [Parser de linha + eco](etapas/etapa-1-parser.md) | §4, §7.1 | U01–U23, D01–D02 |
| 2 | [Agregação e relatório texto](etapas/etapa-2-agregacao.md) | §6.1, §7.2 | U24–U34, U49–U51, I01, I12–I13, D03, E01 |
| 3 | [Robustez, stderr e exit codes](etapas/etapa-3-robustez.md) | §4.2 (R7–R9), §6.3, §6.4 | U17–U19, I07–I11, E01, E03–E04, E08–E10 |
| 4 | [CLI completa e filtros](etapas/etapa-4-cli.md) | §5 | U35–U48, I02–I06, E05–E07, E11–E13 |
| 5 | [JSON, performance e gates](etapas/etapa-5-json-e-gates.md) | §6.2, §7.3, §9.6 | U52–U54, E02, E14, D04, Q01–Q06, P01–P04 |

**Leia esta spec inteira uma vez antes de começar** — você precisa saber para onde
está indo para não desenhar tipos que travam a Etapa 3. Depois volte e trabalhe
etapa por etapa: implemente, feche a DoD, mande para revisão, só então avance.

A §11 (Definition of Done) e a §13 (como vamos revisar) valem para o projeto
inteiro, ao fim da Etapa 5.

---

## 1. Contexto de produto

Times de plataforma precisam responder rápido: "o que quebrou nas últimas horas e onde está lento?". Ferramentas completas (Datadog, Loki) são caras e exigem ingestão. Queremos um binário único, sem dependências, que leia um arquivo de log estruturado e cuspa um relatório determinístico — usável em terminal e em pipeline de CI.

**Não-objetivo:** parsear formatos arbitrários, indexar, servir HTTP, tail em tempo real.

---

## 2. Por que este projeto para começar

Ele força quase todo o bloco "básico" do roadmap de Rust sem cair em exercício de brinquedo:

| Tópico do roadmap | Onde aparece aqui |
|---|---|
| Cargo, crates, módulos | crate lib + bin, 7 módulos |
| Ownership / borrowing | parser devolve `&str` emprestado da linha |
| Lifetimes | assinatura do parser (`'a`) |
| `String` vs `&str`, slices | você vai errar aqui pelo menos uma vez |
| Structs e enums | `LogEntry`, `Level`, `ParseError` |
| Pattern matching | `match` no parser e no render |
| `Option` / `Result` / `?` | fluxo inteiro |
| Error handling | enum de erro + `Display` + `std::error::Error` + `From` |
| Traits | `FromStr`, `Display`, `Ord` |
| Coleções | `Vec`, `HashMap`, `BTreeMap` |
| Iteradores e closures | agregação com `filter`/`fold`/`collect` |
| Genéricos (leve) | `fn analyze(r: impl BufRead)` |
| Testes | unit, integração, doc, e2e |

**Fora de escopo nesta v1:** async, threads, `unsafe`, macros declarativas, smart pointers (`Rc`/`RefCell`), trait objects. Vem no Projeto 02+.

---

## 3. Restrições técnicas (obrigatórias)

1. **Zero dependências em runtime.** `Cargo.toml` sem seção `[dependencies]`. Nada de `clap`, `serde`, `chrono`, `regex`, `anyhow`. `[dev-dependencies]` só nas extensões opcionais.
2. **Crate duplo:** `src/lib.rs` (toda a lógica, pública e testável) + `src/main.rs` (args, I/O, exit codes). O binário é uma casca fina.
3. **Streaming obrigatório.** Nada de `read_to_string`. Consumo de memória tem que ser O(módulos + amostras de duração), não O(tamanho do arquivo).
4. `#![forbid(unsafe_code)]` no topo de `lib.rs`.
5. `cargo clippy --all-targets -- -D warnings` limpo. `cargo fmt --check` limpo.
6. **Nenhum `panic!`/`unwrap()`/`expect()` em caminho alcançável por input do usuário.** `unwrap()` em testes é permitido.
7. Edition 2021 ou 2024, Rust estável.

---

## 4. Formato de entrada

### 4.1 Gramática

```
linha      := timestamp SP+ level SP+ module ( SP+ campo )*
timestamp  := YYYY-MM-DDTHH:MM:SSZ        (exatamente 20 chars, UTC)
level      := TRACE | DEBUG | INFO | WARN | ERROR    (case-sensitive)
module     := [a-z0-9_]{1,32}
campo      := chave "=" valor
chave      := [a-z0-9_]+
valor      := token_sem_espaco | '"' ( qualquer char exceto '"' )* '"'
```

Exemplo:

```
2026-03-14T10:22:33Z ERROR db    query=select_users duration_ms=1503 error="connection timeout"
```

### 4.2 Regras semânticas

| # | Regra |
|---|---|
| R1 | Linha vazia ou só whitespace → **em branco**: não conta como válida nem inválida, tem contador próprio. |
| R2 | Qualquer linha que não case com a gramática → **inválida**: incrementa contador, registra `(número_da_linha, motivo)`, e o processamento **continua**. |
| R3 | Chave repetida na mesma linha → o **último** valor vence. |
| R4 | O campo `duration_ms`, se presente, tem que parsear como `u64`. Se não parsear (vazio, `abc`, negativo, overflow) → linha inválida. |
| R5 | Timestamp é validado **sintaticamente e semanticamente**: mês 1–12, dia 1–31 (não precisa validar mês×dia nem ano bissexto), hora 0–23, min/seg 0–59. |
| R6 | Comparação temporal é **lexicográfica** sobre a string do timestamp (o formato permite isso — aproveite, não implemente calendário). |
| R7 | Terminadores `\n` e `\r\n` ambos suportados. Windows é ambiente de primeira classe aqui. |
| R8 | Bytes que não formam UTF-8 válido → a linha conta como inválida (motivo `InvalidUtf8`) e o processamento continua. |
| R9 | Última linha sem `\n` final é processada normalmente. |
| R10 | Valores podem conter UTF-8 multibyte (acentos, emoji) sem quebrar nada. |

---

## 5. Interface de linha de comando

```
logstats [OPÇÕES] <ARQUIVO>
logstats [OPÇÕES] -            # lê de stdin
```

| Opção | Default | Comportamento |
|---|---|---|
| `--level <LEVEL>` | `TRACE` | Considera só linhas com nível **>=** LEVEL. Ordem: TRACE < DEBUG < INFO < WARN < ERROR. |
| `--module <NAME>` | (todos) | Repetível. Se informado ≥1 vez, só esses módulos entram. |
| `--since <TS>` | (sem limite) | Timestamp RFC3339. **Inclusivo.** |
| `--until <TS>` | (sem limite) | Timestamp RFC3339. **Exclusivo.** |
| `--top <N>` | `5` | Quantos módulos listar no ranking de erros. `0` é válido → lista vazia. |
| `--format <text\|json>` | `text` | Formato de saída. |
| `--max-invalid-report <N>` | `5` | Quantas linhas inválidas detalhar no stderr. |
| `-h`, `--help` | — | Uso no stdout, exit 0. |
| `-V`, `--version` | — | `logstats <versão do Cargo.toml>`, exit 0. |

**Regras do parser de argumentos (feito à mão, sem clap):**

- `--opt valor` e `--opt=valor` são equivalentes.
- `--` encerra as opções; tudo depois é posicional.
- Opção desconhecida → erro de uso.
- Opção que exige valor sem valor → erro de uso.
- Zero ou ≥2 posicionais → erro de uso.
- `--since` > `--until` → erro de uso.
- Filtros são aplicados **antes** da agregação. Linhas filtradas não entram em `lines_valid`.

---

## 6. Saída

### 6.1 Formato `text` (exato — os testes comparam byte a byte)

Blocos separados por **uma** linha em branco. Sempre termina com `\n`.

Regras de alinhamento em cada bloco: `"  " + nome.ljust(W) + "  " + valor.rjust(V)`, onde `W` = maior nome **daquele bloco** e `V` = maior valor **daquele bloco**.

`by_level` sempre imprime os 5 níveis, na ordem TRACE→ERROR, mesmo zerados.

`top_modules_by_error` lista só módulos com ≥1 ERROR, ordenados por `(contagem desc, nome asc)`. Se vazio: `  (none)`.

`mean` = 1 casa decimal, arredondamento half-away-from-zero.
Percentis = **nearest-rank**: `i = clamp(ceil(p/100 × n), 1, n)` sobre o vetor ordenado, 1-indexado. Nada de interpolação.
Se `samples == 0`: o bloco vira só `  (no samples)`.

### 6.2 Formato `json`

Uma linha, compacta (sem espaços), **ordem de chaves fixa** conforme o exemplo em §7.3. `null` para as métricas quando `samples == 0`. Escapar `"`, `\` e chars de controle (`\u00XX`) no campo `file`.

### 6.3 stderr

Uma linha por linha inválida, até `--max-invalid-report`, no formato:

```
warning: line 8: bad_timestamp
```

Motivos válidos (use exatamente estes identificadores): `too_few_tokens`, `bad_timestamp`, `bad_level`, `bad_module`, `field_without_eq`, `bad_key`, `unterminated_quote`, `bad_duration`, `invalid_utf8`.

Se houve mais inválidas do que o limite, uma linha final:

```
warning: 12 more invalid lines suppressed
```

### 6.4 Exit codes

| Código | Quando |
|---|---|
| 0 | Sucesso (inclui arquivo vazio, e inclui `--help`/`--version`) |
| 1 | Erro de uso (argumentos) |
| 2 | Erro de I/O (arquivo inexistente, sem permissão, é um diretório) |
| 3 | `lines_valid == 0` **e** `lines_total > 0`. O relatório ainda é impresso no stdout. |

Erros de uso e de I/O vão para **stderr**; stdout fica vazio.

---

## 7. Caso de aceitação canônico

### 7.1 `tests/fixtures/sample.log`

```
2026-03-14T10:22:31Z INFO  auth  user=alice action=login duration_ms=42
2026-03-14T10:22:33Z ERROR db    query=select_users duration_ms=1503 error="connection timeout"
2026-03-14T10:22:35Z WARN  http  path=/api/v1/pets status=429 duration_ms=87
2026-03-14T10:22:36Z INFO  http  path=/api/v1/pets status=200 duration_ms=12
2026-03-14T10:22:38Z DEBUG cache key=pets:1 hit=true
2026-03-14T10:22:39Z ERROR db    query=insert_pet duration_ms=980 error="deadlock detected"
2026-03-14T10:22:40Z INFO  auth  user=bob action=logout
LINHA QUEBRADA
2026-03-14T10:22:42Z ERROR http  path=/api/v1/pets status=500 duration_ms=310
2026-03-14T10:22:43Z INFO  db    query=select_pets duration_ms=25
2026-03-14T10:22:44Z FATAL db    query=drop_all
2026-03-14T10:22:45Z TRACE cache key=pets:2 hit=false
2026-03-14T10:22:46Z WARN  auth  user=carol reason="weak password"
2026-03-14T10:22:47Z INFO  http  path=/health status=200 duration_ms=3
2026-03-14T10:22:48Z ERROR db    query=select_users duration_ms=abc

2026-03-14T10:22:50Z ERROR auth  user=dave reason="brute force" duration_ms=7
2026-03-14T10:22:51Z INFO  http  path=/api/v1/pets status=200 duration_ms=15
2026-03-14T10:22:52Z WARN  db    query=select_pets duration_ms=640
2026-03-14T10:22:53Z INFO  auth  user=erin action=login duration_ms=38
```

(3 inválidas: linha 8 `too_few_tokens`, linha 11 `bad_level`, linha 15 `bad_duration`. Linha 16 em branco.)

### 7.2 `logstats tests/fixtures/sample.log` → stdout

```
file: tests/fixtures/sample.log
lines_total: 20
lines_valid: 16
lines_invalid: 3
lines_blank: 1

by_level:
  TRACE  1
  DEBUG  1
  INFO   7
  WARN   3
  ERROR  4

top_modules_by_error (3):
  db    2
  auth  1
  http  1

duration_ms (samples: 12):
  mean  305.2
  p50      38
  p90     980
  p95    1503
  max    1503
```

Durações ordenadas para conferência: `[3, 7, 12, 15, 25, 38, 42, 87, 310, 640, 980, 1503]` — soma 3662, média 305.1666… → `305.2`.

stderr:

```
warning: line 8: too_few_tokens
warning: line 11: bad_level
warning: line 15: bad_duration
```

Exit 0.

### 7.3 `--format json` → stdout

```json
{"file":"tests/fixtures/sample.log","lines_total":20,"lines_valid":16,"lines_invalid":3,"lines_blank":1,"by_level":{"TRACE":1,"DEBUG":1,"INFO":7,"WARN":3,"ERROR":4},"top_modules_by_error":[{"module":"db","errors":2},{"module":"auth","errors":1},{"module":"http","errors":1}],"duration_ms":{"samples":12,"mean":305.2,"p50":38,"p90":980,"p95":1503,"max":1503}}
```

---

## 8. Estrutura sugerida

```
logstats/
├── Cargo.toml
├── src/
│   ├── main.rs      # args → run() → exit code. Sem lógica de domínio.
│   ├── lib.rs       # re-exports + #![forbid(unsafe_code)]
│   ├── cli.rs       # Config, parse_args(), help/version
│   ├── error.rs     # LogStatsError + From<io::Error>
│   ├── entry.rs     # Level (Ord, FromStr, Display), LogEntry<'a>
│   ├── parser.rs    # parse_line<'a>(&'a str) -> Result<Parsed<'a>, ParseError>
│   ├── stats.rs     # Report, analyze(impl BufRead, &Config)
│   └── render.rs    # render_text(&Report), render_json(&Report)
└── tests/
    ├── fixtures/
    │   ├── sample.log
    │   ├── empty.log
    │   ├── all_invalid.log
    │   ├── crlf.log
    │   ├── no_trailing_newline.log
    │   ├── utf8_mixed.log
    │   └── lo g ção.log
    ├── lib_api.rs
    └── cli.rs
```

---

## 9. Plano de testes

Notação: **U** = unitário, **I** = integração de biblioteca, **D** = doctest, **E** = end-to-end, **P** = propriedade, **Q** = qualidade.

### 9.1 Unitários — `#[cfg(test)] mod tests` dentro de cada módulo

Use tabelas (`for (input, expected) in [...]`) em vez de um `#[test]` por caso quando fizer sentido. **Asserte sobre a variante do enum de erro, nunca sobre a string da mensagem.**

**`parser.rs`**

| ID | Caso | Esperado |
|---|---|---|
| U01 | Linha mínima válida, sem campos | Ok, 0 campos |
| U02 | Linha com 3 campos | Ok, 3 campos |
| U03 | Valor entre aspas com espaços | Ok, valor sem as aspas |
| U04 | Valor entre aspas contendo `=` | Ok |
| U05 | Chave duplicada | Ok, último vence |
| U06 | Múltiplos espaços entre tokens | Ok |
| U07 | Timestamp inválido (sem `Z`, sem `T`, mês 13, hora 25, 19 chars, 21 chars) | `BadTimestamp` (6 casos) |
| U08 | Level `FATAL` | `BadLevel` |
| U09 | Level `info` (minúsculo) | `BadLevel` |
| U10 | Module `Db`, `db-1`, `` (vazio), 33 chars | `BadModule` |
| U11 | Campo sem `=` | `FieldWithoutEq` |
| U12 | `duration_ms=abc` | `BadDuration` |
| U13 | `duration_ms=` | `BadDuration` |
| U14 | `duration_ms=-1` | `BadDuration` |
| U15 | `duration_ms=99999999999999999999999` (overflow u64) | `BadDuration` |
| U16 | Aspas não fechadas | `UnterminatedQuote` |
| U17 | Só espaços / string vazia | `Blank` |
| U18 | Linha terminando em `\r` | Ok (o `\r` é removido) |
| U19 | Valores com acento e emoji | Ok, sem panic |
| U20 | Menos de 3 tokens | `TooFewTokens` |

**`entry.rs`**

| ID | Caso |
|---|---|
| U21 | `Level::from_str` para os 5 níveis + 1 inválido |
| U22 | Round-trip `Level::from_str(&level.to_string()) == Ok(level)` para todos |
| U23 | Ordenação: `TRACE < DEBUG < INFO < WARN < ERROR` |

**`stats.rs`**

| ID | Caso | Esperado |
|---|---|---|
| U24 | Entrada vazia | tudo 0, `samples == 0` |
| U25 | Contagem por nível | bate |
| U26 | `top_modules` com empate | desempate alfabético |
| U27 | `top_modules` com `N` > nº de módulos | retorna todos |
| U28 | `top_modules` com `N = 0` | vazio |
| U29 | Percentil, `n=1` | p50 = p90 = p95 = único valor |
| U30 | Percentil, `n=2`, `[10,20]` | p50=10, p90=20, p95=20 |
| U31 | Percentil, valores `1..=100` | p50=50, p90=90, p95=95 |
| U32 | Percentil com valores repetidos | correto |
| U33 | `mean` de `[1,2]` → `1.5`; `[1,1,2]` → `1.3` | arredondamento |
| U34 | `max` | correto |
| U35 | Filtro `--level WARN` | descarta TRACE/DEBUG/INFO |
| U36 | Filtro de módulo com 2 módulos | union, não interseção |

**`cli.rs`**

| ID | Caso | Esperado |
|---|---|---|
| U37 | `--top=3` e `--top 3` | mesmo Config |
| U38 | Nenhuma flag | defaults de §5 |
| U39 | `--top abc` | erro de uso |
| U40 | `--frobnicate` | erro de uso |
| U41 | `--top` no fim, sem valor | erro de uso |
| U42 | Dois posicionais | erro de uso |
| U43 | Zero posicionais | erro de uso |
| U44 | `-- --weird.log` | arquivo chamado `--weird.log` |
| U45 | `--module a --module b` | acumula |
| U46 | `--since abc` | erro de uso |
| U47 | `--since` > `--until` | erro de uso |
| U48 | `--format xml` | erro de uso |

**`render.rs`**

| ID | Caso |
|---|---|
| U49 | Alinhamento com nome de módulo longo vs curto |
| U50 | Bloco de duração com 0 amostras → `  (no samples)` |
| U51 | Ranking vazio → `  (none)` |
| U52 | JSON: escaping de `"` e `\` no `file` |
| U53 | JSON: `null` quando `samples == 0` |
| U54 | JSON: ordem das chaves estável em 100 execuções |

### 9.2 Integração de biblioteca — `tests/lib_api.rs`

Só a API pública, com `Cursor<&str>` / `&[u8]` como `impl BufRead`. Sem tocar em disco.

| ID | Caso |
|---|---|
| I01 | `sample.log` inline → `Report` idêntico ao de §7.2 |
| I02 | `--level WARN` → só WARN e ERROR |
| I03 | Filtro por módulo `db` |
| I04 | `--since` na borda exata (timestamp igual → **incluído**) |
| I05 | `--until` na borda exata (timestamp igual → **excluído**) |
| I06 | `--since` + `--until` + `--level` combinados |
| I07 | Entrada só com linhas inválidas → `lines_valid == 0`, `lines_invalid == n` |
| I08 | Entrada vazia (0 bytes) |
| I09 | Sem `\n` na última linha |
| I10 | CRLF em todas as linhas |
| I11 | Bytes UTF-8 inválidos no meio → linha contada como inválida **e as linhas seguintes continuam sendo processadas** |
| I12 | Timestamps fora de ordem cronológica → não afeta o resultado |
| I13 | Módulo aparece em 2 níveis diferentes → contagens independentes |

### 9.3 Doctests

| ID | Item |
|---|---|
| D01 | `parse_line` com exemplo executável |
| D02 | `Level` com exemplo de `from_str` |
| D03 | `analyze` com exemplo usando `Cursor` |
| D04 | `cargo test --doc` verde |

### 9.4 E2E — `tests/cli.rs`

Rodar o binário de verdade: `Command::new(env!("CARGO_BIN_EXE_logstats"))`. Assertar **stdout, stderr e exit code**, sempre os três.

| ID | Caso | Esperado |
|---|---|---|
| E01 | `sample.log` | stdout byte-a-byte igual a §7.2, stderr igual a §7.3, exit 0 |
| E02 | `--format json` | stdout igual a §7.3, exit 0 |
| E03 | Arquivo inexistente | exit 2, stderr não vazio, **stdout vazio** |
| E04 | Caminho é um diretório | exit 2 |
| E05 | `--frobnicate` | exit 1, stdout vazio |
| E06 | `--help` | exit 0, stdout contém `Usage` |
| E07 | `-V` | exit 0, stdout contém `env!("CARGO_PKG_VERSION")` |
| E08 | `-` com o conteúdo do sample no stdin | mesmo relatório do E01, exceto `file: -` |
| E09 | `all_invalid.log` | exit **3**, relatório ainda impresso no stdout |
| E10 | `empty.log` | exit **0**, tudo zerado |
| E11 | `--max-invalid-report 2` num arquivo com 5 inválidas | exatamente 2 `warning: line ...` + 1 linha de resumo |
| E12 | `--max-invalid-report 0` | nenhum detalhe, só o resumo |
| E13 | Caminho com espaço e acento (`tests/fixtures/lo g ção.log`) | funciona |
| E14 | `#[ignore]` — 200k linhas geradas em `tmp` | completa em < 3s, RSS estável |

> Sobre o E14: rode com `cargo test -- --ignored`. Se a memória crescer com o tamanho do arquivo, você quebrou a restrição de streaming (§3.3) — é o teste que pega o `read_to_string`.

### 9.5 Propriedade (opcional, mas recomendado)

`[dev-dependencies] proptest = "1"`.

| ID | Propriedade |
|---|---|
| P01 | `parse_line` nunca entra em panic, para **qualquer** `String` arbitrária |
| P02 | Round-trip: gerar `LogEntry` → serializar → `parse_line` → igual ao original |
| P03 | `percentile(100, xs) == max(xs)` para qualquer `xs` não vazio |
| P04 | `min(xs) <= mean(xs) <= max(xs)` |

### 9.6 Qualidade

| ID | Gate |
|---|---|
| Q01 | `cargo fmt --check` |
| Q02 | `cargo clippy --all-targets -- -D warnings` |
| Q03 | `cargo test` (unit + integração + e2e) |
| Q04 | `cargo test --doc` |
| Q05 | `cargo llvm-cov` ≥ 85% de linhas em `parser.rs` e `stats.rs` |
| Q06 | `grep -rn "unwrap()\|expect(" src/` retorna 0 ocorrências |

---

## 10. Armadilhas que o QA vai testar de propósito

Não são pegadinhas gratuitas — são exatamente os erros que todo dev experiente comete na primeira semana de Rust:

1. **Slicing por índice em string UTF-8** (`&s[0..20]`) entra em panic no meio de um char multibyte. U19 pega isso. Use `char_indices`, `split_whitespace`, `strip_prefix`, ou valide `is_char_boundary`.
2. **`lines()` remove `\n` mas não `\r`.** U18 e I10 pegam. Você está no Windows — isso vai te morder.
3. **`split(' ')` vs `split_whitespace()`** com espaços múltiplos. U06 pega.
4. **`unwrap()` no parse de número.** U15 pega (overflow).
5. **`read_to_string`.** E14 pega.
6. **Ordem de iteração de `HashMap` é não determinística.** Se você renderizar direto do HashMap, U54/E01 vão ficar intermitentes. Ordene antes de renderizar (ou use `BTreeMap` onde a ordem importa).
7. **Formatação de float.** `{:.1}` em Rust usa round-half-to-even em alguns casos de borda; confira U33.
8. **Lifetimes no parser.** `LogEntry<'a>` emprestando da linha é o desenho certo, mas obriga a agregar dentro do loop. Se você começar a clonar `String` em tudo para fugir do borrow checker, o código compila — e você não aprendeu nada. Quando eu revisar, vou perguntar onde você clonou e por quê.

---

## 11. Definition of Done

- [ ] Todos os testes de §9.1–§9.4 escritos e verdes
- [ ] Q01–Q06 passando
- [ ] `README.md` com: exemplo de uso, gramática do log, tabela de exit codes
- [ ] `cargo run -- tests/fixtures/sample.log` reproduz §7.2 exatamente
- [ ] Nenhuma dependência em `[dependencies]`
- [ ] Você consegue explicar, sem consultar nada: por que `LogEntry` tem lifetime, e o que aconteceria se você trocasse `&'a str` por `String`

---

## 12. Extensões (só depois da v1 entregue)

| # | Extensão | O que ensina |
|---|---|---|
| X1 | Trocar o parser de args manual por `clap` (derive) | macros derive, ergonomia de crates |
| X2 | Trocar `render_json` por `serde` + `serde_json` | traits derive, serialização |
| X3 | Adicionar `--follow` (tail -f) | `std::thread`, channels, sinais |
| X4 | Paralelizar com `rayon` e medir o ganho | paralelismo de dados, `Send`/`Sync` |
| X5 | `cargo-fuzz` no parser | fuzzing |

X3 e X4 podem virar o Projeto 02 se você quiser continuidade em vez de projeto novo.

---

## 13. Como vamos revisar

Quando terminar, me manda:

1. A árvore de arquivos (`tree src tests`)
2. O `Cargo.toml`
3. Conteúdo de `parser.rs` e `stats.rs`
4. Saída de `cargo test` e `cargo clippy --all-targets -- -D warnings`

Eu vou: conferir §7.2 byte a byte, procurar `unwrap`/`clone` desnecessários, e propor 2–3 casos de teste adversariais que você não escreveu. Aí seguimos para o Projeto 02.
