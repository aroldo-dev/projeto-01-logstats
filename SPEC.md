# `logstats` — Especificação de Produto e Plano de Testes

**Autores:** Produto + QA · **Destinatário:** time de desenvolvimento (você)
**Nível:** básico (primeiro projeto) · **Estimativa de referência:** 13–18h, em 5 entregas

Este documento diz **o que** o `logstats` precisa fazer e **como o QA vai verificar**
que ele faz. Ele não diz **como** construir: arquitetura, organização do código,
nomes, tipos e técnicas são decisões do time de desenvolvimento.

A suíte de testes valida este contrato — não uma implementação específica.

---

## 0. Como este documento é usado

Este é o **contrato final** da v1. A ordem de construção está em `etapas/`, em 5
entregas incrementais. Cada entrega tem histórias, critérios de aceite, o recorte
do plano de testes (§7) e a sua própria *Definition of Done*.

| # | Entrega | Seções desta spec | Casos de teste |
|---|---|---|---|
| 1 | [Classificação de linhas](etapas/etapa-1-classificacao.md) | §3, §6.1 | U01–U23, U55 |
| 2 | [Relatório em texto](etapas/etapa-2-relatorio.md) | §5.1, §6.2 | U24–U34, U49–U51, I01, I12–I13, E01 |
| 3 | [Entradas problemáticas, avisos e códigos de saída](etapas/etapa-3-robustez.md) | §3.2 (R7–R9), §5.3, §5.4 | U17–U19, I07–I11, E01, E03–E04, E08–E10 |
| 4 | [Opções de linha de comando e filtros](etapas/etapa-4-opcoes-e-filtros.md) | §4 | U35–U48, I02–I06, E05–E07, E11–E13, E15 |
| 5 | [Saída JSON, requisitos não funcionais e aceite final](etapas/etapa-5-json-e-aceite-final.md) | §2, §5.2, §6.3, §7.5, §9 | U52–U54, E02, E14, P01–P04, Q01–Q05 |

Leia esta spec inteira uma vez antes de começar: saber onde o produto chega evita
decisões na Entrega 1 que custam caro na Entrega 3. Depois, uma entrega por vez.

---

## 1. Contexto do produto

### Problema

Times de plataforma precisam responder rápido: **"o que quebrou nas últimas horas
e onde está lento?"**. Ferramentas completas (Datadog, Loki) são caras, exigem
ingestão e não estão disponíveis em todo ambiente.

### Proposta

Um executável único que lê um arquivo de log estruturado e emite um relatório
**determinístico**: contagens por nível, ranking de módulos com mais erros e
estatísticas de latência.

### Quem usa

| Persona | Como usa | O que importa para ela |
|---|---|---|
| **Engenheiro de plantão** | Roda no terminal durante um incidente | Resposta rápida, leitura fácil, funciona em qualquer máquina (inclusive Windows) |
| **Pipeline de CI** | Roda sobre os logs de um teste de carga | Saída estável para comparar, JSON para consumir, códigos de saída que distinguem "deu certo" de "entrada inútil" |

### Não-objetivos da v1

Aceitar formatos de log arbitrários, indexar, servir HTTP, acompanhar arquivo em
tempo real.

---

## 2. Requisitos não funcionais

