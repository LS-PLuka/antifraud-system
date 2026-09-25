# ADR-002 — RabbitMQ como sistema de mensageria

## Contexto

A entrada da transação não precisa aguardar a análise de risco nem o registro da auditoria. Chamadas HTTP internas acoplariam a disponibilidade dos três serviços.

## Decisão

A comunicação interna é exclusivamente assíncrona via RabbitMQ 3.13, com exchanges diretas e filas duráveis declaradas pelos próprios microsserviços por Spring AMQP:

- `servico-transacao` publica em `transacoes.exchange`, routing key `transacoes.risco`, para `transacoes.analise`;
- `motor-risco` consome `transacoes.analise` e publica em `risco.exchange`, routing key `risco.resultado`, para `risco.resultados`;
- `servico-auditoria` consome `risco.resultados`.

O Docker Compose fornece somente o broker; não declara manualmente a topologia.

## Consequências

- O cliente recebe a confirmação da transação sem esperar toda a cadeia.
- Os microsserviços não conhecem endereços HTTP uns dos outros.
- Os contratos JSON e os nomes da topologia são interfaces de integração que precisam permanecer compatíveis.
- O resultado de auditoria fica disponível após o processamento assíncrono.
