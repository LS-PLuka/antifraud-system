# antifraud-system

[![Java](https://img.shields.io/badge/Java-21-orange?style=flat-square)](https://www.java.com)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.5.x-6DB33F?style=flat-square)](https://spring.io/projects/spring-boot)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3.13-FF6600?style=flat-square)](https://www.rabbitmq.com)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?style=flat-square)](https://www.postgresql.org)
[![MongoDB](https://img.shields.io/badge/MongoDB-7-47A248?style=flat-square)](https://www.mongodb.com)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square)](https://www.docker.com)

Sistema antifraude distribuído composto por três microsserviços. As transações entram por uma API REST, são processadas de forma assíncrona por RabbitMQ, persistidas no PostgreSQL e analisadas por um motor de risco; o resultado é registrado no MongoDB e disponibilizado por uma API REST de consulta.

> Este repositório centraliza a orquestração com Docker Compose e a documentação da arquitetura. O código de negócio permanece nos repositórios de cada microsserviço.

<p align="center">
  <img src="docs/banner.png" alt="antifraud-system" width="100%">
</p>

---

## Índice

- [Arquitetura](#arquitetura)
- [Microsserviços](#microsserviços)
- [Regras e classificação de risco](#regras-e-classificação-de-risco)
- [Contratos RabbitMQ](#contratos-rabbitmq)
- [Como executar](#como-executar)
- [Recursos disponíveis](#recursos-disponíveis)
- [Validação E2E](#validação-e2e)
- [Configuração](#configuração)
- [Stack](#stack)
- [Status do projeto](#status-do-projeto)
- [Decisões de arquitetura](#decisões-de-arquitetura)

## Arquitetura

```text
[Cliente]
    |
    | REST
    v
servico-transacao ----> PostgreSQL
    |
    | transacoes.exchange / transacoes.risco
    v
transacoes.analise
    |
    v
motor-risco
    |
    | risco.exchange / risco.resultado
    v
risco.resultados
    |
    v
servico-auditoria ----> MongoDB
    |
    +----> API REST de consulta
```

O `servico-transacao` é a entrada do sistema: valida e persiste a transação, depois publica seu evento. O `motor-risco` consome esse evento, aplica em cadeia as cinco regras, soma o score e publica a classificação. O `servico-auditoria` consome o resultado, registra o histórico na coleção `auditorias` e o expõe em uma API somente leitura.

A comunicação interna `servico-transacao -> motor-risco -> servico-auditoria` ocorre exclusivamente pelo RabbitMQ. Nenhum microsserviço acessa o banco de outro: PostgreSQL pertence ao `servico-transacao`, MongoDB pertence ao `servico-auditoria` e o `motor-risco` não possui banco.

## Microsserviços

| Repositório | Responsabilidade | Banco | Acesso externo |
|---|---|---|---|
| [servico-transacao](https://github.com/LS-PLuka/servico-transacao) | API REST, persistência e publicação RabbitMQ | PostgreSQL | porta `8080` |
| [motor-risco](https://github.com/LS-PLuka/motor-risco) | Consumo, análise pela cadeia de regras e publicação do resultado | Sem banco | sem API externa |
| [servico-auditoria](https://github.com/LS-PLuka/servico-auditoria) | Consumo, persistência do histórico e API REST de consulta | MongoDB | porta `8082` |

## Regras e classificação de risco

As cinco regras são executadas em uma Chain of Responsibility. Elas são cumulativas: toda regra aplicável adiciona seus pontos e seu identificador ao resultado.

| Regra | Condição | Pontos |
|---|---|---:|
| `RegraValorAlto` | valor maior que R$ 5.000 | +30 |
| `RegraContaNova` | conta com menos de 30 dias e valor maior que R$ 1.000 | +35 |
| `RegraHorarioSuspeito` | horário entre 00:00 (inclusive) e 06:00 (exclusive) e valor maior que R$ 500 | +20 |
| `RegraValorMuitoAltoContaNova` | conta com menos de 30 dias e valor maior que R$ 5.000 | +45 |
| `RegraPaisEstrangeiro` | `codigoPais` diferente de `BRA` | +25 |

| Pontuação | Classificação |
|---:|---|
| 0–39 | `APROVADA` |
| 40–69 | `SINALIZADA` |
| 70 ou mais | `BLOQUEADA` |

## Contratos RabbitMQ

Cada microsserviço declara a topologia que utiliza por Spring AMQP; o Docker Compose não cria exchanges nem filas.

| Fluxo | Exchange | Routing key | Fila |
|---|---|---|---|
| Transação -> motor | `transacoes.exchange` | `transacoes.risco` | `transacoes.analise` |
| Motor -> auditoria | `risco.exchange` | `risco.resultado` | `risco.resultados` |

### Transação -> motor

O evento contém `transacaoId`, `contaId`, `valor`, `categoria`, `codigoPais`, `dataHora` e `contaCriadaEm`. Com esses dados, o motor executa a análise sem consultar o PostgreSQL do serviço de transação.

### Motor -> auditoria

O resultado contém `transacaoId`, `pontuacao`, `nivel`, `regrasDisparadas` e `analisadoEm`. A auditoria persiste esses campos e acrescenta `registradoEm`.

## Como executar

Requisitos: Git, Docker com suporte a Docker Compose e acesso ao GitHub durante o build.

```bash
git clone https://github.com/LS-PLuka/antifraud-system
cd antifraud-system
cp .env.example .env
```

Preencha `JWT_SECRET` no arquivo `.env` com um segredo de no mínimo 256 bits. Em seguida:

```bash
docker compose up --build
```

Com Make disponível, o comando equivalente é:

```bash
make up
```

O Docker usa os três repositórios GitHub diretamente como contextos remotos de build. Somente o `antifraud-system` precisa ser clonado pelo usuário.

Para encerrar, acompanhar logs ou remover também volumes e containers órfãos:

```bash
make down
make logs
make clean
```

## Recursos disponíveis

| Recurso | Endereço |
|---|---|
| servico-transacao | http://localhost:8080 |
| Swagger do servico-transacao | http://localhost:8080/swagger-ui.html |
| servico-auditoria | http://localhost:8082 |
| Swagger do servico-auditoria | http://localhost:8082/swagger-ui.html |
| RabbitMQ Management | http://localhost:15672 |
| PostgreSQL | `localhost:5432` |
| MongoDB | `localhost:27017` |

O painel do RabbitMQ usa as credenciais definidas por `RABBITMQ_USERNAME` e `RABBITMQ_PASSWORD` (`guest` / `guest` no exemplo). O `motor-risco` não publica porta HTTP no host.

## Validação E2E

1. Suba o Compose e aguarde os containers ficarem prontos.
2. Registre um usuário comum em `POST /auth/registro` no `servico-transacao`.
3. Autentique-o em `POST /auth/login` e use o JWT retornado.
4. Envie a transação em `POST /transacoes/efetuar`, usando como `contaId` o `id` retornado no registro.
5. Copie o campo `id` da resposta da transação; ele é o `transacaoId` do fluxo assíncrono.
6. Aguarde o processamento pelo `motor-risco` e pelo `servico-auditoria`.
7. Consulte `GET /auditorias/transacao/{transacaoId}` em `http://localhost:8082`.
8. Confira `pontuacao`, `nivel`, `regrasDisparadas`, `analisadoEm` e `registradoEm`.

Durante o desenvolvimento, o RabbitMQ Management permite inspecionar `transacoes.analise` e `risco.resultados` enquanto as mensagens atravessam o sistema.

## Configuração

O `.env` central fornece as credenciais compartilhadas pelo Compose. Os hostnames internos, portas dos containers e URI do MongoDB pertencem à topologia Docker e ficam definidos no próprio Compose.

| Variável | Valor no exemplo | Utilização |
|---|---|---|
| `DB_USERNAME` | `postgres` | PostgreSQL e servico-transacao |
| `DB_PASSWORD` | `postgres` | PostgreSQL e servico-transacao |
| `RABBITMQ_USERNAME` | `guest` | RabbitMQ e três microsserviços |
| `RABBITMQ_PASSWORD` | `guest` | RabbitMQ e três microsserviços |
| `JWT_SECRET` | sem valor | Assinatura JWT no servico-transacao |
| `JWT_EXPIRACAO` | `86400000` | Validade do JWT em milissegundos |
| `ADMIN_EMAIL` | `admin@antifraude.com` | Admin inicial do servico-transacao |
| `ADMIN_SENHA` | `admin123456` | Senha do admin inicial |
| `ADMIN_NOME` | `Administrador` | Nome do admin inicial |

## Stack

| Tecnologia | Versão / papel |
|---|---|
| Java | 21 |
| Spring Boot | 3.5.x |
| Spring Web | APIs REST |
| Spring Security | autenticação e autorização do servico-transacao |
| Spring Data JPA | persistência no PostgreSQL |
| Spring Data MongoDB | persistência de auditoria |
| Spring AMQP | integração RabbitMQ |
| PostgreSQL | 16 |
| MongoDB | 7 |
| RabbitMQ | 3.13 |
| Docker / Docker Compose | build e orquestração |
| GitHub Actions | integração contínua dos microsserviços |
| JUnit 5, Mockito e Testcontainers | testes unitários e de integração |
| Swagger / OpenAPI | documentação das APIs REST |

## Status do projeto

| Componente | Status |
|---|---|
| servico-transacao | Concluído |
| motor-risco | Concluído |
| servico-auditoria | Concluído |
| Docker Compose | Implementado |
| Integração central | Pronta para validação E2E |

## Decisões de arquitetura

- [ADR-001 — Arquitetura de microsserviços](docs/adr/001-microsservicos.md)
- [ADR-002 — RabbitMQ como sistema de mensageria](docs/adr/002-rabbitmq.md)
- [ADR-003 — PostgreSQL no servico-transacao](docs/adr/003-postgresql.md)
- [ADR-004 — MongoDB no servico-auditoria](docs/adr/004-mongodb.md)
- [ADR-005 — Cadeia de Responsabilidade no motor-risco](docs/adr/005-chain-of-responsibility.md)

---

Desenvolvido por [Pedro Luka](https://github.com/LS-PLuka) · [LinkedIn](https://linkedin.com/in/pedroluka-dev)
