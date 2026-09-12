# Entrega 1 — Classificação de linhas

**Estimativa de referência:** 3–4h · **Seções da SPEC:** §3, §6.1

---

## Por que esta entrega

Antes de qualquer número, o produto precisa distinguir linha válida, inválida e em
branco. Tudo o que vem depois depende disso estar certo.

## História

> **Como** engenheiro de plantão,
> **quero** ver como cada linha do meu log foi classificada — válida, inválida (e
> por quê) ou em branco —
> **para** confiar que o relatório das próximas entregas conta as linhas certas.

---

## Escopo

`logstats <ARQUIVO>` lê o arquivo e mostra, para cada linha:

- **válida:** número da linha, nível, módulo e quantidade de campos;
- **inválida:** número da linha e motivo (identificador de SPEC §3.2);
- **em branco:** número da linha.

O formato dessa listagem é **livre**: é uma prévia interna, substituída pelo
relatório na Entrega 2. Um exemplo, só para ilustrar:

```
1 OK      INFO  auth  (3 campos)
2 OK      ERROR db    (3 campos)
...
8 INVALID too_few_tokens
...
16 BLANK
```

Código de saída: sempre 0 nesta entrega.

## Fora do escopo

Contagens e relatório, opções de linha de comando, avisos no stderr, códigos
1/2/3, JSON, leitura via stdin.

## Regras de negócio que entram

- **R1–R6 e R10** (SPEC §3.2).
- Todos os motivos de rejeição de §3.2, exceto `invalid_utf8`, que entra na Entrega 3.
- Prioridade quando há mais de um problema: vale o primeiro, da esquerda para a direita.
- R7 parcial: uma linha terminada em CRLF já não pode deixar resíduo do CR em
  nenhum campo (U18). O suporte completo a R7, R8 e R9 é da Entrega 3.

---

## Critérios de aceite

**Dado** `tests/fixtures/sample.log`, **quando** rodo `logstats tests/fixtures/sample.log`, **então**:

- [ ] 16 linhas aparecem como válidas
- [ ] 3 aparecem como inválidas: linha 8 `too_few_tokens`, linha 11 `bad_level`, linha 15 `bad_duration`
- [ ] a linha 16 aparece como em branco
- [ ] a linha 2 aparece como ERROR, módulo `db`, 3 campos
- [ ] a linha 13 aparece como WARN, módulo `auth`, 2 campos (o valor entre aspas com espaço conta como um campo só)
- [ ] o código de saída é 0

## Casos de teste (SPEC §7.1)

| Grupo | IDs | Nível |
|---|---|---|
| Validação de linha | **U01–U20, U55** | unitário |
| Níveis | **U21–U23** | unitário |

Lembrete do QA: nos casos de rejeição, o critério é o **motivo** (identificador
de §3.2), não o texto de uma mensagem.

---

## Definition of Done

- [ ] Critérios de aceite atendidos
- [ ] U01–U23 e U55 automatizados e verdes
- [ ] Q01 e Q02 (formatação e linter) atendidos — valem desde a primeira entrega
- [ ] Nenhuma dependência de terceiros (RNF1)
- [ ] Nenhum dos casos acima encerra o programa de forma abrupta (RNF4)

## Pacote de entrega para o QA

1. Saída completa da suíte de testes
2. Saída da verificação de formatação e do linter
3. Saída de `logstats tests/fixtures/sample.log`
4. Limitações conhecidas, se houver