| # | Requisito | Como o QA verifica |
|---|---|---|
| RNF1 | **Distribuição:** um único executável, **sem dependências de terceiros** (política de segurança da cadeia de suprimentos). Ferramentas usadas apenas nos testes são permitidas. Plataforma padrão: Rust estável. | Inspeção do manifesto do projeto |
| RNF2 | **Memória limitada:** o consumo pode crescer com o número de módulos distintos e de amostras de duração (necessárias para percentis exatos), mas **não com o volume de texto lido**. Precisa processar arquivos maiores que a RAM e entrada contínua via pipe. | E14 |
| RNF3 | **Desempenho:** 200.000 linhas processadas em menos de 3 s numa máquina de desenvolvimento comum. | E14 |
| RNF4 | **Robustez:** nenhuma entrada — conteúdo do arquivo, stdin ou argumentos — pode fazer a ferramenta encerrar de forma abrupta. Toda execução termina com um dos códigos de §5.4; **qualquer outro código de saída é defeito**. | P01, §8, toda a suíte E2E |
| RNF5 | **Determinismo:** mesma entrada + mesmos argumentos → saída idêntica byte a byte, em qualquer execução. | U54, E01 repetido |
| RNF6 | **Windows é plataforma de primeira classe:** terminadores CRLF e caminhos com espaço e acento funcionam como em Linux/macOS. | I10, E13 |
| RNF7 | **Padrão de engenharia:** formatação padrão da linguagem, linter oficial sem avisos, cobertura mínima nas regras de negócio. | Q01–Q05 |

---

## 3. Formato de entrada

### 3.1 Gramática

```
linha      := timestamp SP+ level SP+ module ( SP+ campo )*
timestamp  := YYYY-MM-DDTHH:MM:SSZ        (exatamente 20 caracteres, UTC)
level      := TRACE | DEBUG | INFO | WARN | ERROR    (maiúsculas, exatamente assim)
module     := [a-z0-9_]{1,32}
campo      := chave "=" valor
chave      := [a-z0-9_]+
valor      := token_sem_espaco | '"' ( qualquer caractere exceto '"' )* '"'
```

`SP+` = um ou mais espaços.

Exemplo:

```
2026-03-14T10:22:33Z ERROR db    query=select_users duration_ms=1503 error="connection timeout"
```

### 3.2 Regras de negócio

| # | Regra |
|---|---|
| R1 | Linha vazia ou só com espaços → **em branco**: não é válida nem inválida, tem contador próprio. |
| R2 | Linha que não segue a gramática → **inválida**: é contada, registrada como (número da linha, motivo), e o processamento **continua**. |
| R3 | Chave repetida na mesma linha → o **último** valor vence. |
| R4 | O campo `duration_ms`, se presente, precisa ser um inteiro não negativo que caiba em 64 bits. Vazio, texto, negativo ou grande demais → linha inválida (`bad_duration`). |
| R5 | O timestamp é validado na forma **e** nos valores: mês 1–12, dia 1–31 (não é preciso cruzar mês×dia nem tratar ano bissexto), hora 0–23, minuto e segundo 0–59. |
| R6 | Ordem temporal = ordem alfabética do texto do timestamp. O formato foi escolhido para isso; não há cálculo de calendário no produto. |
| R7 | Linhas terminadas em `LF` e em `CRLF` são igualmente suportadas. |
| R8 | Linha com bytes que não formam UTF-8 válido → inválida (`invalid_utf8`), e o processamento **continua** nas linhas seguintes. |
| R9 | A última linha sem terminador é processada normalmente. |
| R10 | Valores podem conter caracteres acentuados e emoji sem afetar nada. |

Motivos de rejeição — identificadores oficiais, usados na saída (§5.3) e nos testes:

| Identificador | Quando |
|---|---|
| `too_few_tokens` | menos de 3 elementos (timestamp, nível, módulo) |
| `bad_timestamp` | timestamp fora do formato ou com valores fora da faixa (R5) |
| `bad_level` | nível fora da lista |
| `bad_module` | módulo fora de `[a-z0-9_]{1,32}` |
| `field_without_eq` | campo sem `=` |
| `bad_key` | chave vazia ou com caractere fora de `[a-z0-9_]` |
| `unterminated_quote` | valor entre aspas sem aspa de fechamento |
| `bad_duration` | `duration_ms` inválido (R4) |
| `invalid_utf8` | bytes não UTF-8 (R8) |

Se uma linha tem mais de um problema, vale o **primeiro** encontrado lendo da
esquerda para a direita (timestamp → nível → módulo → campos).

