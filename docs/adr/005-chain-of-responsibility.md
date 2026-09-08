# ADR-005 — Padrão Cadeia de Responsabilidade no motor-risco

## Contexto
O `motor-risco` precisa aplicar um conjunto de regras de risco sobre cada transação. O número de regras vai crescer com o tempo. É necessário que cada regra possa ser testada isoladamente, que novas regras possam ser adicionadas sem modificar as existentes, e que o resultado acumule as contribuições de todas as regras disparadas.

## Decisão
As regras de risco são implementadas usando o padrão de projeto Cadeia de Responsabilidade (Chain of Responsibility). Cada regra é uma classe independente que avalia uma condição, adiciona pontos ao contexto se necessário, e passa o contexto para a próxima regra da cadeia.

## Consequências

**Positivas:**
- Cada regra é completamente independente e testável de forma isolada
- Adicionar uma nova regra não requer modificar nenhuma regra existente — implementa o Princípio Aberto/Fechado do SOLID
- A lógica de cada regra fica encapsulada em sua própria classe
- A ordem de avaliação é explícita e controlada na montagem da cadeia

**Negativas:**
- Ligeiramente mais complexo que um bloco de if/else para quem não conhece o padrão
- A cadeia precisa ser montada explicitamente no serviço principal

## Alternativas consideradas
**Sequência de if/else:** descartado porque misturaria todas as regras em um único método, dificultando testes isolados e violando o Princípio de Responsabilidade Única do SOLID.

**Strategy Pattern:** considerado, mas a Cadeia de Responsabilidade é mais adequada porque permite que múltiplas regras sejam avaliadas em sequência e acumulem resultado, enquanto o Strategy seleciona apenas uma estratégia por vez.