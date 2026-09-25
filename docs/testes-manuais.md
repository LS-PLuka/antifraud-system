# Testes manuais do antifraud-system

Este guia valida manualmente o fluxo completo do sistema pelo Swagger, desde a entrada da transação até a consulta do histórico de auditoria.

```text
Cliente
  -> servico-transacao
  -> PostgreSQL
  -> RabbitMQ: transacoes.analise
  -> motor-risco
  -> RabbitMQ: risco.resultados
  -> servico-auditoria
  -> MongoDB
  -> API REST de auditoria
```

## 1. Subir o ambiente

Na raiz do repositório:

```bash
docker compose up --build -d
docker compose ps
```

O resultado esperado é:

- `postgres`, `rabbitmq` e `mongodb` com estado `healthy`;
- `servico-transacao`, `motor-risco` e `servico-auditoria` com estado `Up`.

Para acompanhar o processamento:

```bash
docker compose logs -f servico-transacao motor-risco servico-auditoria
```

## 2. Abrir as interfaces

| Interface | Endereço |
|---|---|
| Swagger do servico-transacao | http://localhost:8080/swagger-ui.html |
| Swagger do servico-auditoria | http://localhost:8082/swagger-ui.html |
| RabbitMQ Management | http://localhost:15672 |

O RabbitMQ Management usa as credenciais do `.env`, por padrão `guest` / `guest`.

O `motor-risco` não possui Swagger nem API HTTP de negócio. Ele é validado por seus efeitos no fluxo assíncrono e pelos logs.

## 3. Registrar o usuário de teste

No Swagger do `servico-transacao`, execute:

```text
POST /auth/registro
```

```json
{
  "nome": "Usuario Teste Manual",
  "email": "manual@antifraude.com",
  "senha": "senha123"
}
```

A resposta deve ter status `201`. Copie o campo `id`; ele será o `contaId` das três transações.

```json
{
  "id": "UUID-DA-CONTA",
  "nome": "Usuario Teste Manual",
  "email": "manual@antifraude.com",
  "perfil": "USUARIO",
  "criadoEm": "2026-09-25T17:30:00"
}
```

Se o e-mail já estiver cadastrado, use outro endereço, como `manual2@antifraude.com`, ou remova os volumes para recomeçar conforme a seção [Reiniciar os testes](#9-reiniciar-os-testes).

## 4. Autenticar e autorizar o Swagger

Execute:

```text
POST /auth/login
```

```json
{
  "email": "manual@antifraude.com",
  "senha": "senha123"
}
```

A resposta deve ter status `200`:

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "tipo": "Bearer",
  "perfil": "USUARIO"
}
```

Copie o valor de `token`, clique em **Authorize** no topo do Swagger e informe o JWT. Em uma interface configurada como Bearer, normalmente deve ser colado somente o token, sem escrever `Bearer` antes dele.

## 5. Cenário APROVADA

No Swagger do `servico-transacao`, execute:

```text
POST /transacoes/efetuar
```

Substitua `UUID-DA-CONTA` pelo `id` obtido no registro:

```json
{
  "contaId": "UUID-DA-CONTA",
  "valor": 100.00,
  "categoria": "SUPERMERCADO",
  "codigoPais": "BRA",
  "dataHora": "2026-09-25T14:00:00"
}
```

A resposta deve ter status `201` e status de transação `PENDENTE`, pois a análise é assíncrona. Copie o campo `id` da resposta; ele é o `transacaoId`.

No Swagger do `servico-auditoria`, execute:

```text
GET /auditorias/transacao/{transacaoId}
```

Resultado esperado:

```json
{
  "id": "ID-DA-AUDITORIA",
  "transacaoId": "UUID-DA-TRANSACAO",
  "pontuacao": 0,
  "nivel": "APROVADA",
  "regrasDisparadas": [],
  "analisadoEm": "DATA-HORA-DA-ANALISE",
  "registradoEm": "DATA-HORA-DO-REGISTRO"
}
```

## 6. Cenário SINALIZADA

Execute novamente `POST /transacoes/efetuar`, usando a mesma conta recém-criada:

```json
{
  "contaId": "UUID-DA-CONTA",
  "valor": 1500.00,
  "categoria": "ELETRONICOS",
  "codigoPais": "USA",
  "dataHora": "2026-09-25T15:00:00"
}
```

Pontuação esperada:

- `CONTA_NOVA`: +35;
- `PAIS_ESTRANGEIRO`: +25;
- total: 60.

Copie o novo `id` e consulte `GET /auditorias/transacao/{transacaoId}`.

```json
{
  "id": "ID-DA-AUDITORIA",
  "transacaoId": "UUID-DA-TRANSACAO",
  "pontuacao": 60,
  "nivel": "SINALIZADA",
  "regrasDisparadas": [
    "CONTA_NOVA",
    "PAIS_ESTRANGEIRO"
  ],
  "analisadoEm": "DATA-HORA-DA-ANALISE",
  "registradoEm": "DATA-HORA-DO-REGISTRO"
}
```

## 7. Cenário BLOQUEADA

Execute novamente `POST /transacoes/efetuar`:

```json
{
  "contaId": "UUID-DA-CONTA",
  "valor": 6000.00,
  "categoria": "JOALHERIA",
  "codigoPais": "BRA",
  "dataHora": "2026-09-25T16:00:00"
}
```

Pontuação esperada:

- `VALOR_ALTO`: +30;
- `CONTA_NOVA`: +35;
- `VALOR_MUITO_ALTO_CONTA_NOVA`: +45;
- total: 110.

Copie o novo `id` e consulte `GET /auditorias/transacao/{transacaoId}`.

```json
{
  "id": "ID-DA-AUDITORIA",
  "transacaoId": "UUID-DA-TRANSACAO",
  "pontuacao": 110,
  "nivel": "BLOQUEADA",
  "regrasDisparadas": [
    "VALOR_ALTO",
    "CONTA_NOVA",
    "VALOR_MUITO_ALTO_CONTA_NOVA"
  ],
  "analisadoEm": "DATA-HORA-DA-ANALISE",
  "registradoEm": "DATA-HORA-DO-REGISTRO"
}
```

## 8. Conferência final

No Swagger do `servico-auditoria`, execute:

```text
GET /auditorias?pagina=0
```

A página deve conter as três auditorias:

| Cenário | Valor | País | Pontuação | Nível |
|---|---:|---|---:|---|
| Aprovada | R$ 100 | BRA | 0 | `APROVADA` |
| Sinalizada | R$ 1.500 | USA | 60 | `SINALIZADA` |
| Bloqueada | R$ 6.000 | BRA | 110 | `BLOQUEADA` |

No RabbitMQ Management, abra **Queues and Streams**. As filas `transacoes.analise` e `risco.resultados` devem existir. Depois do processamento, ambas normalmente ficam com zero mensagens pendentes e um consumidor conectado.

Se a consulta da auditoria retornar `404` logo após criar a transação, aguarde um ou dois segundos e repita a chamada: o fluxo é assíncrono.

## 9. Reiniciar os testes

Para encerrar preservando PostgreSQL e MongoDB:

```bash
docker compose down
```

Para remover também os volumes e recomeçar sem os dados anteriores:

```bash
make clean
```

`make clean` apaga os dados locais persistidos nos volumes `postgres_data` e `mongodb_data`.
