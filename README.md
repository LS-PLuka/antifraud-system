# antifraud-system

[![Java](https://img.shields.io/badge/Java-21-orange?style=flat-square)](https://www.java.com)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-6DB33F?style=flat-square)](https://spring.io/projects/spring-boot)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3.13-FF6600?style=flat-square)](https://www.rabbitmq.com)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?style=flat-square)](https://www.postgresql.org)
[![MongoDB](https://img.shields.io/badge/MongoDB-7-47A248?style=flat-square)](https://www.mongodb.com)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square)](https://www.docker.com)

Repositório central do **Motor de Análise de Risco de Transações** — sistema antifraude baseado em microsserviços que analisa transações financeiras em tempo real, aplicando um conjunto de regras de risco para classificá-las como aprovadas, sinalizadas ou bloqueadas.

> Este repositório centraliza o Docker Compose, a documentação de arquitetura e os links para todos os microsserviços. Cada serviço tem seu próprio repositório.

<p align="center">
  <img src="docs/banner.png" alt="antifraud-system" width="100%">
</p>

---

## Índice

- [O problema que este sistema resolve](#o-problema-que-este-sistema-resolve)
- [Arquitetura](#arquitetura)
- [Microsserviços](#microsserviços)
- [Classificação de risco](#classificação-de-risco)
- [Como rodar o projeto completo](#como-rodar-o-projeto-completo)
- [Configuração](#configuração)
- [Stack](#stack)
- [Status do projeto](#status-do-projeto)
- [Decisões técnicas](#decisões-técnicas)

---

## O problema que este sistema resolve

Toda fintech precisa decidir, em milissegundos, se uma transação é legítima ou suspeita. Fazer essa análise de forma síncrona — bloqueando o cliente até ter uma resposta — é inviável em escala.

A solução: a transação é recebida e registrada imediatamente. Em paralelo, um motor de risco analisa os dados de forma assíncrona, aplica regras de negócio e classifica o risco. O histórico de cada decisão fica registrado para auditoria.

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

O motor aplica até 6 regras por transação. Cada regra adiciona pontos ao score de risco:

| Pontuação | Classificação | Descrição |
|---|---|---|
| 0 – 39 | ✅ APROVADA | Risco baixo |
| 40 – 69 | ⚠️ SINALIZADA | Risco médio — requer revisão |
| 70+ | 🚫 BLOQUEADA | Risco alto — fraude provável |

---

## Como rodar o projeto completo

> ⚠️ Este repositório está em construção. O Docker Compose completo será atualizado conforme os microsserviços forem finalizados. O `servico-transacao` já está disponível de forma independente — veja o [repositório do serviço](https://github.com/LS-PLuka/servico-transacao) para rodá-lo isoladamente agora.

Requisitos: Docker e Docker Compose. Os repositórios dos microsserviços devem estar clonados lado a lado:

```
projetos/
├── antifraud-system/
├── servico-transacao/
├── motor-risco/
└── servico-auditoria/
```

```bash
git clone https://github.com/LS-PLuka/antifraud-system
cd antifraud-system
cp .env.example .env   # preencha JWT_SECRET antes de subir
make up
```

Após subir, os serviços estarão disponíveis em:

| Recurso | URL |
|---|---|
| servico-transacao API | http://localhost:8080 |
| servico-transacao Swagger | http://localhost:8080/swagger-ui.html |
| servico-auditoria API | http://localhost:8082 |
| servico-auditoria Swagger | http://localhost:8082/swagger-ui.html |
| RabbitMQ Management | http://localhost:15672 (`guest` / `guest`) |

---

## Configuração

Todas as variáveis têm default para desenvolvimento local. Em qualquer ambiente compartilhado, `JWT_SECRET` e `ADMIN_SENHA` **precisam** ser sobrescritas — os defaults estão neste repositório público.

Copie `.env.example` para `.env` e preencha:

| Variável | Default | Descrição |
|---|---|---|
| `DB_USERNAME` | `postgres` | Usuário do PostgreSQL |
| `DB_PASSWORD` | `postgres` | Senha do PostgreSQL |
| `RABBITMQ_USERNAME` | `guest` | Usuário do RabbitMQ |
| `RABBITMQ_PASSWORD` | `guest` | Senha do RabbitMQ |
| `JWT_SECRET` | *(obrigatório)* | Segredo HMAC — mínimo 256 bits |
| `JWT_EXPIRACAO` | `86400000` | Validade do token em milissegundos |
| `ADMIN_EMAIL` | `admin@antifraude.com` | E-mail do admin inicial |
| `ADMIN_SENHA` | `admin123456` | Senha do admin inicial |
| `ADMIN_NOME` | `Administrador` | Nome do admin inicial |

---

## Stack

| Tecnologia | Versão | Papel |
|---|---|---|
| Java | 21 | LTS |
| Spring Boot | 3.x | Framework base |
| Spring Security | 6.x | Autenticação e autorização |
| Spring Data JPA | — | Persistência relacional |
| Spring Data MongoDB | — | Persistência de documentos |
| Spring AMQP | — | Integração RabbitMQ |
| PostgreSQL | 16 | Banco do servico-transacao |
| MongoDB | 7 | Banco do servico-auditoria |
| RabbitMQ | 3.13 | Broker de mensagens |
| Docker + Compose | — | Containerização e orquestração |
| GitHub Actions | — | CI por microsserviço |
| JUnit 5 + Mockito | — | Testes unitários |
| Testcontainers | — | Testes de integração |
| Swagger / OpenAPI | — | Documentação interativa |

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

Desenvolvido por [Pedro Luka](https://github.com/LS-PLuka) · [LinkedIn](https://linkedin.com/in/pedroluka-dev)
