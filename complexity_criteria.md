# CensoBench — Critérios de Classificação de Complexidade

Este documento descreve o sistema de pontuação utilizado para classificar automaticamente as queries SQL em níveis de complexidade. O critério foi projetado para ser **reproduzível**, **auditável** e **escalável** para novos exemplos.

---

## Motivação

A classificação de complexidade é central para análise de erros de modelos Text-to-SQL: permite identificar se um modelo falha sistematicamente em construções específicas (ex: CTEs, window functions) ou apenas em queries de alta profundidade de aninhamento. O sistema de pontuação composta garante que a categorização seja determinística e justificável em publicações científicas.

---

## Sistema de Pontuação

Cada query recebe uma **pontuação total** correspondente à soma de pontos atribuídos por critério. A pontuação de cada critério é independente.

| Critério | Coluna no dataset | Regra de cálculo | Pontos máximos |
|---|---|---|---|
| Número de JOINs | `pts_joins` | `min(num_joins, 3)` — cada JOIN = +1, cap em 3 | 3 |
| FULL / CROSS JOIN | `pts_full_cross_join` | +2 se FULL OUTER JOIN ou CROSS JOIN presente | 2 |
| CTE (*Common Table Expression*) | `pts_cte` | +2 pela presença + +1 por cada CTE extra além da primeira | variável |
| Window Function | `pts_window_fn` | +3 se cláusula `OVER()` presente | 3 |
| Subqueries | `pts_subqueries` | `num_subqueries + profundidade_subquery` | variável |
| Agregação aninhada | `pts_agg_aninhada` | +2 se há agregação dentro de agregação | 2 |
| HAVING | `pts_having` | +1 se cláusula `HAVING` presente | 1 |
| NOT IN / EXISTS | `pts_not_in_exists` | +2 se `NOT IN` ou `EXISTS` / `NOT EXISTS` presente | 2 |
| CASE WHEN | `pts_case_when` | +1 se expressão `CASE WHEN` presente | 1 |
| Mais de 3 tabelas | `pts_tabelas_extra` | +1 se `num_tabelas > 3` | 1 |
| **SCORE TOTAL** | `score_total` | Soma de todos os critérios acima | — |

### Detalhamento dos critérios

**JOINs (`pts_joins`):** cada JOIN adicional exige que o modelo identifique corretamente a chave de ligação, ampliando o espaço de *schema linking*. O teto em 3 pontos evita inflação em queries com muitos JOINs simples.

**FULL / CROSS JOIN (`pts_full_cross_join`):** JOINs de produto cartesiano ou bidirecional são construções raras e semanticamente complexas. O FULL OUTER JOIN requer tratamento de NULLs nas duas direções; o CROSS JOIN exige compreensão de cardinalidade.

**CTE (`pts_cte`):** *Common Table Expressions* introduzem escopo intermediário que o modelo deve manter ao construir a query principal. Cada CTE extra cria uma nova sub-tarefa de resolução e aumenta o tamanho do output esperado.

**Window Functions (`pts_window_fn`):** funções como `ROW_NUMBER()`, `LAG()`, `AVG() OVER()` exigem compreensão de `PARTITION BY` e `ORDER BY` dentro de `OVER()`. 

**Subqueries (`pts_subqueries`):** a pontuação combina duas dimensões:
- `num_subqueries`: contagem de ocorrências de `(SELECT` na query
- `profundidade_subquery`: nível máximo de `SELECT` aninhado dentro de parênteses

> **Nota metodológica:** a profundidade é calculada contando apenas parênteses que abrem um `SELECT`, ignorando funções como `COUNT()`, `EXTRACT()` e `COALESCE()`. 

**Agregação aninhada (`pts_agg_aninhada`):** agregações dentro de agregações — ex: `SUM(CASE WHEN ... THEN COUNT(...))`.

**HAVING (`pts_having`):** a cláusula `HAVING` filtra após agregação e requer que o modelo distinga semanticamente `WHERE` de `HAVING`. Erros de confusão entre as duas cláusulas são recorrentes em avaliações de Text-to-SQL.

**NOT IN / EXISTS (`pts_not_in_exists`):** operadores de negação semântica exigem raciocínio sobre complementos de conjuntos. 

**CASE WHEN (`pts_case_when`):** expressões condicionais inline adicionam lógica de controle de fluxo à query, ampliando o espaço de tokens que o modelo precisa gerar corretamente.

**Mais de 3 tabelas (`pts_tabelas_extra`):** queries com mais de 3 tabelas ampliam significativamente o espaço de *schema linking* e aumentam a probabilidade de erro na seleção de atributos e aliases.

---

## Faixas de Classificação

| Nível | Score Total | Perfil típico |
|---|---|---|
| **Fácil** | 0 – 2 | `SELECT` com `GROUP BY`, sem CTE ou subquery aninhada; no máximo 1 JOIN direto |
| **Média** | 3 – 5 | 1 JOIN + subquery escalar de referência temporal (`MAX(ano)`); sem CTE |
| **Difícil** | 6 – 9 | CTE simples; `NOT IN`/`EXISTS`; `HAVING` com subquery; múltiplos JOINs |
| **Muito Difícil** | 10+ | CTE + Window Function + subqueries profundas + FULL/CROSS JOIN |

---

## Exemplos de Pontuação

### Exemplo 1 — Nível Fácil (score = 2)

```sql
SELECT COUNT(DISTINCT id_docente) AS total_mestres
FROM docente
WHERE sigla_uf = 'SC'
  AND mestrado = 1
  AND ano = (SELECT MAX(ano) FROM docente);
```

| Critério | Valor detectado | Pontos |
|---|---|---|
| JOINs | 0 | 0 |
| Subqueries | `num_subqueries=1`, `profundidade=1` | **+2** |
| Todos os outros | ausentes | 0 |
| **TOTAL** | | **2 → Fácil** |

---

### Exemplo 2 — Nível Difícil (score = 7)

```sql
WITH turmas_ano_anterior AS (
  SELECT DISTINCT id_turma, id_escola, id_municipio, quantidade_matriculas, sigla_uf, ano
  FROM turma
  WHERE ano = (EXTRACT(YEAR FROM CURRENT_DATE()) - 1)
),
turmas_ano_atual AS (
  SELECT DISTINCT id_turma
  FROM turma
  WHERE ano = (EXTRACT(YEAR FROM CURRENT_DATE()))
)
SELECT t.id_turma, t.quantidade_matriculas, t.id_municipio, t.sigla_uf
FROM turmas_ano_anterior t
LEFT JOIN turmas_ano_atual ta ON t.id_turma = ta.id_turma
WHERE ta.id_turma IS NULL;
```

| Critério | Valor detectado | Pontos |
|---|---|---|
| JOINs | 1 LEFT JOIN | **+1** |
| CTE | 2 CTEs → 2 base + 1 extra | **+3** |
| Subqueries | `num_subqueries=2`, `profundidade=1` | **+3** |
| Todos os outros | ausentes | 0 |
| **TOTAL** | | **7 → Difícil** |

> **Nota:** a query envolve **1 tabela física** do schema (`turma`). As CTEs `turmas_ano_anterior` e `turmas_ano_atual` são aliases temporários definidos no próprio `WITH`, não tabelas reais — portanto o critério "tabelas > 3" não se aplica.

---
