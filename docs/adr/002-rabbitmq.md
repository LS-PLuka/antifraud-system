# ADR-002 — RabbitMQ como sistema de mensageria

## Contexto
Os microsserviços precisam se comunicar. A análise de risco não precisa acontecer de forma síncrona — o cliente não precisa esperar o resultado para receber a confirmação de que a transação foi registrada. Comunicação via HTTP direto entre serviços cria acoplamento: se o `motor-risco` cair, o `servico-transacao` falharia junto.

## Decisão
Toda comunicação entre serviços é feita de forma assíncrona via RabbitMQ.

- O `servico-transacao` publica eventos na fila `transacoes.analise`
- O `motor-risco` publica resultados na fila `risco.resultados`

Nenhum serviço chama outro diretamente via HTTP.

## Consequências

**Positivas:**
- Serviços completamente desacoplados — nenhum conhece o endereço do outro
- Resiliência: se um serviço cair, as mensagens ficam na fila até ele voltar
- Escalabilidade: múltiplas instâncias do `motor-risco` podem consumir a mesma fila sem alterar nenhum outro serviço

**Negativas:**
- Maior dificuldade para rastrear o fluxo completo de uma transação
- Complexidade adicional na configuração do ambiente local

## Alternativas consideradas
**Chamada HTTP direta (REST):** descartado por criar acoplamento forte entre serviços e eliminar a resiliência a falhas.

**Apache Kafka:** considerado, mas descartado para a versão inicial. Kafka é mais adequado para volumes muito altos e persistência longa de eventos. RabbitMQ atende bem ao caso de uso atual e é mais simples de operar.