---

## 4. Interface de linha de comando

```
logstats [OPÇÕES] <ARQUIVO>
logstats [OPÇÕES] -            # lê de stdin
```

| Opção | Padrão | Comportamento |
|---|---|---|
| `--level <LEVEL>` | `TRACE` | Considera só linhas com nível **maior ou igual** a LEVEL. Ordem: TRACE < DEBUG < INFO < WARN < ERROR. |
| `--module <NOME>` | (todos) | Repetível. Se informado ao menos uma vez, só esses módulos entram (união). |
| `--since <TS>` | (sem limite) | Timestamp no formato de §3.1. **Inclusivo.** |
| `--until <TS>` | (sem limite) | Timestamp no formato de §3.1. **Exclusivo.** |
| `--top <N>` | `5` | Quantos módulos listar no ranking de erros. `0` é válido → lista vazia. |
| `--format <text\|json>` | `text` | Formato do relatório. |
| `--max-invalid-report <N>` | `5` | Quantas linhas inválidas detalhar no stderr. |
| `-h`, `--help` | — | Texto de uso no stdout (contém `Usage`), código 0. |
| `-V`, `--version` | — | `logstats <versão do pacote>` no stdout (ex.: `logstats 0.1.0`), código 0. |

Regras de interpretação dos argumentos:

- `--opcao valor` e `--opcao=valor` são equivalentes.
- `--` encerra as opções; tudo depois é tratado como nome de arquivo.
- Opção desconhecida → erro de uso.
- Opção que exige valor e não recebeu → erro de uso.
- Valor inválido para a opção (`--top abc`, `--format xml`, `--since ontem`) → erro de uso.
- Nenhum arquivo, ou mais de um → erro de uso.
- `--since` posterior a `--until` → erro de uso.

Filtros:

- São aplicados **antes** da agregação: linha válida filtrada não entra em
  `lines_valid`, no ranking nem nas durações.
- `lines_total` conta todas as linhas lidas. Linhas inválidas e em branco são
  contadas e reportadas independentemente dos filtros.

---

## 5. Saída

### 5.1 Relatório em texto (padrão)

O QA compara **byte a byte**. Estrutura:

- Blocos separados por **exatamente uma** linha em branco; a saída termina com quebra de linha.
- `file:` mostra o caminho **exatamente como foi informado** (ou `-` para stdin).
- Linhas de item: 2 espaços, o nome alinhado à esquerda e completado com espaços
  até a largura do maior nome **do bloco**, 2 espaços, e o valor alinhado à
  direita até a largura do maior valor **do bloco**.
- `by_level` lista sempre os 5 níveis, de TRACE a ERROR, mesmo com zero.
- `top_modules_by_error (N)` lista só módulos com ao menos 1 ERROR, ordenados por
  quantidade de erros (maior primeiro) e, no empate, por nome (A→Z). `N` é a
  quantidade listada. Lista vazia → a linha `  (none)`.
- `duration_ms (samples: N)` considera as linhas que têm `duration_ms`:
  - `mean` com 1 casa decimal, arredondando metades para longe do zero (`1.25 → 1.3`, `1.35 → 1.4`).
  - `p50`, `p90`, `p95` pelo método **nearest-rank**, sem interpolação: com os
    valores ordenados e numerados a partir de 1, o percentil *p* é o valor na
    posição `ceil(p/100 × n)`, limitada entre 1 e n.
  - `max` é o maior valor.
  - Sem amostras → o bloco tem apenas a linha `  (no samples)`.

Exemplo completo em §6.2.

### 5.2 Relatório JSON (`--format json`)

- Uma linha, compacta (sem espaços), terminada em quebra de linha.
- **Ordem de chaves fixa**, exatamente como em §6.3. O QA compara texto, não
  "JSON equivalente".
