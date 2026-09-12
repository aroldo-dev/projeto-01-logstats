# Entrega 5 — Saída JSON, requisitos não funcionais e aceite final

**Estimativa de referência:** 2–3h · **Seções da SPEC:** §2, §5.2, §6.3, §7.5, §9

---

## Por que esta entrega

A pipeline de CI precisa consumir o resultado sem interpretar texto. E a v1 só
está pronta quando os requisitos não funcionais — memória, desempenho, robustez —
forem medidos, não presumidos.

## Histórias

> **Como** pipeline de CI, **quero** o relatório em JSON com estrutura fixa,
> **para** comparar execuções e alimentar dashboards.

> **Como** engenheiro de plantão, **quero** rodar a ferramenta num arquivo de
> vários GB sem travar a máquina, **para** usá-la no log de produção de verdade.

---

## Escopo

- `--format json` (SPEC §5.2), igual a SPEC §6.3 para `sample.log`.
- Medição de RNF2 e RNF3 (E14).
- Portões de qualidade Q01–Q05.
- Propriedades P01–P04 (opcionais, recomendadas).

## Regras de negócio em destaque

- **Uma linha**, compacta, sem espaços.
- **Ordem de chaves fixa**, exatamente como em SPEC §6.3. O QA compara texto:
  "um JSON equivalente" com outra ordem é defeito.
- `mean` com 1 casa decimal, como no texto.
- Sem amostras de duração: métricas `null`. Ranking vazio: `[]`.
- No campo `file`: `"`, `\` e caracteres de controle escapados.

---

## Critérios de aceite

| Comando | stdout | stderr | Código |
|---|---|---|---|
| `logstats --format json tests/fixtures/sample.log` | = `expected/sample.json.txt` | = `expected/sample.stderr.txt` | 0 |
| `logstats --format json tests/fixtures/empty.log` | JSON zerado, com `[]` e métricas `null` | vazio | 0 |
| `logstats --format json tests/fixtures/all_invalid.log` | JSON com `"lines_valid":0` | 5 avisos | 3 |
| E14 com 200.000 linhas | — | — | 0, em menos de 3 s |
| E14 com 2.000.000 linhas | — | — | 0, com pico de memória praticamente igual ao de 200.000 |

## Casos de teste (SPEC §7)

| Grupo | IDs | Nível |
|---|---|---|
| JSON | **U52, U53, U54** | unitário |
| JSON ponta a ponta | **E02** | ponta a ponta |
| Memória e desempenho | **E14** | não funcional, sob demanda |
| Propriedades | **P01–P04** | propriedade (opcional) |
| Portões | **Q01–Q05** | qualidade |

Sobre o **E14**: a massa de teste é gerada na hora, em diretório temporário. Se a
memória crescer junto com o arquivo, o RNF2 não foi atendido — e esse é o defeito
mais caro de corrigir no fim.

Sobre o **P01**: é o caso que costuma achar o que os exemplos não acharam. O QA vai
jogar texto aleatório — inclusive caracteres multibyte em posições de tamanho
fixo — contra a validação de linha.

---

## Definition of Done — v1 (SPEC §9)

- [ ] Todos os casos U, I e E automatizados e verdes
- [ ] Q01–Q05 atendidos
- [ ] E14 executado e resultado registrado
- [ ] SPEC §6.2 e §6.3 reproduzidos exatamente, nos três canais
- [ ] Nenhuma dependência de terceiros no produto (RNF1)
- [ ] Documentação de uso para o usuário final: exemplos, formato do log, códigos de saída

## Pacote de entrega final para o QA

1. Saída completa da suíte de testes
2. E14: tempo e pico de memória com 200.000 e com 2.000.000 linhas
3. Evidência de Q01–Q05, incluindo o relatório de cobertura
4. stdout, stderr e código de saída de cada linha da tabela de critérios de aceite
5. A documentação de uso
6. Limitações conhecidas, se houver

Com o aceite final, o Projeto 02 é liberado. Os candidatos estão em SPEC §10.
