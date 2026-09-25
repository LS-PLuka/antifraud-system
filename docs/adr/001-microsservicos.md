# ADR-001 — Arquitetura de Microsserviços

## Contexto

O sistema precisa receber e persistir transações, analisar seu risco de forma assíncrona e registrar o histórico das decisões. Essas responsabilidades possuem modelos de dados e formas de acesso diferentes.

## Decisão

O sistema é composto por três microsserviços independentes:

- `servico-transacao`: API REST de entrada, validação, persistência no PostgreSQL e publicação da transação;
- `motor-risco`: consumo da transação, aplicação das cinco regras de risco e publicação do resultado, sem banco e sem API externa;
- `servico-auditoria`: consumo do resultado, persistência no MongoDB e API REST somente leitura.

Cada serviço possui repositório e ciclo de build próprios. Este repositório central apenas os orquestra, usando seus repositórios GitHub como contextos remotos do Docker Compose.

## Consequências

- As responsabilidades e os dados de cada serviço permanecem isolados.
- Nenhum serviço acessa diretamente o banco de outro.
- A integração interna depende dos contratos de eventos e do RabbitMQ.
- O sistema completo é executado e coordenado pelo Docker Compose central.