- `mean` com 1 casa decimal, como no texto (ex.: `12.0`).
- Sem amostras de duração: `"duration_ms":{"samples":0,"mean":null,"p50":null,"p90":null,"p95":null,"max":null}`.
- Ranking vazio: `"top_modules_by_error":[]`.
- No campo `file`, `"` e `\` são escapados, e caracteres de controle viram `\u00XX`.

### 5.3 Avisos (stderr)

Uma linha por linha inválida, na ordem do arquivo, até o limite de
`--max-invalid-report`:

```
warning: line 8: bad_timestamp
```

Se houver mais inválidas que o limite, uma linha final com a quantidade omitida:

```
warning: 12 more invalid lines suppressed
```

### 5.4 Códigos de saída

| Código | Quando |
|---|---|
| 0 | Sucesso — inclusive arquivo vazio, `--help` e `--version` |
| 1 | Erro de uso (argumentos) |
| 2 | Erro de leitura: arquivo inexistente, sem permissão, caminho é um diretório |
| 3 | O arquivo tem linhas, mas **nenhuma é válida pela gramática** (§3): todas são inválidas ou em branco. O relatório **é impresso** no stdout mesmo assim. Filtros não influenciam: se há linhas válidas e os filtros excluem todas, o código é 0 e o relatório sai zerado. |

Nos códigos 1 e 2: mensagem explicativa no stderr e **stdout vazio**.

---

## 6. Caso de aceitação canônico

A massa de teste está em `tests/fixtures/` e as saídas esperadas em `expected/`.

### 6.1 Entrada: `tests/fixtures/sample.log`

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

Inválidas: linha 8 `too_few_tokens`, linha 11 `bad_level`, linha 15 `bad_duration`.
Linha 16 em branco.

### 6.2 `logstats tests/fixtures/sample.log`

stdout (= `expected/sample.stdout.txt`):

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

Conferência: durações ordenadas `[3, 7, 12, 15, 25, 38, 42, 87, 310, 640, 980, 1503]`,
soma 3662, n = 12, média 305,1666… → `305.2`.

stderr (= `expected/sample.stderr.txt`):

```
warning: line 8: too_few_tokens
warning: line 11: bad_level
warning: line 15: bad_duration
```

Código de saída: 0.

### 6.3 `logstats --format json tests/fixtures/sample.log`

stdout (= `expected/sample.json.txt`):

```json
{"file":"tests/fixtures/sample.log","lines_total":20,"lines_valid":16,"lines_invalid":3,"lines_blank":1,"by_level":{"TRACE":1,"DEBUG":1,"INFO":7,"WARN":3,"ERROR":4},"top_modules_by_error":[{"module":"db","errors":2},{"module":"auth","errors":1},{"module":"http","errors":1}],"duration_ms":{"samples":12,"mean":305.2,"p50":38,"p90":980,"p95":1503,"max":1503}}
```

stderr e código de saída: iguais a §6.2.

---

## 7. Plano de testes

### Níveis de teste exigidos

| Sigla | Nível | O que cobre |
|---|---|---|
| **U** | Unitário | Uma regra isolada — uma regra de validação, um cálculo, uma regra de argumento — sem ler arquivo nem executar o programa. |
| **I** | Integração | O fluxo completo (leitura → validação → filtros → agregação) exercitado **sem passar pela linha de comando**, com entradas em memória (sem arquivos em disco). |
| **E** | Ponta a ponta (E2E) | O executável real, como o usuário roda. Todo caso verifica os **três canais: stdout, stderr e código de saída.** |
| **P** | Propriedade | Invariantes verificadas com entradas geradas aleatoriamente. Opcional, recomendado. |
| **Q** | Portões de qualidade | Critérios que bloqueiam a entrega. |

O QA cobra que cada caso **exista, seja automatizado, rode na suíte padrão do
projeto e passe**. Onde e como cada teste é escrito é decisão sua.

Nos casos de validação, o critério é o **motivo da rejeição** (identificador de
§3.2), nunca o texto de uma mensagem.

### 7.1 Unitários

**Validação de linha**

| ID | Caso | Esperado |
|---|---|---|
| U01 | Linha mínima válida, sem campos | válida, 0 campos |
| U02 | Linha com 3 campos | válida, 3 campos |
| U03 | Valor entre aspas contendo espaços | válida, valor sem as aspas |
| U04 | Valor entre aspas contendo `=` | válida |
| U05 | Chave repetida na mesma linha | válida, último valor vence |
| U06 | Vários espaços entre os elementos | válida |
| U07 | Timestamp sem `Z`; sem `T`; mês 13; hora 25; com 19 caracteres; com 21 caracteres | `bad_timestamp` (6 casos) |
| U08 | Nível `FATAL` | `bad_level` |
| U09 | Nível `info` (minúsculo) | `bad_level` |
| U10 | Módulo `Db`; `db-1`; vazio; com 33 caracteres | `bad_module` (4 casos) |
| U11 | Campo sem `=` | `field_without_eq` |
| U55 | Chave `Key`; `a-b`; chave vazia (`=valor`) | `bad_key` (3 casos) |
| U12 | `duration_ms=abc` | `bad_duration` |
| U13 | `duration_ms=` | `bad_duration` |
| U14 | `duration_ms=-1` | `bad_duration` |
| U15 | `duration_ms=99999999999999999999999` (não cabe em 64 bits) | `bad_duration` |
| U16 | Aspas sem fechamento | `unterminated_quote` |
| U17 | String vazia; só espaços | em branco |
| U18 | Linha com terminador CRLF | válida, sem resíduo do CR em nenhum campo |
| U19 | Valores com acento e emoji | válida, valores preservados |
| U20 | Menos de 3 elementos | `too_few_tokens` |

**Níveis**

| ID | Caso | Esperado |
|---|---|---|
| U21 | Os 5 níveis; mais `FATAL`, `Info`, `warning` | os 5 aceitos, os demais rejeitados |
| U22 | Para cada nível: o nome exibido no relatório | idêntico ao aceito na entrada |
| U23 | Ordem de severidade | TRACE < DEBUG < INFO < WARN < ERROR |

**Estatísticas**

| ID | Caso | Esperado |
|---|---|---|
| U24 | Entrada vazia | todos os contadores 0, 0 amostras |
| U25 | Contagem por nível | bate com a entrada |
| U26 | Ranking com empate | desempate por nome A→Z |
| U27 | `--top` maior que o nº de módulos com erro | lista todos |
| U28 | `--top 0` | lista vazia |
| U29 | Percentis com 1 amostra | p50 = p90 = p95 = o valor |
| U30 | Percentis de `[10, 20]` | p50 = 10, p90 = 20, p95 = 20 |
| U31 | Percentis de `1..=100` | p50 = 50, p90 = 90, p95 = 95 |
| U32 | Percentis com valores repetidos | corretos pelo nearest-rank |
| U33 | Média de `[1, 2]` e de `[1, 1, 2]` | `1.5` e `1.3` |
| U34 | Máximo | correto |
| U35 | Filtro de nível WARN | TRACE, DEBUG e INFO descartados |
| U36 | Filtro com 2 módulos | união, não interseção |

**Argumentos**

| ID | Caso | Esperado |
|---|---|---|
| U37 | `--top=3` e `--top 3` | mesma configuração resultante |
| U38 | Só o arquivo, nenhuma opção | padrões de §4 |
| U39 | `--top abc` | erro de uso |
| U40 | `--frobnicate` | erro de uso |
| U41 | `--top` no fim, sem valor | erro de uso |
| U42 | Dois arquivos | erro de uso |
| U43 | Nenhum arquivo | erro de uso |
| U44 | `-- --weird.log` | arquivo chamado `--weird.log` |
| U45 | `--module a --module b` | ambos considerados |
| U46 | `--since abc` | erro de uso |
| U47 | `--since` posterior a `--until` | erro de uso |
| U48 | `--format xml` | erro de uso |

**Formatação da saída**

| ID | Caso | Esperado |
|---|---|---|
| U49 | Módulos com nomes de tamanhos diferentes | alinhamento de §5.1 |
| U50 | 0 amostras de duração | `  (no samples)` |
| U51 | Ranking vazio | `  (none)` |
| U52 | JSON com `"` e `\` no caminho | escapados |
| U53 | JSON com 0 amostras | métricas `null` |
| U54 | JSON gerado 100 vezes a partir do mesmo resultado | 100 saídas idênticas |

