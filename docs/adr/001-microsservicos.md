# ADR-001 — Arquitetura de Microsserviços

## Contexto
O sistema precisa receber transações financeiras, analisar o risco de cada uma e registrar o histórico de decisões. Essas três responsabilidades têm naturezas diferentes: a entrada é síncrona (API REST), a análise é assíncrona e intensiva em regras, e o histórico é append-only. Colocar tudo em um único serviço misturaria responsabilidades distintas e dificultaria a evolução independente de cada parte.

## Decisão
O sistema foi dividido em três microsserviços independentes:

- `servico-transacao` — entrada e validação
- `motor-risco` — análise de risco
- `servico-auditoria` — histórico de decisões

Cada serviço tem seu próprio repositório, banco de dados e ciclo de deploy.

## Consequências

**Positivas:**
- Cada serviço pode evoluir, escalar e ser implantado de forma independente
- Responsabilidades claras facilitam manutenção e testes
- Falha em um serviço não derruba os outros

**Negativas:**
- Maior complexidade operacional em relação a um monolito
- Necessidade de orquestração via Docker Compose

## Alternativas consideradas
**Monolito:** descartado porque misturaria responsabilidades muito distintas e dificultaria a demonstração de conceitos como mensageria e bancos de dados especializados.