# ADR-004 — MongoDB no servico-auditoria

## Contexto
O `servico-auditoria` precisa registrar o histórico de decisões do motor de risco. Cada registro contém uma lista de regras disparadas que varia de transação para transação — uma pode disparar 2 regras, outra pode disparar 5. Além disso, registros de auditoria são imutáveis: nunca são atualizados, apenas criados.

## Decisão
O `servico-auditoria` usa MongoDB como banco de dados orientado a documentos.

## Consequências

**Positivas:**
- Estrutura flexível de documento acomoda listas de tamanho variável sem tabelas auxiliares
- Padrão append-only é natural em bancos de documentos
- Sem necessidade de joins — cada documento é autocontido

**Negativas:**
- Sem garantias ACID por padrão (aceitável aqui, pois auditoria não exige consistência transacional entre documentos)

## Alternativas consideradas
**PostgreSQL:** possível, mas exigiria uma tabela separada para as regras disparadas com chave estrangeira, aumentando a complexidade das queries sem benefício real para esse caso de uso.