### 7.2 Integração

Entradas fornecidas em memória, sem arquivos e sem executar o programa.

| ID | Caso | Esperado |
|---|---|---|
| I01 | Conteúdo de `sample.log` | os números de §6.2 |
| I02 | Filtro de nível WARN | só WARN e ERROR contam |
| I03 | Filtro de módulo `db` | só `db` conta |
| I04 | `--since` igual a um timestamp existente | linha **incluída** |
| I05 | `--until` igual a um timestamp existente | linha **excluída** |
| I06 | `--since` + `--until` + `--level` combinados | só a interseção dos três |
| I07 | Só linhas inválidas | `lines_valid` 0, `lines_invalid` = n |
| I08 | Entrada vazia (0 bytes) | tudo zerado |
| I09 | Última linha sem terminador | processada |
| I10 | CRLF em todas as linhas | mesmo resultado que com LF |
| I11 | Bytes UTF-8 inválidos no meio | aquela linha `invalid_utf8` **e as seguintes processadas** |
| I12 | Timestamps fora de ordem cronológica | resultado não muda |
| I13 | Mesmo módulo em 2 níveis diferentes | contagens independentes |

### 7.3 Ponta a ponta

| ID | Caso | Esperado |
|---|---|---|
| E01 | `sample.log` | stdout = §6.2, stderr = §6.2, código 0 |
| E02 | `--format json` com `sample.log` | stdout = §6.3, código 0 |
| E03 | Arquivo inexistente | código 2, stderr não vazio, **stdout vazio** |
| E04 | Caminho é um diretório | código 2, **stdout vazio** |
| E05 | `--frobnicate` | código 1, stderr não vazio, **stdout vazio** |
| E06 | `--help` | código 0, stdout contém `Usage` |
| E07 | `-V` | código 0, stdout contém a versão do pacote |
| E08 | `-` com o conteúdo de `sample.log` no stdin | mesmo que E01, exceto `file: -` |
| E09 | `all_invalid.log` | código **3**, relatório impresso no stdout |
| E10 | `empty.log` | código **0**, tudo zerado, `  (none)` e `  (no samples)` |
| E11 | `--max-invalid-report 2` com `all_invalid.log` | exatamente 2 linhas `warning: line …` + 1 linha de resumo |
| E12 | `--max-invalid-report 0` com `all_invalid.log` | só a linha de resumo |
| E13 | `lo g ção.log` (espaço e acento no caminho) | funciona, `file:` com o caminho exato |
| E14 | **Não funcional** — 200.000 linhas geradas em diretório temporário | < 3 s; pico de memória praticamente igual ao de 2.000.000 linhas |
| E15 | `--module payments` com `sample.log` (nenhuma linha desse módulo) | código **0** (não 3), `lines_valid: 0`, os 3 avisos de §6.2 |

