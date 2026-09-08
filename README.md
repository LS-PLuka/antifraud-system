# antifraud-system

Sistema antifraude distribuído com análise de risco assíncrona.

<p align="center">
  <img src="docs/banner.png" alt="antifraud-system" width="100%">
</p>

---

## O problema que este sistema resolve

Toda fintech precisa decidir, em milissegundos, se uma transação é legítima ou suspeita. Fazer essa análise de forma síncrona — bloqueando o cliente até ter uma resposta — é inviável em escala.

A solução: a transação é recebida e registrada imediatamente. Em paralelo, um motor de risco analisa os dados de forma assíncrona, aplica regras de negócio e classifica o risco. O histórico de cada decisão fica registrado para auditoria.

Essa é a arquitetura usada por empresas como Nubank, PicPay e produtos como Konduto e ClearSale.

---

## Arquitetura

```
[Cliente / Sistema Externo]
          │
          ▼ REST API
┌─────────────────────┐
│  servico-transacao  │  Recebe, valida e persiste a transação
└─────────────────────┘
          │
          ▼ RabbitMQ — fila: transacoes.analise
┌─────────────────────┐
│     motor-risco     │  Aplica cadeia de regras e calcula score de risco
└─────────────────────┘
          │
          ▼ RabbitMQ — fila: risco.resultados
┌─────────────────────┐
│  servico-auditoria  │  Registra o histórico completo da decisão
└─────────────────────┘
```

Nenhum serviço se comunica diretamente com outro via HTTP. Toda comunicação é assíncrona via RabbitMQ. Se qualquer serviço cair, as mensagens ficam na fila e são processadas quando ele voltar.

---

## Microsserviços

| Repositório | Responsabilidade | Banco | Porta |
|---|---|---|---|
| [servico-transacao](https://github.com/LS-PLuka/servico-transacao) | Recebe transações via API REST, valida, persiste e publica na fila | PostgreSQL | 8080 |
| [motor-risco](https://github.com/LS-PLuka/motor-risco) | Consome da fila, aplica cadeia de regras e classifica a transação | — | 8081 |
| [servico-auditoria](https://github.com/LS-PLuka/servico-auditoria) | Consome resultados e persiste o histórico de decisões | MongoDB | 8082 |

---

## Classificação de risco

O motor aplica até 6 regras por transação. Cada regra adiciona pontos ao score:

| Pontuação | Classificação | Descrição |
|---|---|---|
| 0 – 39 | ✅ APROVADA | Risco baixo |
| 40 – 69 | ⚠️ SINALIZADA | Risco médio — requer revisão |
| 70+ | 🚫 BLOQUEADA | Risco alto — fraude provável |

---

## Como rodar o projeto completo

> ⚠️ Este repositório está em construção. O Docker Compose completo será atualizado conforme os microsserviços forem finalizados. O `servico-transacao` já está disponível de forma independente — veja o [repositório do serviço](https://github.com/LS-PLuka/servico-transacao) para rodá-lo isoladamente.

Com Docker instalado:

```bash
git clone https://github.com/LS-PLuka/antifraud-system
cd antifraud-system
cp .env.example .env   # preencha JWT_SECRET antes de subir
make up
```

Após subir, os serviços estarão disponíveis em:

| Serviço | URL |
|---|---|
| servico-transacao API | http://localhost:8080 |
| servico-transacao Swagger | http://localhost:8080/swagger-ui.html |
| servico-auditoria API | http://localhost:8082 |
| servico-auditoria Swagger | http://localhost:8082/swagger-ui.html |
| RabbitMQ Management | http://localhost:15672 |

---

## Stack

- **Java 21** + **Spring Boot 3**
- **Spring Security** + **JWT**
- **Spring Data JPA** — PostgreSQL
- **Spring Data MongoDB**
- **Spring AMQP** — RabbitMQ
- **Docker** + **Docker Compose**
- **GitHub Actions** — CI por microsserviço
- **JUnit** + **Mockito** + **Testcontainers**
- **Swagger / OpenAPI**

---

## Status do projeto

| Componente | Status |
|---|---|
| servico-transacao | ✅ Concluído |
| motor-risco | 🔄 Em desenvolvimento |
| servico-auditoria | ⏳ Aguardando |
| Docker Compose completo | ⏳ Aguardando |

---

## Decisões técnicas

As decisões de arquitetura estão documentadas em [`docs/adr/`](docs/adr/):

- [ADR-001 — Arquitetura de microsserviços](docs/adr/001-microsservicos.md)
- [ADR-002 — RabbitMQ como sistema de mensageria](docs/adr/002-rabbitmq.md)
- [ADR-003 — PostgreSQL no servico-transacao](docs/adr/003-postgresql.md)
- [ADR-004 — MongoDB no servico-auditoria](docs/adr/004-mongodb.md)
- [ADR-005 — Cadeia de Responsabilidade no motor-risco](docs/adr/005-chain-of-responsibility.md)

---

## Autor

**Pedro Luka**
[GitHub](https://github.com/LS-PLuka) · [LinkedIn](https://linkedin.com/in/pedroluka-dev)
