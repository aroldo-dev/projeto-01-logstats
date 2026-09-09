# Etapa 2 — Agregação e relatório em texto

**Estimativa:** 3–4h · **Entregável:** o relatório canônico da SPEC §7.2 · **Referência:** SPEC §6.1, §7.2

---

## Objetivo

`logstats <ARQUIVO>` agora imprime o relatório de verdade — contagens por nível,
ranking de módulos por erro, estatísticas de duração — **byte a byte igual** à
SPEC §7.2. O andaime de eco da Etapa 1 morre aqui.

Ainda sem flags (defaults fixos: `--top 5`, formato texto) e sem os warnings no
stderr. Exit code sempre 0.

---

## O que você vai encostar

| Tópico | Onde |
|---|---|
| Coleções: `Vec`, `HashMap`, `BTreeMap` | contagem por módulo e por nível |
| Iteradores: `filter`, `fold`, `collect`, `sort_by` | agregação e ranking |
| Genéricos leves: `impl BufRead` | `analyze(r: impl BufRead)` de verdade |
| `Ord` / desempate customizado | ranking `(contagem desc, nome asc)` |
| Aritmética de ponto flutuante e arredondamento | `mean` |
| Formatação com `{:width$}` / `ljust`/`rjust` na mão | alinhamento do relatório |

---

## Escopo de código

Entram dois módulos:

```
src/
├── stats.rs     # Report, analyze(impl BufRead) -> Report
└── render.rs    # render_text(&Report) -> String
```

`main.rs` vira: abre arquivo → `analyze` → `render_text` → `print!`.

**Restrição que vale a partir daqui (SPEC §3.3):** nada de `read_to_string`. A
memória tem que ser O(módulos + amostras de duração), não O(tamanho do arquivo).
Agregue **dentro** do loop de linhas. O teste E14 (Etapa 5) é quem cobra isso,
mas se você fizer errado agora vai ter que reescrever depois.

---

## As três armadilhas desta etapa

1. **`HashMap` tem ordem de iteração não determinística.** Se você renderizar
   iterando o HashMap direto, o relatório sai numa ordem diferente a cada
   execução e U54/E01 ficam intermitentes — o pior tipo de teste quebrado.
   Ordene antes de renderizar, ou use `BTreeMap` onde a ordem importa.
2. **Percentil é *nearest-rank*, não interpolação.** `i = clamp(ceil(p/100 × n), 1, n)`
   sobre o vetor ordenado, **1-indexado**. Se você usar a fórmula do NumPy, U31 quebra.
3. **`{:.1}` em Rust arredonda half-to-even em alguns casos de borda**, e a SPEC
   pede half-away-from-zero. U33 (`[1,1,2]` → `1.3`) é exatamente esse caso.

Alinhamento (SPEC §6.1): `"  " + nome.ljust(W) + "  " + valor.rjust(V)`, com `W`
= maior nome **daquele bloco** e `V` = maior valor **daquele bloco**. Blocos
separados por **uma** linha em branco. Sempre termina com `\n`.

`by_level` imprime sempre os 5 níveis na ordem TRACE→ERROR, mesmo zerados.
Ranking vazio → `  (none)`. Zero amostras → `  (no samples)`.

---

## Testes desta etapa

| Grupo | IDs da SPEC §9 | Tipo | Onde |
|---|---|---|---|
| Agregação | **U24–U34** | unitário | `src/stats.rs` |
| Renderização texto | **U49–U51** | unitário | `src/render.rs` |
| API pública | **I01, I12, I13** | integração | `tests/lib_api.rs` (novo) |
| Doctest de `analyze` | **D03** | doctest | `///` em `analyze` |
| Caso canônico ponta a ponta | **E01** (só stdout e exit code) | e2e | `tests/cli.rs` (novo) |

Notas:

- `tests/lib_api.rs` usa `Cursor<&str>` como `impl BufRead` — **não toca em disco.**
  É aqui que o `impl BufRead` paga o investimento.
- **E01 nesta etapa cobra só stdout e exit 0.** A parte de stderr (`warning:
  line 8: ...`) entra na Etapa 3; deixe o assert de stderr comentado com
  `// TODO etapa 3` ou parta o teste em dois.
- E2E roda o binário de verdade: `Command::new(env!("CARGO_BIN_EXE_logstats"))`.

**Conferência aritmética** (SPEC §7.2): durações ordenadas
`[3, 7, 12, 15, 25, 38, 42, 87, 310, 640, 980, 1503]`, soma 3662, n=12 →
mean 305.1666… → `305.2`; p50=38, p90=980, p95=1503, max=1503.

---

## Definition of Done

- [ ] `cargo run -- tests/fixtures/sample.log` reproduz a SPEC §7.2 **byte a byte** (confira com `diff`, não com o olho)
- [ ] U24–U34, U49–U51, I01, I12, I13 verdes
- [ ] D03 escrito; `cargo test --doc` verde
- [ ] E01 verde para stdout + exit code
- [ ] Nenhum `read_to_string` no código
- [ ] `cargo fmt --check` e `cargo clippy --all-targets -- -D warnings` limpos
- [ ] Rodar `cargo test` 10 vezes seguidas dá o mesmo resultado (pega o problema do HashMap)

## Para a revisão, me manda

1. `src/stats.rs` e `src/render.rs`
2. `diff <(cargo run -q -- tests/fixtures/sample.log) expected/sample.stdout.txt` — tem que sair vazio
3. Saída de `cargo test`