O E14 fica fora da execução padrão da suíte e roda sob demanda. A medição de
memória pode ser manual, desde que o resultado venha no pacote de entrega.

### 7.4 Propriedade (opcional, recomendado)

| ID | Propriedade |
|---|---|
| P01 | Para **qualquer** texto como linha, a validação sempre termina com uma classificação — válida, inválida com motivo de §3.2, ou em branco. Nunca trava. |
| P02 | Um registro válido gerado aleatoriamente e escrito conforme §3.1 é aceito, com os mesmos timestamp, nível, módulo e campos. |
| P03 | Para qualquer conjunto não vazio de durações: p50 ≤ p90 ≤ p95 ≤ max, e cada percentil é um dos valores da amostra. |
| P04 | Para qualquer conjunto não vazio de durações: menor valor ≤ mean ≤ max. |

### 7.5 Portões de qualidade

| ID | Portão |
|---|---|
| Q01 | Código na formatação padrão da linguagem, sem divergências |
| Q02 | Linter oficial da linguagem sem nenhum aviso, incluindo o código de teste |
| Q03 | Suíte completa verde (unitários, integração, ponta a ponta) |
| Q04 | Cobertura de linhas ≥ 85% no código das regras de validação (§3) e de estatística (§5.1) |
| Q05 | Todos os ataques de §8 executados sem encerramento abrupto (RNF4) |

