# Etapa 3 — Robustez de entrada, stderr e exit codes

**Estimativa:** 2–3h · **Entregável:** o binário para de mentir quando a entrada é feia · **Referência:** SPEC §4.2 (R7–R9), §6.3, §6.4

---

## Objetivo

Até aqui você só rodou contra um arquivo bem-comportado. Esta etapa é sobre tudo
que dá errado: arquivo vazio, só lixo, CRLF do Windows, última linha sem `\n`,
bytes que não são UTF-8, arquivo que não existe, caminho que é um diretório.

Entra também o canal stderr (os `warning:`) e os exit codes 0/2/3.

Ainda sem flags. Exit 1 (erro de uso) só na Etapa 4.

---

## O que você vai encostar

| Tópico | Onde |
|---|---|
| Error handling de verdade: enum + `Display` + `std::error::Error` + `From` | `error.rs` |
| `?` atravessando tipos de erro diferentes | `io::Error` → `LogStatsError` |
| `std::process::exit` e separação stdout/stderr | `main.rs` |
| Bytes vs texto: `read_until` / `from_utf8` | leitura tolerante a UTF-8 inválido |
| `stdin()` como `impl BufRead` | entrada `-` |

---

## Escopo de código

Entra `src/error.rs` com `LogStatsError` e `impl From<io::Error>`. `main.rs`
passa a mapear erro → exit code. `stats.rs` passa a coletar as linhas inválidas
`(número, motivo)`.

### A mudança estrutural desta etapa

`BufRead::lines()` te dá `Result<String, io::Error>` e **falha a leitura inteira**
no primeiro byte não-UTF-8. A regra R8 diz que a linha vira inválida e o
**processamento continua**. Ou seja: `lines()` não serve mais.

Troque por `read_until(b'\n', &mut buf)` + `String::from_utf8_lossy` /
`str::from_utf8`, e trate o `Err` como `ParseError::InvalidUtf8` daquela linha.
Aproveite e remova o `\r` final ali (R7) e trate o caso da última linha sem `\n` (R9).

---

## Regras que entram

**R7** (`\n` e `\r\n`), **R8** (UTF-8 inválido → linha inválida, processamento
continua), **R9** (última linha sem `\n` processada normalmente).

### stderr (SPEC §6.3)

Uma linha por linha inválida, até 5 (o default de `--max-invalid-report`, que
ainda é fixo nesta etapa):

```
warning: line 8: bad_timestamp
```

Motivos com **exatamente** estes identificadores: `too_few_tokens`,
`bad_timestamp`, `bad_level`, `bad_module`, `field_without_eq`, `bad_key`,
`unterminated_quote`, `bad_duration`, `invalid_utf8`.

Se passou do limite, uma linha final: `warning: 12 more invalid lines suppressed`.

### Exit codes (SPEC §6.4)

| Código | Quando |
|---|---|
| 0 | sucesso, inclusive arquivo vazio |
| 2 | erro de I/O: arquivo inexistente, sem permissão, é um diretório |
| 3 | `lines_valid == 0` **e** `lines_total > 0` — e o relatório **ainda é impresso** no stdout |

Erro de I/O vai para stderr e **stdout fica vazio**. Esse "stdout vazio" é
assertado no E03; é fácil vazar um `println!` antes do erro.

---

## Testes desta etapa

| Grupo | IDs da SPEC §9 | Tipo | Fixture |
|---|---|---|---|
| Robustez de linha | **U17, U18, U19** (revisitados) | unitário | — |
| Entradas degeneradas | **I07, I08, I09, I10, I11** | integração | `all_invalid`, `empty`, `no_trailing_newline`, `crlf`, `utf8_mixed` |
| Erros de I/O | **E03, E04** | e2e | — |
| stdin | **E08** | e2e | `sample.log` via pipe |
| Exit 3 e vazio | **E09, E10** | e2e | `all_invalid.log`, `empty.log` |
| stderr do caso canônico | **E01** (completar a parte de stderr) | e2e | `sample.log` |

**I11 é o teste que importa mais nesta etapa:** ele não checa só que a linha 2 de
`utf8_mixed.log` foi contada como inválida — checa que as linhas 3 e 4 **continuaram
sendo processadas**. Se você abortar a leitura no byte ruim, ele pega.

**Cuidado com o I10:** se o `crlf.log` chegou até você com os `\r\n` normalizados
para `\n` pelo git, o teste passa sem testar nada. Copie o `.gitattributes` desta
pasta para a raiz do repositório antes de commitar as fixtures, e confira:

```bash
python -c "print(open('tests/fixtures/crlf.log','rb').read()[:80])"
```

Tem que aparecer `\r\n`.

---

## Definition of Done

- [ ] I07–I11 verdes
- [ ] E01 (agora com stderr), E03, E04, E08, E09, E10 verdes
- [ ] `logstats arquivo_que_nao_existe.log` → exit 2, stderr com mensagem, **stdout vazio**
- [ ] `logstats tests/fixtures/all_invalid.log` → exit 3 **e o relatório sai no stdout mesmo assim**
- [ ] `logstats tests/fixtures/empty.log` → exit 0, tudo zerado, `  (none)` e `  (no samples)`
- [ ] `cat tests/fixtures/sample.log | logstats -` → mesmo relatório do E01, com `file: -`
- [ ] Nenhum `read_to_string`; nenhum `unwrap()` em `src/`
- [ ] `cargo fmt --check` e `cargo clippy --all-targets -- -D warnings` limpos

## Para a revisão, me manda

1. `src/error.rs` e o trecho de leitura de linhas de `stats.rs`
2. `tests/lib_api.rs`
3. Os 3 canais (stdout, stderr, exit code) de `all_invalid.log` e de `empty.log`
