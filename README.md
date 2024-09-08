1. Requisitos:

- Só realizar a venda de ingressos para pessoas que realmente vão receber o ingresso (não deixar pessoas comprarem um ingresso sem que haja mais ingressos disponíveis)
- A velocidade da internet não pode ser algo decisivo (se o cliente chegou antes, ele deveria conseguir prosseguir na compra do ingresso alocado a ele)

2. Entidades de dados/modelo de dados:

Uma base de dados relacional (como MySQL, PostgreSQL) seria a escolha adequada, pois a consistência dos dados é algo crítico nesse problema, como indicado pelo primeiro requisito acima. Bases de dados relacionais tem ACID (Atomicity, Consistency, Isolation and Durability) como uma de suas características centrais, o que não seria garantido em uma base dados não relacional.

- Usuário

  - ID: Primary Key
  - Nome: String
  - ...
  
- Banda:
  
  - ID: Primary Key
  - Nome: String
  - Gênero: String
  - Descrição: String
  - ...
    
- Show

  - ID: Primary Key
  - Banda: Foreign Key (Banda)
  - Festival: String
  - Endereço: String
  - ...
    
- Ingresso

  - ID: Primary Key
  - Usuário: Foreign Key (Usuário)
  - Show: Foreign Key (Show)
  - Setor: String
  - Meia: Boolean
  - Status da compra: String ("AVAILABLE" | "IN PROGRESS" | "BOOKED")
  - ...

3. APIs:

Não há necessidade de comunicação bidirecional e tempo real nesse problema, então uma RESTful API poderia ser utilizada.

- PATCH /<show_id>/ingresso/request: endpoint para iniciar a compra de um ingresso
- PATCH /<show_id>/ingresso/cancel: endpoint a ser acionado caso a pessoa desista da compra
- PATCH /<show_id>/ingresso/book: endpoint a ser acionado quando a compra for finalizada (depois do pagamento, o qual é geralmente realizado por um third-party service)

4. High-level design:

- Uma arquitetura de microserviços seria algo bem conveniente nesse problema, pois o serviço de venda de ingressos precisa escalar independentemente e diferentemente dos demais serviços (login, reset de senha, busca de shows, gestão de conta/perfil...).
- Uma "sala de espera" precisa ser implementada. A ideia central é que somente um numero de pessoas igual ao numero de ingressos disponiveis possam iniciar o processo de compra. As demais pessoas devem aguardar que alguém desista da compra para que ingressos voltem a ficar disponíveis.
- Essa "sala de espera" seria implementada através de um serviço de fila (opções populares são AWS SQS, Kafka, RabbitMQ). Essa fila seria do tipo FIFO (First In, First Out) para garantir que a ordem de chegada das pessoas fosse respeitada.
- Um sistema de notificações deveria ser utilizado para notificar a aplicação do cliente que um ingresso está disponível para aquela pessoa.
- A maioria dos sistemas de compra possui um tempo máximo dentro do qual a compra tem que ser realizada. Isso evitaria que um comprador consiga iniciar a compra e deixar ela indefinitivamente em andamento.

Fluxo:

- O usuário seleciona a opção de "iniciar compra", o que é convertido em uma mensagem para o serviço de filas pelo back-end do site de compra (producer).
- Os workers (consumers) da fila verificariam se há ingressos disponíveis.
- Caso haja, o status da compra do ingresso passa para "IN PROGRESS" e isso vai impedir que um outro usuário tente comprá-lo. Nesse momento, o sistema de notificações avisaria a aplicação do cliente que ele pode prosseguir para os próximos passos da compra (preenchimento de dados, pagamento, etc).
- Caso não haja ingressos disponíveis, os usuários devem ser redirecionados para uma tela de espera.
- Quando uma compra é finalizada, o status de compra do ingresso passa para "BOOKED".
- Quando um ingresso for liberado (por desistência, demora para finalizar a compra, etc), o status de compra retorna para "AVAILABLE", o sistema de notificações avisa o próximo da fila, o qual vai ter a oportunidade de realizar a compra.

5. Deep dive:

Algo que vale a pena ser debatido diz respeito ao mecanismo que libera um ingresso para compra. Como mencionado, o status de compra "IN PROGRESS" indica que alguém está no meio do processo de compra de um dado ingresso, evitando que ele seja simultaneamente disputado por dois usuários. No entanto, caso esse cliente deseje desistir da compra ou demore demais para concluir o processo, esse ingresso deve ter o status de compra setado para "AVAILABLE" novamente. Vou propor dois modos de fazer isso:

- Podemos ter um cron job que verifica de tempos em tempos se a compra do ingresso expirou. Para isso, precisaríamos guardar na base o momento onde a compra foi iniciada.
- Um modo mais "sofisticado" de implementar isso seria por meio de uma distributed lock. Distributed locks permitem definir TTLs (Time To Live), o que faria com que o ingresso fosse liberado após certo tempo.
