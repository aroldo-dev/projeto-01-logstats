# Etapa 4 — CLI completa e filtros

**Estimativa:** 3–4h · **Entregável:** todas as flags da SPEC §5 · **Referência:** SPEC §5

---

## Objetivo

Trocar o `argv[1]` cru por um parser de argumentos de verdade — **escrito à mão,
sem `clap`** — e ligar os filtros (`--level`, `--module`, `--since`, `--until`)
na agregação.

A restrição de zero dependências é deliberada: escrever esse parser é o que faz
você entender o que o `clap` resolve, e é onde `Option`, `Result`, `match` e
iteradores param de ser exercício e viram ferramenta. Na Etapa 5 (extensão X1)
você troca por `clap` e compara.

---

## O que você vai encostar

| Tópico | Onde |
|---|---|
| `std::env::args` e consumo de iterador com estado | `parse_args` |
| `Option`/`Result` combinados em profundidade | validação de flags |
| Comparação lexicográfica de strings como ordem temporal | `--since`/`--until` |
| `Ord` derivado em enum como filtro de nível | `--level` |
| Separar *parsing* de *validação* | `Config` só existe se for válido |

---

## Escopo de código

Entra `src/cli.rs` com `Config`, `parse_args()`, `help()`, `version()`.
`main.rs` vira uma casca fina: `parse_args` → `analyze` → `render` → exit code.

### Regras do parser (SPEC §5)

- `--opt valor` e `--opt=valor` são equivalentes.
- `--` encerra as opções; tudo depois é posicional. (`-- --weird.log` é um arquivo chamado `--weird.log`.)
- Opção desconhecida → erro de uso.
- Opção que exige valor e não tem → erro de uso.
- Zero ou ≥2 posicionais → erro de uso.
- `--since` > `--until` → erro de uso.
- `--module` é **repetível** e acumula: dois módulos é **união**, não interseção.
- `--top 0` é **válido** e produz lista vazia. Não confunda "zero" com "não informado".
- `-h`/`--help` e `-V`/`--version` → stdout, exit **0**.
- Erro de uso → stderr, **stdout vazio**, exit **1**.

### Filtros

**Aplicados antes da agregação.** Linha filtrada não entra em `lines_valid`.
`--since` é **inclusivo**, `--until` é **exclusivo** — as bordas exatas são
testadas (I04, I05), então não chute o operador.

Comparação temporal é lexicográfica sobre a string do timestamp (R6). O formato
foi escolhido para permitir isso — não implemente calendário.

Entra também `--max-invalid-report <N>`, que na Etapa 3 estava fixo em 5.

---

## Testes desta etapa

| Grupo | IDs da SPEC §9 | Tipo | Onde |
|---|---|---|---|
| Filtros na agregação | **U35, U36** | unitário | `src/stats.rs` |
| Parser de argumentos | **U37–U48** | unitário | `src/cli.rs` |
| Filtros pela API pública | **I02–I06** | integração | `tests/lib_api.rs` |
| Erro de uso, help, version | **E05, E06, E07** | e2e | `tests/cli.rs` |
| `--max-invalid-report` | **E11, E12** | e2e | `all_invalid.log` |
| Caminho com espaço e acento | **E13** | e2e | `lo g ção.log` |

Notas:

- **U39 (`--top abc`), U41 (`--top` sem valor), U46 (`--since abc`), U48
  (`--format xml`)** são os quatro casos onde é tentador dar `unwrap()` no
  parse. Não dê — a regra §3.6 vale aqui.
- **E11** usa `--max-invalid-report 2` contra `all_invalid.log` (5 inválidas):
  espere exatamente 2 linhas `warning: line ...` **mais** 1 linha de resumo.
- **E12** usa `--max-invalid-report 0`: nenhum detalhe, só o resumo.
- **E13** existe porque você está no Windows e o caminho tem espaço **e** acento.
  Se você concatenou strings em vez de usar `Path`/`OsStr`, ele pega.
- **I04/I05** são as bordas: timestamp exatamente igual a `--since` entra;
  exatamente igual a `--until` não entra.

---

## Definition of Done

- [ ] U35–U48 verdes
- [ ] I02–I06 verdes
- [ ] E05, E06, E07, E11, E12, E13 verdes
- [ ] `logstats --help` e `logstats -V` → exit 0, saída no **stdout**
- [ ] `logstats --frobnicate x.log` → exit 1, **stdout vazio**
- [ ] `--top=3` e `--top 3` produzem o mesmo `Config` (assertado, não conferido no olho)
- [ ] Todos os testes das Etapas 1–3 continuam verdes
- [ ] `cargo fmt --check` e `cargo clippy --all-targets -- -D warnings` limpos

## Para a revisão, me manda

1. `src/cli.rs` inteiro
2. Saída de `cargo test`
3. Uma frase sobre onde você decidiu separar *parsing* de *validação* — e por quê
