# Questão 1 - Solução de Validação de Endereço Cadastral

A solução proposta consiste em um microsserviço responsável por validar o endereço cadastral de uma empresa a partir da comparação entre os dados retornados por duas APIs de terceiros: uma consulta via CNPJ e outra via CEP. O serviço compara unidade federativa, cidade e logradouro, retornando HTTP 200 quando houver correspondência e HTTP 404 quando não houver coincidência, mantendo resposta síncrona conforme exigido no enunciado.

A arquitetura adotada foi a Hexagonal (Ports and Adapters), isolando a regra de negócio das integrações externas. O domínio depende apenas de interfaces, enquanto as integrações são implementadas por meio do padrão Adapter, que padroniza as respostas das APIs. O chaveamento automático entre dois provedores de CEP foi implementado com o padrão Strategy, permitindo fallback após N tentativas sem impactar a lógica principal.

Para garantir resiliência e desempenho, foram aplicadas retentativas automáticas com backoff exponencial e o padrão Circuit Breaker, evitando falhas em cascata e protegendo o sistema contra indisponibilidades externas. A separação clara entre domínio e infraestrutura favorece testabilidade, manutenibilidade e aderência às boas práticas de engenharia, atendendo aos critérios de desempenho, testes e qualidade esperados para a solução.

github:  [desafio 01](https://github.com/alfaia/Challenge01-Shipay)