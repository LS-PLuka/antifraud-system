# AGENTS.md

## Escopo

- Este repositório centraliza orquestração e documentação; não contém código de negócio.
- Não altere os microsserviços a partir daqui. Seus repositórios remotos são `servico-transacao`, `motor-risco` e `servico-auditoria` na organização `LS-PLuka`.
- O Docker Compose usa esses repositórios GitHub diretamente como contextos de build.

## Arquitetura fechada

- Componentes: PostgreSQL 16, RabbitMQ 3.13, MongoDB 7 e os três microsserviços.
- `servico-transacao` (`8080`) usa PostgreSQL e publica em `transacoes.exchange` com `transacoes.risco`, fila `transacoes.analise`.
- `motor-risco` não possui banco nem API externa; consome transações, aplica cinco regras cumulativas e publica em `risco.exchange` com `risco.resultado`, fila `risco.resultados`.
- `servico-auditoria` (`8082`) consome resultados, persiste na coleção MongoDB `auditorias` e expõe API REST somente leitura.
- Bancos não são compartilhados e a comunicação interna ocorre exclusivamente via RabbitMQ.

## Contratos e manutenção

- Evento de transação: `transacaoId`, `contaId`, `valor`, `categoria`, `codigoPais`, `dataHora`, `contaCriadaEm`.
- Resultado de risco: `transacaoId`, `pontuacao`, `nivel`, `regrasDisparadas`, `analisadoEm`.
- Não invente componentes, topologias, regras, endpoints ou requisitos novos.
- Confirme mudanças de contrato na implementação dos microsserviços e mantenha Compose, README e ADRs coerentes.
- Operações Git são responsabilidade do mantenedor.
