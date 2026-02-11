# Questão 3 — Estratégia de Validação do Scheduler

O objetivo desta etapa é garantir que o scheduler atenda aos requisitos funcionais e não funcionais definidos. O requisito principal é suportar 1.000 requisições por segundo com P99 de até 30ms, mantendo comportamento previsível e estável do workflow.

Como o endpoint apenas registra o agendamento em um banco chave/valor em memória, como o Redis, e a publicação no Kafka ocorre de forma assíncrona por meio de um worker, os testes devem validar separadamente o desempenho do endpoint e o comportamento do processamento agendado.

## Testes Unitários

Os testes unitários asseguram previsibilidade e consistência do comportamento do endpoint.

São avaliados os seguintes cenários:

- Requisição sem payload.
- Campos obrigatórios ausentes.
- Formato inválido de data ou hora.
- Agendamentos com valores inconsistentes.
- Tratamento adequado de exceções internas.
- Garantia de que erros não exponham informações sensíveis.

Esses testes garantem que o serviço se comporte corretamente antes de qualquer validação sob carga.

## Testes de Integração

Os testes de integração validam o fluxo completo do agendamento até a execução do evento, incluindo persistência em memória e funcionamento do mecanismo de polling.

Os principais cenários considerados são:

- Persistência correta do agendamento no Redis após a chamada ao endpoint.
- Garantia de que o evento não seja executado antes do horário configurado.
- Execução do evento após o vencimento.
- Remoção do agendamento somente após a publicação no Kafka.
- Processamento adequado quando múltiplos eventos vencem simultaneamente.
- Continuidade do processamento após reinício do worker.

Esses testes são executados com Redis real em ambiente isolado, garantindo que o mecanismo de polling opere de forma previsível e estável.

## Testes de Carga do Endpoint

O requisito central da questão é suportar 1.000 requisições por segundo com P99 de até 30ms. Essa meta deve ser validada por meio de testes de carga sustentados, executados em ambiente controlado e com métricas claramente definidas. Como a execução do evento ocorre de forma assíncrona, a latência considerada corresponde exclusivamente ao tempo de resposta do endpoint /v1/render/scheduler.

### Preparação do ambiente

Antes da execução dos testes de carga, o ambiente deve ser configurado de forma controlada e reproduzível.

A aplicação e o Redis devem executar isoladamente, sem concorrência com outros serviços que possam consumir CPU, memória ou rede. O pool de conexões com o Redis deve estar previamente configurado e estabilizado antes do início do teste. O gerador de carga deve ser executado em máquina separada da aplicação testada, evitando competição por recursos. Os recursos computacionais da aplicação, como CPU e memória, devem ser definidos previamente e mantidos constantes durante todo o teste, sem autoscaling ou alterações dinâmicas.

### Execução do Teste

O teste de carga deve ser conduzido de forma estruturada, permitindo observar o comportamento do sistema em diferentes estágios de utilização e evitando conclusões baseadas em medições pontuais.

A execução pode ser organizada nas seguintes fases:

#### 1. Fase de aquecimento

Inicialmente, aplica-se uma carga moderada por um curto período com o objetivo de estabilizar o ambiente. Essa etapa permite que conexões com o Redis sejam estabelecidas, que o runtime da aplicação atinja estado estável e que possíveis inicializações tardias sejam concluídas. As métricas coletadas nessa fase não são consideradas para validação final, servindo apenas para estabilização.

#### 2. Subida gradual da carga

Após o aquecimento, a carga é aumentada progressivamente até atingir 1.000 requisições por segundo. Essa progressão deve ser controlada, evitando picos abruptos. O objetivo é observar como a latência evolui conforme o volume cresce e identificar antecipadamente possíveis pontos de saturação.

#### 3. Fase de carga constante

Uma vez atingido o volume de 1.000 requisições por segundo, o sistema deve permanecer nesse nível por período suficiente para caracterizar estabilidade. Durante essa fase são coletadas as métricas que servirão como base para validação do requisito. Essa etapa deve ser longa o bastante para permitir a observação de comportamento consistente e não apenas respostas pontuais.

#### 4. Redução de carga

Opcionalmente, a carga pode ser reduzida ao final do teste para observar a recuperação do sistema e verificar se não houve degradação persistente de desempenho ou acúmulo de recursos.

As requisições utilizadas durante o teste devem representar o cenário real esperado em produção. O payload deve conter scheduler_datetime distribuído de forma realista e event_content com tamanho compatível com o uso previsto.

### Métricas Monitoradas

Durante todas as fases do teste, as seguintes métricas devem ser coletadas e analisadas de forma conjunta:

**Latência (P50, P90 e P99):** O P50 representa o comportamento médio do sistema, enquanto o P90 e o P99 evidenciam o comportamento sob carga elevada. O P99 é a principal métrica de validação do requisito.

**Throughput efetivo:** Indica quantas requisições por segundo foram efetivamente processadas pelo sistema.

**Taxa de erro:** Inclui respostas 4xx, 5xx e timeouts. A taxa de erro deve permanecer baixa e estável durante toda a fase de carga constante.

**Consumo de CPU e memória da aplicação:** Permite identificar saturação de recursos ou crescimento anormal de consumo ao longo do tempo.

**Tempo médio de resposta do Redis:** Ajuda a verificar se o acesso ao banco em memória está se tornando um gargalo sob carga.

**Número de conexões ativas:** Permite detectar exaustão do pool de conexões ou comportamento anômalo no gerenciamento de recursos.

A análise dessas métricas deve ser feita de forma correlacionada. A elevação do P99, por exemplo, deve ser analisada em conjunto com CPU, memória e latência do Redis.

### Critérios de Aprovação

O teste é considerado satisfatório quando todas as condições abaixo são atendidas simultaneamente:

- O sistema sustenta 1.000 requisições por segundo durante a fase de carga constante.
- O P99 permanece menor ou igual a 30ms durante o período analisado.
- A taxa de erro permanece baixa e não apresenta crescimento progressivo.
- O consumo de recursos permanece estável, sem tendência de degradação contínua.

Caso o P99 ultrapasse o limite estabelecido, a investigação deve seguir uma abordagem estruturada, analisando possíveis cenários como:

- Saturação de CPU da aplicação.
- Aumento da latência nas operações de Redis.
- Exaustão do pool de conexões.
- Limitação de recursos de rede.
- Crescimento não controlado de filas internas.

Essa análise permite identificar o componente responsável pela degradação e ajustar a configuração do sistema antes de nova execução do teste.

## Testes de Processamento sob Pico

Além da carga constante no endpoint, é necessário validar o comportamento do worker quando múltiplos eventos vencem simultaneamente.

Para isso, são agendados milhares de eventos para o mesmo horário, observando:

- Capacidade de processamento progressivo.
- Redução contínua do backlog.
- Ausência de travamentos ou crescimento indefinido de eventos pendentes.

Esse cenário garante que o mecanismo de polling opere de forma estável mesmo sob condições de pico.

A estratégia de testes apresentada garante a validação completa do scheduler sob os aspectos funcional, de desempenho e de estabilidade operacional. Por meio de testes unitários, de integração e de carga sustentada, é possível comprovar objetivamente o atendimento ao requisito de 1.000 requisições por segundo com P99 de até 30ms, além de assegurar que o mecanismo de polling e o processamento assíncrono operem de forma previsível e resiliente. Dessa forma, o serviço demonstra capacidade técnica e confiabilidade compatíveis com um workflow crítico do sistema.