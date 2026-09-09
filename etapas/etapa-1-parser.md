# Etapa 1 — Parser de linha + eco

**Estimativa:** 3–4h · **Entregável:** binário que roda · **Referência:** SPEC §4, §7.1

---

## Objetivo

`logstats <ARQUIVO>` lê o arquivo linha a linha e imprime, para cada linha, se ela
é válida, inválida (com o motivo) ou em branco. **Sem agregação, sem relatório,
sem flags.** O ponto desta etapa é o parser e o desenho dos tipos.

Saída esperada rodando contra `sample.log` (formato livre — só esta etapa usa,
é um andaime, some na Etapa 2):

```
1 OK    INFO  auth (3 campos)
2 OK    ERROR db (3 campos)
...
8 INVALID too_few_tokens
...
11 INVALID bad_level
...
15 INVALID bad_duration
16 BLANK
...
```

Exit code sempre 0 nesta etapa.

---

## O que você vai encostar

| Tópico | Onde |
|---|---|
| Cargo, crate lib + bin | `src/lib.rs` + `src/main.rs` |
| Structs e enums | `LogEntry`, `Level`, `ParseError` |
| Pattern matching | `match` no parser |
| `Option` / `Result` / `?` | fluxo do parser |
| Traits: `FromStr`, `Display`, `Ord`, `PartialOrd` | `Level` |
| `String` vs `&str`, slices | você **vai** errar aqui pelo menos uma vez |
| Lifetimes | `LogEntry<'a>` emprestando da linha |
| Iteradores e closures | `split_whitespace`, `map`, `collect` |
| `impl Trait` em argumento | `fn analyze(r: impl BufRead)` (só o esqueleto) |
| Streaming com `BufRead::lines()` | leitura sem carregar o arquivo todo |

**Fora desta etapa:** filtros, estatísticas, formatação de saída, parser de
argumentos, exit codes 1/2/3, JSON.

---

## Escopo de código

```
logstats/
├── Cargo.toml          # sem [dependencies] — isso vale para o projeto inteiro
├── src/
│   ├── main.rs         # abre o arquivo do argv[1], chama a lib, imprime
│   ├── lib.rs          # #![forbid(unsafe_code)] + re-exports
│   ├── entry.rs        # Level, LogEntry<'a>
│   └── parser.rs       # parse_line<'a>(&'a str) -> Result<LogEntry<'a>, ParseError>
└── tests/
    └── fixtures/
        └── sample.log
```

`error.rs` ainda não — `ParseError` pode morar em `parser.rs` e ser promovido na
Etapa 3, quando aparecer o erro de I/O.

### Assinatura-alvo

```rust
pub fn parse_line<'a>(line: &'a str) -> Result<LogEntry<'a>, ParseError>;
```

Se você não consegue fazer isso compilar sem `String` nem `clone()`, **pare e
resolva isso antes de seguir** — é o ponto da etapa. A dica está na SPEC §10.8.

---

## Regras que valem aqui

Da SPEC §4.2: **R1** (linha em branco), **R2** (inválida não interrompe o
processamento), **R3** (chave repetida, último vence), **R4** (`duration_ms` tem
que ser `u64`), **R5** (timestamp validado semanticamente), **R10** (UTF-8
multibyte nos valores).

R7 (CRLF), R8 (UTF-8 inválido) e R9 (sem `\n` final) ficam para a Etapa 3 — mas
se o seu parser já remover um `\r` final, ótimo, U18 cobre isso aqui.

---

## Testes desta etapa

Todos unitários (`#[cfg(test)] mod tests` dentro de cada módulo) + 2 doctests.

| Grupo | IDs da SPEC §9 | Onde |
|---|---|---|
| Parser | **U01–U20** | `src/parser.rs` |
| Level / LogEntry | **U21–U23** | `src/entry.rs` |
| Doctests | **D01, D02** | `///` em `parse_line` e `Level` |

Regras que valem para todos:

- Use tabelas (`for (input, expected) in [...]`) em vez de um `#[test]` por caso.
- **Asserte sobre a variante do enum de erro, nunca sobre a string da mensagem.**
  Se você escrever `assert_eq!(e.to_string(), "bad timestamp")`, o teste quebra
  quando você melhorar a mensagem — e isso não é regressão.
- `unwrap()` em teste é permitido. Em `src/`, não.

---

## Definition of Done

- [ ] `cargo run -- tests/fixtures/sample.log` classifica as 20 linhas: 16 válidas, 3 inválidas (8 `too_few_tokens`, 11 `bad_level`, 15 `bad_duration`), 1 em branco
- [ ] U01–U23 verdes
- [ ] D01 e D02 escritos; `cargo test --doc` verde
- [ ] `cargo fmt --check` limpo
- [ ] `cargo clippy --all-targets -- -D warnings` limpo
- [ ] `Cargo.toml` sem `[dependencies]`
- [ ] Zero `unwrap()` / `expect()` / `panic!` em `src/`
- [ ] `#![forbid(unsafe_code)]` no topo de `lib.rs`

## Para a revisão, me manda

1. `src/parser.rs` e `src/entry.rs` inteiros
2. Saída de `cargo test` e `cargo clippy --all-targets -- -D warnings`
3. Resposta em uma frase: **por que `LogEntry` tem lifetime, e o que quebraria se você trocasse `&'a str` por `String`?**
