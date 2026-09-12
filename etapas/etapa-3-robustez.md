# Entrega 3 — Entradas problemáticas, avisos e códigos de saída

**Estimativa de referência:** 2–3h · **Seções da SPEC:** §3.2 (R7–R9), §5.3, §5.4

---

## Por que esta entrega

Até aqui a ferramenta só viu um arquivo bem-comportado. Na vida real o arquivo
chega vazio, cheio de lixo, gerado no Windows, cortado no meio, com bytes
corrompidos — ou nem existe. E a pipeline de CI precisa distinguir esses casos
pelo código de saída, sem ler texto.

## Histórias

> **Como** engenheiro de plantão, **quero** que arquivos gerados no Windows, sem
> quebra de linha no final ou com trechos corrompidos sejam processados até o
> fim, **para** não perder o relatório por causa de uma linha ruim.

> **Como** engenheiro de plantão, **quero** ver no stderr quais linhas foram
> rejeitadas e por quê, **para** investigar o log sem poluir o relatório.

> **Como** pipeline de CI, **quero** códigos de saída diferentes para sucesso,
> erro de leitura e "nenhuma linha válida", **para** decidir o próximo passo sem
> interpretar texto.

> **Como** engenheiro de plantão, **quero** passar o log via pipe usando `-`,
> **para** encadear o `logstats` com outras ferramentas.

---

## Escopo

- Regras **R7** (LF e CRLF), **R8** (UTF-8 inválido) e **R9** (última linha sem terminador).
- Avisos no stderr (SPEC §5.3), com limite fixo de 5 — a opção
  `--max-invalid-report` chega na Entrega 4.
- Códigos de saída **0, 2 e 3** (SPEC §5.4). O código 1 chega na Entrega 4.
- `-` como nome de arquivo lê do stdin.

## Fora do escopo

Opções de linha de comando e JSON.

## Regras de negócio em destaque

- Linha com bytes corrompidos é **uma** linha inválida. As linhas seguintes
  continuam sendo processadas normalmente.
- No erro de leitura (código 2), **nada** pode sair no stdout.
- No código 3, o relatório **sai** no stdout mesmo assim.

---

## Critérios de aceite

| Comando | stdout | stderr | Código |
|---|---|---|---|
| `logstats tests/fixtures/sample.log` | = `expected/sample.stdout.txt` | = `expected/sample.stderr.txt` | 0 |
| `logstats nao_existe.log` | **vazio** | mensagem de erro | 2 |
| `logstats tests/fixtures` (um diretório) | **vazio** | mensagem de erro | 2 |
| `logstats tests/fixtures/all_invalid.log` | relatório, com `lines_valid: 0` | 5 avisos (SPEC §5.3) | **3** |
| `logstats tests/fixtures/empty.log` | relatório zerado, com `  (none)` e `  (no samples)` | vazio | 0 |
| `logstats tests/fixtures/crlf.log` | total 5, válidas 4, inválidas 0, em branco 1 | vazio | 0 |
| `logstats tests/fixtures/utf8_mixed.log` | total 4, válidas 3, inválidas 1, em branco 0 | `warning: line 2: invalid_utf8` | 0 |
| `logstats tests/fixtures/no_trailing_newline.log` | total 3, válidas 3 | vazio | 0 |
| conteúdo de `sample.log` via pipe para `logstats -` | igual ao da 1ª linha, com `file: -` | = `expected/sample.stderr.txt` | 0 |

## Casos de teste (SPEC §7)

| Grupo | IDs | Nível | Massa de teste |
|---|---|---|---|
| Validação revisitada | **U17, U18, U19** | unitário | — |
| Entradas degeneradas | **I07, I08, I09, I10, I11** | integração | conteúdos de `all_invalid`, `empty`, `no_trailing_newline`, `crlf`, `utf8_mixed` |
| Erros de leitura | **E03, E04** | ponta a ponta | — |
| stdin | **E08** | ponta a ponta | `sample.log` via pipe |
| Código 3 e arquivo vazio | **E09, E10** | ponta a ponta | `all_invalid.log`, `empty.log` |
| Caso canônico completo | **E01** — agora com stderr | ponta a ponta | `sample.log` |

O caso que o QA mais valoriza nesta entrega é o **I11**: não basta a linha 2 de
`utf8_mixed.log` ser inválida. As linhas 3 e 4 precisam **continuar sendo processadas**.

## Integridade da massa de teste

`crlf.log` precisa manter os terminadores CRLF. Se o controle de versão converter
os finais de linha, o I10 passa sem testar nada. Confira o arquivo depois de
clonar, antes de confiar no resultado do I10.

---

## Definition of Done

- [ ] Critérios de aceite atendidos, nos três canais
- [ ] Casos desta entrega automatizados e verdes
- [ ] Casos das Entregas 1 e 2 continuam verdes
- [ ] Q01 e Q02 atendidos

## Pacote de entrega para o QA

1. Saída completa da suíte de testes
2. Saída da verificação de formatação e do linter
3. stdout, stderr e código de saída de cada linha da tabela de critérios de aceite
4. Limitações conhecidas, se houver
