# Etapa 5 — JSON, performance e gates de qualidade

**Estimativa:** 2–3h · **Entregável:** a v1 fechada · **Referência:** SPEC §6.2, §7.3, §9.6, §11

---

## Objetivo

Fechar a v1: formato JSON, o teste de streaming que cobra a restrição de memória,
e os gates de qualidade. É a etapa mais curta em código e a mais chata em detalhe —
JSON à mão não perdoa.

---

## O que você vai encostar

| Tópico | Onde |
|---|---|
| Escaping e serialização manual | `render_json` |
| Determinismo de saída | ordem fixa de chaves |
| Teste de performance e de memória | E14 |
| Ferramental: `llvm-cov`, `clippy`, `fmt` | gates |
| (opcional) property testing | `proptest` |

---

## Escopo de código

Entra `render_json(&Report) -> String` em `render.rs`, e a flag `--format json`
(que a Etapa 4 já aceita e valida) passa a fazer algo.

### Regras do JSON (SPEC §6.2, §7.3)

- **Uma linha**, compacta, sem espaços.
- **Ordem de chaves fixa**, exatamente como a SPEC §7.3. Não é "qualquer JSON
  equivalente" — o teste compara string.
- `null` nas métricas de duração quando `samples == 0`.
- Escapar `"`, `\` e chars de controle (`\u00XX`) no campo `file`.

Isso é o teste do determinismo: se em algum lugar você guardou algo num `HashMap`
e itera na renderização, U54 (mesma saída em 100 execuções) pega.

---

## Testes desta etapa

| Grupo | IDs da SPEC §9 | Tipo |
|---|---|---|
| JSON | **U52, U53, U54** | unitário |
| JSON ponta a ponta | **E02** | e2e |
| Streaming / performance | **E14** | e2e (`#[ignore]`) |
| Doctests verdes | **D04** | gate |
| Qualidade | **Q01–Q06** | gate |
| Propriedade (opcional) | **P01–P04** | proptest |

### E14 — o teste que cobra o streaming

Gera 200k linhas num arquivo temporário, roda o binário, exige < 3s e RSS estável.
Roda com `cargo test -- --ignored` (não entra no `cargo test` normal).

Se a memória crescer com o tamanho do arquivo, você quebrou a §3.3 — é este teste
que pega o `read_to_string` que sobreviveu desde a Etapa 2.

### P01–P04 (opcional, recomendado)

`[dev-dependencies] proptest = "1"` — é a **única** dependência permitida, e só
em dev. P01 (`parse_line` nunca entra em panic para qualquer `String` arbitrária)
é o que dá mais retorno: ele encontra o slicing por índice em UTF-8 que os testes
de exemplo não encontraram.

---

## Gates de qualidade (SPEC §9.6)

| ID | Comando |
|---|---|
| Q01 | `cargo fmt --check` |
| Q02 | `cargo clippy --all-targets -- -D warnings` |
| Q03 | `cargo test` |
| Q04 | `cargo test --doc` |
| Q05 | `cargo llvm-cov` ≥ 85% de linhas em `parser.rs` e `stats.rs` |
| Q06 | `grep -rn "unwrap()\|expect(" src/` retorna 0 ocorrências |

---

## Definition of Done — do projeto inteiro

- [ ] Todos os testes das Etapas 1–5 verdes (U01–U54, I01–I13, D01–D04, E01–E14)
- [ ] Q01–Q06 passando
- [ ] `README.md` do repositório com: exemplo de uso, gramática do log, tabela de exit codes
- [ ] `cargo run -- tests/fixtures/sample.log` reproduz a SPEC §7.2 exatamente
- [ ] `cargo run -- --format json tests/fixtures/sample.log` reproduz a SPEC §7.3 exatamente
- [ ] `Cargo.toml` sem nada em `[dependencies]`
- [ ] Você consegue explicar, sem consultar nada: por que `LogEntry` tem lifetime, e o que aconteceria se você trocasse `&'a str` por `String`

## Para a revisão final, me manda

1. `tree src tests`
2. `Cargo.toml`
3. `src/parser.rs` e `src/stats.rs`
4. Saída de `cargo test`, `cargo test -- --ignored` e `cargo clippy --all-targets -- -D warnings`

Eu vou conferir a §7.2 byte a byte, procurar `clone()` desnecessário, e propor
2–3 casos adversariais que você não escreveu. Depois disso o Projeto 02 é liberado.

---

## Depois da v1 — extensões (SPEC §12)

Não são obrigatórias, mas X1 e X2 valem muito **agora**, com o código fresco: você
troca o que escreveu à mão por crate e sente a diferença.

| # | Extensão | O que ensina |
|---|---|---|
| X1 | Trocar o parser de args manual por `clap` (derive) | macros derive, ergonomia de crates |
| X2 | Trocar `render_json` por `serde` + `serde_json` | traits derive, serialização |
| X3 | `--follow` (tail -f) | `std::thread`, channels, sinais |
| X4 | Paralelizar com `rayon` e medir o ganho | `Send`/`Sync`, paralelismo de dados |
| X5 | `cargo-fuzz` no parser | fuzzing |

X3 e X4 são candidatos a virar o Projeto 02.
