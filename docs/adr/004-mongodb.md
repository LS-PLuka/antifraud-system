# ADR-004 — MongoDB no servico-auditoria

## Contexto

Cada resultado de risco é um registro autocontido com score, classificação, lista variável de regras disparadas e timestamps. A auditoria recebe esses resultados de forma assíncrona e os disponibiliza somente para leitura.

## Decisão

O `servico-auditoria` usa MongoDB 7 como seu banco exclusivo. Os documentos são persistidos na coleção `auditorias`, no banco `antifraude`, sem autenticação MongoDB nesta versão. Cada documento contém o resultado recebido e acrescenta `registradoEm`.

## Consequências

- A lista de regras disparadas é armazenada no mesmo documento da decisão.
- O serviço consulta seu próprio histórico sem depender do PostgreSQL.
- A escrita de auditorias ocorre apenas pelo consumo de `risco.resultados`; a API REST é somente leitura.
- O Compose mantém os dados no volume `mongodb_data`.
