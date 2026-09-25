# ADR-005 — Padrão Cadeia de Responsabilidade no motor-risco

## Contexto

Uma transação pode satisfazer simultaneamente várias condições de risco. Cada condição precisa ser avaliada isoladamente, e seus pontos devem ser acumulados antes da classificação final.

## Decisão

O `motor-risco` implementa as cinco regras com Chain of Responsibility, em ordem explícita:

1. `RegraValorAlto`;
2. `RegraContaNova`;
3. `RegraHorarioSuspeito`;
4. `RegraValorMuitoAltoContaNova`;
5. `RegraPaisEstrangeiro`.

Cada regra avalia sua condição, registra nome e pontos quando disparada e sempre encaminha o mesmo contexto para a próxima. Ao fim da cadeia, o score acumulado define `APROVADA` (0–39), `SINALIZADA` (40–69) ou `BLOQUEADA` (70 ou mais).

## Consequências

- As regras são cumulativas e testáveis de forma isolada.
- A ordem da cadeia é determinística.
- A classificação acontece uma vez, depois que todas as cinco regras foram avaliadas.
- O motor publica um único resultado com `pontuacao`, `nivel`, `regrasDisparadas` e `analisadoEm`.
