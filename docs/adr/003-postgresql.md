# ADR-003 — PostgreSQL no servico-transacao

## Contexto
O `servico-transacao` precisa persistir transações financeiras. Esses dados são estruturados, têm campos bem definidos e exigem consistência garantida — não é aceitável ter uma transação salva pela metade ou com dados corrompidos.

## Decisão
O `servico-transacao` usa PostgreSQL como banco de dados relacional.

## Consequências

**Positivas:**
- Consistência transacional garantida (ACID)
- Schema bem definido evita dados inconsistentes
- Suporte nativo a transações com Spring Data JPA

**Negativas:**
- Menos flexível para mudanças de schema em comparação a bancos NoSQL
- Necessidade de migrations em alterações de estrutura (atualmente gerenciado por `ddl-auto=update`)

## Alternativas consideradas
**MongoDB:** descartado para este serviço porque os dados são estruturados e relacionais. Usar MongoDB aqui seria escolha por modismo, não por adequação técnica.

**MySQL:** considerado, mas PostgreSQL foi preferido por ser mais robusto, ter melhor suporte a tipos de dados avançados e ser o padrão mais adotado no ecossistema Spring Boot moderno.