---

## 8. Onde o QA vai atacar

Não são pegadinhas gratuitas: são os defeitos que mais aparecem em ferramentas
desse tipo. O QA vai além dos casos de §7 nesses pontos.

1. **Caracteres multibyte em qualquer posição** — inclusive onde o produto espera
   um tamanho fixo, como o timestamp. (U19, P01)
2. **Arquivos gerados no Windows** (CRLF), inclusive com linhas em branco. (U18, I10)
3. **Espaçamento irregular** entre os elementos. (U06)
4. **Números fora da faixa** em `duration_ms`. (U15)
5. **Arquivos grandes e entrada contínua via pipe.** (E14, E08)
6. **Determinismo:** a mesma execução repetida várias vezes precisa dar a mesma saída. (U54, E01)
7. **Arredondamento na borda** da média. (U33)
8. **Caminhos com espaço e acento.** (E13)
9. **stdout poluído em erro:** nada pode sair no stdout antes de um erro de uso ou de leitura. (E03, E05)
10. **Lixo no meio do arquivo** não pode derrubar o resto do processamento. (I11)

---

## 9. Definition of Done da v1

- [ ] Todos os casos U, I e E de §7 automatizados e verdes
- [ ] Q01–Q05 atendidos
- [ ] E14 executado e resultado registrado
- [ ] §6.2 e §6.3 reproduzidos exatamente, com os três canais
- [ ] Nenhuma dependência de terceiros no produto (RNF1)
- [ ] Documentação de uso para o usuário final: exemplos, formato do log, códigos de saída
- [ ] Aceite do QA (§11)

---

## 10. Roadmap do produto após a v1

Nada disto entra na v1. São candidatos ao Projeto 02.

| Ideia | Valor para o usuário |
|---|---|
| Modo acompanhamento (`--follow`) | Relatório atualizado enquanto o arquivo cresce, durante o incidente |
| Alto volume | 10 milhões de linhas em poucos segundos em máquinas com vários núcleos |
| Campanha de robustez | QA gera milhões de linhas malformadas contra a validação de linha |

---

## 11. Fluxo de entrega e aceite

A cada entrega:

1. **Dev** fecha a *Definition of Done* da entrega.
2. **Dev** envia o **pacote de entrega**:
   - resultado completo da suíte de testes;
   - resultado dos portões de qualidade que valem na entrega;
   - evidência dos critérios de aceite: stdout, stderr e código de saída dos comandos listados;
   - limitações conhecidas, se houver.
3. **QA** executa os critérios de aceite por conta própria, roda casos
   adversariais extras e devolve **aprovado** ou uma lista de **bugs** (passos
   para reproduzir, esperado × obtido, severidade).
4. Com a aprovação, **Produto** libera a próxima entrega.

Revisão de código não faz parte do aceite de Produto e QA. Para feedback sobre
o código, peça ao tutor no chat.
