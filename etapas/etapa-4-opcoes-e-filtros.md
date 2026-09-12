# Entrega 4 — Opções de linha de comando e filtros

**Estimativa de referência:** 3–4h · **Seções da SPEC:** §4

---

## Por que esta entrega

Durante um incidente, a pergunta raramente é "me mostra tudo". É "só os erros do
`db` desde as 10h". Sem filtros, o usuário volta para o `grep`.

## Histórias

> **Como** engenheiro de plantão, **quero** filtrar por nível mínimo, **para**
> ignorar o ruído de TRACE e DEBUG.

> **Como** engenheiro de plantão, **quero** filtrar por um ou mais módulos,
> **para** focar no serviço suspeito.

> **Como** engenheiro de plantão, **quero** filtrar por janela de tempo, **para**
> olhar só o período do incidente.

> **Como** engenheiro de plantão, **quero** controlar o tamanho do ranking e a
> quantidade de avisos, **para** ajustar o relatório à situação.

> **Como** usuário novo, **quero** `--help` e `--version`, **para** usar a
> ferramenta sem ler a documentação.

> **Como** pipeline de CI, **quero** que um erro de argumento tenha código próprio
> (1) e não suje o stdout, **para** diferenciar "chamei errado" de "o log está ruim".

---

## Escopo

- Todas as opções e regras de interpretação de SPEC §4.
- `--format` já é aceita e validada (`--format xml` é erro de uso). A saída JSON
  chega na Entrega 5; até lá, `--format json` pode produzir o relatório em texto.
- Código de saída **1** (erro de uso).

## Fora do escopo

Saída JSON.

## Regras de negócio em destaque

- **Bordas de tempo:** `--since` é **inclusivo**, `--until` é **exclusivo**. O QA
  testa o timestamp exatamente igual à borda (I04, I05).
- **`--top 0` é válido** e produz lista vazia. "Zero" não é "não informado".
- **`--module` repetido soma** módulos (união), não restringe (interseção).
- Filtros valem **antes** da agregação. Linhas inválidas e em branco são contadas
  e avisadas independentemente dos filtros.
- **Filtro que não encontra nada não é erro:** se o arquivo tem linhas válidas e
  os filtros excluem todas, o código é **0** (não 3) e o relatório sai zerado. O
  código 3 é só para arquivo sem nenhuma linha válida (SPEC §5.4).
- `--since` e `--until` usam o mesmo formato do timestamp do log (§3.1).
- `--help` e `--version` saem no **stdout**, com código 0. Erro de uso sai no
  stderr, com **stdout vazio** e código 1.

---

## Critérios de aceite

| Comando | stdout | stderr | Código |
|---|---|---|---|
| `logstats --help` | contém `Usage` | vazio | 0 |
| `logstats -V` | `logstats 0.1.0` (a versão do pacote) | vazio | 0 |
| `logstats --frobnicate tests/fixtures/sample.log` | **vazio** | mensagem de erro | 1 |
| `logstats --top abc tests/fixtures/sample.log` | **vazio** | mensagem de erro | 1 |
| `logstats --level ERROR tests/fixtures/sample.log` | `lines_valid: 4`; `by_level` só com ERROR 4 | os 3 avisos de sempre | 0 |
| `logstats --module db --module auth tests/fixtures/sample.log` | `lines_valid: 9` | os 3 avisos de sempre | 0 |
| `logstats --since 2026-03-14T10:22:40Z --until 2026-03-14T10:22:47Z tests/fixtures/sample.log` | `lines_valid: 5` (entra a linha das 10:22:40, sai a das 10:22:47) | os 3 avisos de sempre | 0 |
| `logstats --module payments tests/fixtures/sample.log` | `lines_valid: 0`; `lines_total`, `lines_invalid` e `lines_blank` como sempre | os 3 avisos de sempre | **0** |
| `logstats --top 0 tests/fixtures/sample.log` | `top_modules_by_error (0):` seguido de `  (none)` | os 3 avisos de sempre | 0 |
| `logstats --max-invalid-report 1 tests/fixtures/sample.log` | relatório normal | `warning: line 8: too_few_tokens` + `warning: 2 more invalid lines suppressed` | 0 |
| `logstats "tests/fixtures/lo g ção.log"` | `file:` com o caminho exato; total 2, válidas 2 | vazio | 0 |
| `logstats -- --weird.log` | **vazio** | erro de leitura (arquivo `--weird.log` não existe) | 2 |

## Casos de teste (SPEC §7)

| Grupo | IDs | Nível |
|---|---|---|
| Filtros | **U35, U36** | unitário |
| Argumentos | **U37–U48** | unitário |
| Filtros no fluxo completo | **I02–I06** | integração |
| Erro de uso, ajuda e versão | **E05, E06, E07** | ponta a ponta |
| Limite de avisos | **E11, E12** | ponta a ponta |
| Caminho com espaço e acento | **E13** | ponta a ponta |
| Filtro que exclui todas as linhas | **E15** | ponta a ponta |

Destaques do QA:

- **U39, U41, U46, U48** são valores ausentes ou inválidos: todos precisam virar
  erro de uso (código 1), nunca um encerramento abrupto (RNF4).
- **E11:** `--max-invalid-report 2` com `all_invalid.log` (5 inválidas) → exatamente
  2 linhas `warning: line …` **mais** 1 linha de resumo.
- **E12:** `--max-invalid-report 0` → só a linha de resumo.
- **E13:** o caminho tem espaço **e** acento. É um caso clássico de falha no Windows.

---

## Definition of Done

- [ ] Critérios de aceite atendidos, nos três canais
- [ ] Casos desta entrega automatizados e verdes
- [ ] Casos das Entregas 1 a 3 continuam verdes
- [ ] Q01 e Q02 atendidos

## Pacote de entrega para o QA

1. Saída completa da suíte de testes
2. Saída da verificação de formatação e do linter
3. stdout, stderr e código de saída de cada linha da tabela de critérios de aceite
4. Limitações conhecidas, se houver
