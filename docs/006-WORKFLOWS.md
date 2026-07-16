# Atlas Workflows

## Objetivo

Este documento define os workflows responsáveis pela execução das operações do Atlas.

A arquitetura utiliza workflows separados por responsabilidade, permitindo que cada parte do sistema seja desenvolvida, testada e modificada de forma independente.

O n8n será utilizado como camada de orquestração, conectando os diferentes componentes do Atlas, como:

- canais de comunicação;
- banco de dados;
- inteligência artificial;
- serviços internos;
- agendamentos;
- integrações externas.

---

## Princípios da arquitetura

Os workflows do Atlas seguem os seguintes princípios:

### Responsabilidade única

Cada workflow deve possuir uma função principal bem definida.

### Modularidade

Os workflows devem poder ser reutilizados por diferentes partes do sistema.

### Independência de canal

O núcleo do Atlas não deve depender diretamente do WhatsApp ou de qualquer outro canal específico.

Uma mensagem poderá futuramente chegar por diferentes canais, como:

- WhatsApp;
- Instagram;
- site;
- aplicativo;
- outros canais.

Todos devem poder utilizar o mesmo núcleo de processamento.

### Multiempresa

Toda operação relacionada a uma empresa deve utilizar o `company_id`.

Isso garante que os dados, serviços, configurações, conversas e conhecimentos de cada empresa permaneçam separados.

### Rastreabilidade

As conversas e mensagens processadas pelo Atlas devem ser registradas para permitir:

- histórico;
- auditoria;
- análise de erros;
- melhoria do sistema;
- continuidade das conversas.

---

## Workflows previstos

### atlas-webhook

Responsável por receber eventos externos e transformar os dados recebidos em um formato padronizado para o Atlas.

### atlas-core

Responsável por coordenar o processamento principal de uma mensagem.

### atlas-knowledge

Responsável por buscar e preparar as informações necessárias da base de conhecimento da empresa.

### atlas-save-conversation

Responsável por criar e atualizar conversas.

### atlas-save-message

Responsável por registrar mensagens enviadas e recebidas.

### atlas-send-message

Responsável por enviar a resposta do Atlas ao canal de comunicação correspondente.

### atlas-scheduler

Responsável por executar tarefas programadas e processos que dependem de horário.

---

## Fluxo geral

O fluxo principal de uma mensagem será:

1. Um canal externo envia um evento.
2. `atlas-webhook` recebe e normaliza os dados.
3. A empresa responsável pela mensagem é identificada.
4. `atlas-core` inicia o processamento.
5. A conversa é localizada ou criada.
6. A mensagem recebida é registrada.
7. `atlas-knowledge` busca o contexto necessário.
8. O prompt é construído.
9. O modelo de inteligência artificial é consultado.
10. A resposta gerada é registrada.
11. `atlas-send-message` envia a resposta ao canal correto.
12. O resultado do processamento é registrado.

---

## Regra de comunicação entre workflows

Os workflows devem trocar dados utilizando estruturas padronizadas.

Sempre que possível, um workflow deve:

1. receber uma entrada definida;
2. validar os dados recebidos;
3. executar apenas sua responsabilidade;
4. retornar uma saída definida;
5. registrar erros de forma rastreável.

Essa padronização permitirá substituir ou modificar partes do sistema sem alterar toda a arquitetura.
---

# Detalhamento dos workflows

## atlas-webhook

### Responsabilidade

O `atlas-webhook` é a porta de entrada para eventos externos recebidos pelo Atlas.

Sua responsabilidade é receber os dados enviados por um canal de comunicação, validar as informações essenciais e transformar o evento recebido em uma estrutura interna padronizada.

O `atlas-webhook` não deve executar a lógica principal da inteligência artificial. Após normalizar os dados, ele deve encaminhar a requisição para o `atlas-core`.

---

### Canais suportados

Inicialmente, o Atlas poderá receber mensagens por:

- WhatsApp.

A arquitetura deve permitir a inclusão futura de outros canais, como:

- Instagram;
- site;
- aplicativo;
- outros serviços de comunicação.

---

### Entrada

Cada provedor poderá enviar dados em formatos diferentes.

Por exemplo, um provedor de WhatsApp poderá enviar informações como:

- identificador da empresa;
- número do cliente;
- nome do cliente;
- conteúdo da mensagem;
- tipo da mensagem;
- identificador externo da mensagem;
- data e hora do evento.

O `atlas-webhook` deve converter esses dados para o formato interno do Atlas.

---

### Estrutura normalizada

Após o processamento inicial, o `atlas-webhook` deve produzir uma estrutura padronizada semelhante a:

```json
{
  "company_id": "uuid-da-empresa",
  "channel": "whatsapp",
  "customer": {
    "name": "Nome do cliente",
    "phone": "5511999999999"
  },
  "message": {
    "external_id": "identificador-da-mensagem",
    "type": "text",
    "content": "Mensagem enviada pelo cliente",
    "timestamp": "2026-07-13T18:00:00Z"
  }
}

---

## atlas-core

### Responsabilidade

O `atlas-core` é o orquestrador central do processamento de mensagens do Atlas.

Sua responsabilidade é receber uma mensagem já validada e normalizada pelo `atlas-webhook` e coordenar os workflows e serviços necessários para produzir uma resposta.

O `atlas-core` deve controlar a sequência do processamento, mas não deve concentrar responsabilidades que pertencem a outros workflows.

---

### Entrada

O `atlas-core` recebe a estrutura normalizada produzida pelo `atlas-webhook`.

Exemplo:

```json
{
  "company_id": "uuid-da-empresa",
  "channel": "whatsapp",
  "customer": {
    "name": "Nome do cliente",
    "phone": "5511999999999"
  },
  "message": {
    "external_id": "identificador-da-mensagem",
    "type": "text",
    "content": "Mensagem enviada pelo cliente",
    "timestamp": "2026-07-13T18:00:00Z"
  }
}

### Orquestração

O `atlas-core` utiliza a estrutura normalizada recebida do `atlas-webhook` como contexto principal para coordenar o processamento da mensagem.

Durante o processamento, o `atlas-core` deve preservar, no mínimo:

- `company_id`;
- `channel`;
- dados do `customer`;
- dados da `message`.

A partir desse contexto, o `atlas-core` coordena os workflows necessários de acordo com a etapa do processamento.

Fluxo principal previsto:

```text
atlas-webhook
      ↓
atlas-core
      ├──→ atlas-knowledge
      ├──→ atlas-save-conversation
      ├──→ atlas-save-message
      └──→ atlas-send-message
Cada workflow recebe apenas os dados necessários para executar sua responsabilidade.

Os contratos específicos de entrada e saída de cada workflow devem ser respeitados pelo `atlas-core`.

### Saída

O `atlas-core` não possui uma única saída intermediária fixa para todos os workflows.

Como orquestrador, sua responsabilidade é coordenar o fluxo de processamento, consumir as saídas dos workflows chamados e utilizar esses resultados nas etapas seguintes.

Ao final do processamento síncrono de uma mensagem, o `atlas-core` deve produzir um resultado de processamento padronizado.

Exemplo:

```json
{
  "success": true,
  "company_id": "uuid-da-empresa",
  "conversation_id": "uuid-da-conversa",
  "incoming_message_id": "uuid-da-mensagem-recebida",
  "outgoing_message_id": "uuid-da-mensagem-enviada",
  "status": "completed"
}
---

## atlas-knowledge

### Responsabilidade

O `atlas-knowledge` é responsável por buscar e organizar as informações da empresa necessárias para que o Atlas produza respostas contextualizadas.

Sua função é consultar os dados disponíveis para uma empresa específica e retornar um contexto estruturado para o `atlas-core`.

Toda consulta deve respeitar obrigatoriamente o `company_id`, garantindo o isolamento dos dados entre diferentes empresas.

---

### Entrada

O `atlas-knowledge` recebe uma solicitação contendo, no mínimo, a identificação da empresa e a mensagem atual do cliente.

Exemplo:

```json
{
  "company_id": "uuid-da-empresa",
  "message": {
    "content": "Quanto custa o corte de cabelo?"
  }
}

---

### Saída

Após consultar a base de conhecimento da empresa, o workflow deve retornar um contexto estruturado para o `atlas-core`.

Exemplo:

```json
{
  "company_id": "uuid-da-empresa",
  "context": {
    "services": [
      {
        "name": "Corte de cabelo",
        "price": 40.00
      }
    ],
    "information": [
      "Horário de atendimento: 09:00 às 18:00"
    ]
  },
  "status": "completed"
}

---

## atlas-save-conversation

### Responsabilidade

O `atlas-save-conversation` é responsável por criar, localizar e atualizar uma conversa entre um cliente e o Atlas.

Sua função é garantir que cada interação esteja associada a uma conversa válida antes que as mensagens sejam armazenadas.

O workflow deve sempre operar dentro do contexto de uma empresa específica, utilizando o `company_id` para garantir o isolamento dos dados.

---

### Entrada

O `atlas-save-conversation` recebe os dados necessários para identificar a empresa e o cliente.

Exemplo:

```json
{
  "company_id": "uuid-da-empresa",
  "customer": {
    "name": "Nome do cliente",
    "phone": "5511999999999"
  }
}

---

### Saída

Após criar ou localizar a conversa, o workflow deve retornar o identificador da conversa criada ou encontrada.

Exemplo:

```json
{
  "company_id": "uuid-da-empresa",
  "conversation": {
    "id": "uuid-da-conversa",
    "customer": {
      "name": "Nome do cliente",
      "phone": "5511999999999"
    },
    "status": "active"
  }
}

---

## atlas-save-message

### Responsabilidade

O `atlas-save-message` é responsável por armazenar uma mensagem pertencente a uma conversa.

O workflow pode ser utilizado tanto para mensagens recebidas de um cliente quanto para mensagens geradas e enviadas pelo Atlas.

Toda mensagem deve estar obrigatoriamente associada a uma conversa válida por meio do `conversation_id`.

---

### Entrada

O `atlas-save-message` recebe os dados necessários para identificar a conversa e armazenar a mensagem.

Exemplo de uma mensagem recebida do cliente:

```json
{
  "conversation_id": "uuid-da-conversa",
  "sender": "customer",
  "message": "Quanto custa o corte de cabelo?",
  "message_type": "text",
  "direction": "inbound",
  "status": "received"
}
```

Exemplo de uma mensagem gerada pelo Atlas:

```json
{
  "conversation_id": "uuid-da-conversa",
  "sender": "atlas",
  "message": "O corte de cabelo custa R$ 40,00.",
  "message_type": "text",
  "direction": "outbound",
  "status": "generated"
}
```
---

### Saída

Após armazenar a mensagem, o workflow deve retornar os dados da mensagem criada.

Exemplo:

```json
{
  "message": {
    "id": "uuid-da-mensagem",
    "conversation_id": "uuid-da-conversa",
    "sender": "customer",
    "message": "Quanto custa o corte de cabelo?",
    "message_type": "text",
    "direction": "inbound",
    "status": "received"
  }
}

---

### Processamento

Ao receber uma solicitação, o `atlas-save-message` deve:

1. validar o `conversation_id`;
2. verificar se a conversa existe;
3. validar o conteúdo da mensagem;
4. identificar o remetente da mensagem;
5. identificar a direção da mensagem;
6. definir o tipo da mensagem;
7. definir o status da mensagem;
8. salvar a mensagem na tabela `messages`;
9. retornar os dados da mensagem salva.

---

### Associação com a conversa

Toda mensagem pertence obrigatoriamente a uma conversa.

A relação utilizada é:

```text
conversations.id
        ↓
messages.conversation_id
```

O workflow nunca deve criar uma mensagem sem um `conversation_id` válido.

Essa relação permite recuperar posteriormente todo o histórico de mensagens pertencentes a uma conversa.

---

### Remetente

O campo `sender` identifica quem originou a mensagem.

Valores inicialmente previstos:

```text
customer
atlas
```

No futuro, outros remetentes poderão ser adicionados, como:

```text
human_agent
system
```

---

### Direção da mensagem

O campo `direction` identifica o sentido da comunicação.

Valores previstos:

```text
inbound
outbound
```

Onde:

- `inbound` representa uma mensagem recebida pelo Atlas;
- `outbound` representa uma mensagem enviada pelo Atlas.

---

### Tipo da mensagem

Na primeira versão do Atlas, o principal tipo de mensagem será:

```text
text
```

O campo `message_type` foi mantido na arquitetura para permitir futuras expansões, como:

```text
image
audio
document
location
```

Esses tipos não precisam ser implementados na primeira versão.

---

### Status da mensagem

O campo `status` representa o estado atual da mensagem.

Exemplos previstos:

```text
received
generated
sent
delivered
read
failed
```

Nem todos os estados precisam ser implementados inicialmente.

Na primeira versão, os principais estados serão:

```text
received
sent
failed
```

---

### Salvamento de mensagem recebida

Quando uma mensagem chegar de um cliente, o workflow deverá armazená-la antes da geração da resposta.

Exemplo conceitual:

```json
{
  "conversation_id": "uuid-da-conversa",
  "sender": "customer",
  "message": "Quais horários estão disponíveis hoje?",
  "message_type": "text",
  "direction": "inbound",
  "status": "received"
}
```

---

### Salvamento de mensagem gerada pelo Atlas

Após o Atlas gerar uma resposta, o conteúdo também deverá ser armazenado.

Exemplo conceitual:

```json
{
  "conversation_id": "uuid-da-conversa",
  "sender": "atlas",
  "message": "Temos horários disponíveis às 14h e às 16h.",
  "message_type": "text",
  "direction": "outbound",
  "status": "sent"
}
```

---

### Saída

Após salvar a mensagem, o workflow deve retornar uma estrutura padronizada.

Exemplo:

```json
{
  "message": {
    "id": "uuid-da-mensagem",
    "conversation_id": "uuid-da-conversa",
    "sender": "customer",
    "message": "Quanto custa o corte de cabelo?",
    "message_type": "text",
    "direction": "inbound",
    "status": "received",
    "created_at": "2026-07-13T18:00:00Z"
  }
}
```

---

### Histórico da conversa

As mensagens armazenadas poderão ser utilizadas para reconstruir o histórico de uma conversa.

Exemplo:

```text
Conversa
   │
   ├── Cliente: Olá
   ├── Atlas: Olá! Como posso ajudar?
   ├── Cliente: Quanto custa o corte?
   └── Atlas: O corte custa R$ 40,00.
```

Esse histórico poderá futuramente ser utilizado pelo `atlas-core` para fornecer contexto à inteligência artificial.

---

### Regra de isolamento de dados

Embora a tabela `messages` esteja diretamente relacionada à tabela `conversations`, o isolamento entre empresas continua obrigatório.

Uma mensagem pertence a uma conversa, e uma conversa pertence a uma empresa.

A cadeia de relacionamento é:

```text
companies.id
      ↓
conversations.company_id

conversations.id
      ↓
messages.conversation_id
```

Nenhuma mensagem de uma empresa deve ser utilizada no contexto de outra empresa.

---

### O que este workflow não deve fazer

O `atlas-save-message` não deve:

- gerar respostas;
- consultar a OpenAI;
- consultar a base de conhecimento;
- criar empresas;
- gerenciar serviços;
- enviar mensagens para canais externos;
- executar a lógica principal do `atlas-core`.

Sua responsabilidade é exclusivamente validar e armazenar mensagens.

---

### Tratamento de erros

Se o `conversation_id` for inválido ou inexistente, o workflow deve interromper a operação.

Se o conteúdo obrigatório da mensagem estiver ausente, o workflow deve retornar um erro de validação.

Falhas no banco de dados devem ser registradas de forma rastreável.

O workflow nunca deve criar uma mensagem sem uma conversa válida.

---

## atlas-send-message

### Responsabilidade

O `atlas-send-message` é responsável por enviar uma mensagem para um canal externo de comunicação.

Ele recebe uma mensagem já pronta e os dados necessários para realizar a entrega.

O workflow não deve decidir o conteúdo da resposta. Sua responsabilidade é exclusivamente realizar o envio da mensagem pelo canal solicitado.

Inicialmente, o principal canal utilizado pelo Atlas será o WhatsApp.

A arquitetura, porém, deve permitir a inclusão futura de outros canais sem alterar a lógica principal do sistema.

---

### Entrada

O `atlas-send-message` recebe os dados necessários para identificar a empresa, o canal, o destinatário e o conteúdo da mensagem.

Exemplo:

```json
{
  "company_id": "uuid-da-empresa",
  "conversation_id": "uuid-da-conversa",
  "message_id": "uuid-da-mensagem",
  "channel": "whatsapp",
  "recipient": {
    "phone": "5511999999999"
  },
  "message": {
    "type": "text",
    "content": "O corte de cabelo custa R$ 40,00."
  }
}
```

---

### Processamento

Ao receber uma solicitação, o `atlas-send-message` deve:

1. validar o `company_id`;
2. validar o canal solicitado;
3. validar os dados do destinatário;
4. validar o conteúdo da mensagem;
5. identificar a integração responsável pelo canal;
6. preparar a mensagem no formato exigido pelo provedor;
7. realizar a tentativa de envio;
8. interpretar a resposta do provedor;
9. retornar o resultado da operação.

---

---

### Saída

Após realizar o envio, o workflow deve retornar o resultado da operação.

Exemplo:

```json
{
  "success": true,
  "message_id": "uuid-da-mensagem",
  "provider": "whatsapp",
  "channel": "whatsapp",
  "status": "sent",
  "sent_at": "2026-07-13T18:00:00Z"
}

### Seleção do canal

O campo `channel` determina qual integração será utilizada para enviar a mensagem.

Inicialmente:

```text
whatsapp
```

No futuro, o Atlas poderá suportar canais adicionais, como:

```text
instagram
telegram
webchat
email
```

O `atlas-core` não deve conhecer os detalhes técnicos de cada provedor.

Essa responsabilidade pertence à camada de envio.

---

### Separação entre canal e provedor

O canal de comunicação e o provedor utilizado para operar esse canal são conceitos diferentes.

Exemplo:

```text
Canal:
whatsapp

Provedor:
integração configurada para aquela empresa
```

Essa separação permite substituir uma integração de WhatsApp por outra sem alterar a lógica principal do Atlas.

O `atlas-send-message` deve utilizar a configuração correspondente à empresa identificada pelo `company_id`.

---

### Envio de mensagem

Após validar os dados, o workflow deve preparar a requisição exigida pelo provedor responsável pelo canal.

Exemplo conceitual:

```text
Atlas
   ↓
atlas-send-message
   ↓
Adaptador do canal
   ↓
Provedor externo
   ↓
Cliente
```

O formato enviado ao provedor poderá ser diferente do formato interno utilizado pelo Atlas.

O workflow é responsável por realizar essa adaptação.

---

### Saída em caso de sucesso

Quando a mensagem for enviada com sucesso, o workflow deve retornar uma estrutura padronizada.

Exemplo:

```json
{
  "success": true,
  "message_id": "uuid-da-mensagem",
  "channel": "whatsapp",
  "provider_message_id": "identificador-retornado-pelo-provedor",
  "status": "sent"
}
```

O `provider_message_id` representa o identificador da mensagem no sistema externo, quando esse identificador estiver disponível.

---

### Saída em caso de falha

Quando o envio falhar, o workflow deve retornar uma estrutura padronizada.

Exemplo:

```json
{
  "success": false,
  "message_id": "uuid-da-mensagem",
  "channel": "whatsapp",
  "status": "failed",
  "error": {
    "code": "MESSAGE_SEND_FAILED",
    "message": "Não foi possível enviar a mensagem."
  }
}
```

Erros internos do provedor não devem ser enviados diretamente ao cliente final.

---

### Atualização do status da mensagem

Após a tentativa de envio, o resultado poderá ser utilizado para atualizar o campo `status` da mensagem correspondente.

Exemplo:

```text
generated
    ↓
sent
```

Em caso de falha:

```text
generated
    ↓
failed
```

Estados futuros poderão incluir:

```text
delivered
read
```

Esses estados dependerão dos eventos disponibilizados pelo canal ou provedor utilizado.

---

### Regra de isolamento de dados

Toda operação deve respeitar o `company_id`.

As credenciais e configurações utilizadas para o envio devem pertencer exclusivamente à empresa correspondente.

O workflow nunca deve:

- utilizar credenciais de outra empresa;
- enviar uma mensagem utilizando a configuração de outra empresa;
- misturar destinatários entre empresas;
- expor credenciais ou tokens em logs ou respostas.

---

### Idempotência

O workflow deve evitar o envio duplicado da mesma mensagem sempre que possível.

O `message_id` poderá ser utilizado como referência interna para verificar se uma mensagem já foi processada.

Essa proteção é importante porque eventos externos ou execuções de workflows podem ser repetidos.

A estratégia exata de idempotência será definida durante a implementação.

---

### O que este workflow não deve fazer

O `atlas-send-message` não deve:

- gerar o conteúdo da resposta;
- consultar a OpenAI;
- consultar a base de conhecimento;
- criar conversas;
- decidir qual resposta deve ser enviada;
- executar a lógica principal do `atlas-core`.

Sua responsabilidade é exclusivamente entregar uma mensagem já preparada ao canal correto.

---

### Tratamento de erros

Se o `company_id` for inválido, o workflow deve interromper a operação.

Se o canal solicitado não for suportado, o workflow deve retornar um erro identificável.

Se os dados do destinatário estiverem ausentes ou inválidos, o envio não deve ser realizado.

Falhas de autenticação, indisponibilidade do provedor, timeout ou rejeição da mensagem devem ser registradas de forma rastreável.

Credenciais, tokens e outras informações sensíveis nunca devem ser incluídos nos logs de erro.

---

### Princípio de independência do provedor

A lógica principal do Atlas não deve depender diretamente de um provedor específico.

A arquitetura desejada é:

```text
atlas-core
    ↓
atlas-send-message
    ↓
adaptador do canal
    ↓
provedor externo
```

Isso permitirá substituir ou adicionar provedores sem reconstruir o núcleo do sistema.

---

## atlas-scheduler

### Responsabilidade

O `atlas-scheduler` é responsável por iniciar ações que precisam ser executadas em um momento futuro ou de forma recorrente.

Ele funciona como o componente de agendamento temporal do Atlas.

Sua responsabilidade é identificar tarefas que precisam ser executadas e iniciar o fluxo correspondente no momento adequado.

O `atlas-scheduler` não deve conter a lógica completa da tarefa executada. Ele deve apenas identificar que uma ação precisa acontecer e encaminhar a execução para o workflow responsável.

---

### Casos de uso

Exemplos de tarefas que poderão utilizar o `atlas-scheduler`:

```text
lembretes de agendamento
confirmações automáticas
mensagens de acompanhamento
tarefas recorrentes
verificações periódicas
reativação de clientes
execuções programadas
```

Nem todos esses casos precisam ser implementados na primeira versão do Atlas.

---

### Entrada

Uma tarefa agendada deve possuir informações suficientes para identificar:

- a empresa responsável;
- o tipo da tarefa;
- o momento da execução;
- os dados necessários para executar a ação.

Exemplo conceitual:

```json
{
  "company_id": "uuid-da-empresa",
  "task_type": "send_reminder",
  "scheduled_for": "2026-07-14T13:00:00Z",
  "payload": {
    "conversation_id": "uuid-da-conversa",
    "customer_phone": "5511999999999",
    "message": "Lembrete: seu horário está marcado para hoje às 15h."
  }
}
```

---

### Processamento

Ao executar, o `atlas-scheduler` deve:

1. identificar tarefas pendentes;
2. verificar quais tarefas estão prontas para execução;
3. validar o `company_id`;
4. validar os dados necessários da tarefa;
5. identificar o tipo da tarefa;
6. encaminhar a execução para o workflow responsável;
7. registrar o resultado da execução;
8. impedir execuções duplicadas sempre que possível.

---

### Saída

Após identificar e encaminhar a tarefa para o workflow responsável, o `atlas-scheduler` deve retornar o resultado da execução.

Exemplo:

```json
{
  "success": true,
  "task_id": "uuid-da-tarefa",
  "workflow": "atlas-send-message",
  "status": "scheduled",
  "executed_at": "2026-07-14T13:00:00Z"
}

### Execução de tarefas

O `atlas-scheduler` não deve executar diretamente todas as ações do sistema.

Exemplo:

```text
atlas-scheduler
       ↓
identifica tarefa
       ↓
identifica responsabilidade
       ↓
workflow responsável
       ↓
execução
```

Para uma mensagem agendada:

```text
atlas-scheduler
       ↓
atlas-send-message
       ↓
canal
       ↓
cliente
```

Essa separação mantém cada workflow responsável por uma única função.

---

### Tipos de tarefas

Inicialmente, os tipos de tarefa poderão ser representados por valores como:

```text
send_reminder
send_follow_up
scheduled_message
```

No futuro, novos tipos poderão ser adicionados sem alterar a responsabilidade principal do scheduler.

---

### Isolamento entre empresas

Toda tarefa agendada deve estar associada a um `company_id`.

O scheduler nunca deve executar uma tarefa sem identificar corretamente a empresa responsável.

Isso garante que:

- configurações permaneçam isoladas;
- mensagens sejam enviadas pelo canal correto;
- credenciais pertencentes a uma empresa não sejam utilizadas por outra;
- dados de clientes não sejam misturados.

---

### Idempotência

Uma tarefa não deve ser executada mais de uma vez quando sua execução deveria ser única.

O sistema deve possuir uma forma de identificar se uma tarefa:

```text
pending
processing
completed
failed
```

A estratégia definitiva de armazenamento e controle dessas tarefas será definida antes da implementação do scheduler.

---

### Falhas e novas tentativas

Uma tarefa poderá falhar temporariamente devido a fatores externos, como:

- indisponibilidade de um provedor;
- timeout;
- falha de conexão;
- erro temporário de uma API.

Quando apropriado, o sistema poderá realizar uma nova tentativa.

A estratégia de retry deverá possuir limites para impedir loops infinitos.

Exemplo conceitual:

```text
tentativa 1
    ↓
falha
    ↓
aguarda
    ↓
tentativa 2
    ↓
falha
    ↓
registra erro
```

---

### Saída

Após processar uma tarefa, o workflow deve retornar um resultado padronizado.

Exemplo de sucesso:

```json
{
  "success": true,
  "task_id": "uuid-da-tarefa",
  "task_type": "send_reminder",
  "status": "completed",
  "executed_at": "2026-07-14T13:00:02Z"
}
```

Exemplo de falha:

```json
{
  "success": false,
  "task_id": "uuid-da-tarefa",
  "task_type": "send_reminder",
  "status": "failed",
  "error": {
    "code": "TASK_EXECUTION_FAILED",
    "message": "Não foi possível executar a tarefa."
  }
}
```

---

### O que este workflow não deve fazer

O `atlas-scheduler` não deve:

- gerar respostas utilizando inteligência artificial;
- consultar diretamente a OpenAI;
- executar a lógica principal do `atlas-core`;
- armazenar mensagens diretamente;
- implementar internamente todas as ações agendadas;
- depender exclusivamente de um único canal de comunicação.

Sua responsabilidade é coordenar a execução temporal de tarefas.

---

### Tratamento de erros

Se o `company_id` for inválido, a tarefa não deve ser executada.

Se o tipo da tarefa não for reconhecido, o workflow deve registrar um erro identificável.

Se os dados necessários estiverem ausentes, a tarefa deve falhar de forma controlada.

Falhas devem ser registradas de forma rastreável sem expor informações sensíveis.

---

### Observação de implementação

A arquitetura atual ainda não possui uma tabela específica para armazenar tarefas agendadas.

Antes da implementação real do `atlas-scheduler`, deverá ser decidido se será criada uma tabela dedicada, por exemplo:

```text
scheduled_tasks
```

Essa tabela poderá futuramente armazenar informações como:

```text
id
company_id
task_type
payload
scheduled_for
status
attempts
last_error
created_at
executed_at
```

A criação dessa tabela não é necessária neste momento e deverá ser decidida durante a preparação da implementação do scheduler.

---

# Contratos de comunicação entre workflows

## Objetivo

Os workflows do Atlas possuem responsabilidades independentes e não precisam compartilhar exatamente a mesma estrutura de entrada e saída.

O `atlas-core` atua como o principal orquestrador do fluxo e é responsável por utilizar os dados retornados por um workflow para construir a entrada necessária para o próximo.

Essa abordagem evita acoplamento excessivo entre os workflows.

---

## Evento normalizado de entrada

O `atlas-webhook` deve transformar eventos recebidos de canais externos em uma estrutura interna padronizada.

Formato base:

```json
{
  "company_id": "uuid-da-empresa",
  "channel": "whatsapp",
  "customer": {
    "name": "Nome do cliente",
    "phone": "5511999999999"
  },
  "message": {
    "external_id": "identificador-da-mensagem",
    "type": "text",
    "content": "Mensagem enviada pelo cliente",
    "timestamp": "2026-07-13T18:00:00Z"
  }
}
```

Essa estrutura representa o evento normalizado recebido pelo `atlas-core`.

---

## Papel do atlas-core

O `atlas-core` recebe o evento normalizado e coordena os workflows necessários.

O `atlas-core` pode transformar os dados recebidos para construir contratos específicos para cada workflow.

Exemplo:

```text
Evento normalizado
       ↓
atlas-core
       ↓
entrada específica do workflow
```

Isso significa que um workflow não precisa conhecer toda a estrutura original recebida pelo webhook.

Ele deve receber apenas os dados necessários para executar sua responsabilidade.

---

## Fluxo principal de processamento

O fluxo principal de uma mensagem recebida é:

```text
Canal externo
      ↓
atlas-webhook
      ↓
evento normalizado
      ↓
atlas-core
      ↓
atlas-save-conversation
      ↓
conversation_id
      ↓
atlas-save-message
      ↓
atlas-knowledge
      ↓
geração da resposta
      ↓
atlas-save-message
      ↓
atlas-send-message
      ↓
Canal externo
      ↓
Cliente
```

---

## Contrato: atlas-webhook → atlas-core

O `atlas-webhook` entrega ao `atlas-core` o evento normalizado.

Dados mínimos esperados:

```text
company_id
channel
customer
message
```

O `atlas-core` assume a responsabilidade de coordenar o processamento após essa etapa.

---

## Contrato: atlas-core → atlas-save-conversation

O `atlas-core` envia os dados necessários para identificar ou criar uma conversa.

Exemplo:

```json
{
  "company_id": "uuid-da-empresa",
  "customer": {
    "name": "Nome do cliente",
    "phone": "5511999999999"
  }
}
```

O `atlas-save-conversation` retorna uma conversa válida contendo um `conversation_id`.

---

## Contrato: atlas-save-conversation → atlas-core

Saída mínima esperada:

```json
{
  "conversation": {
    "id": "uuid-da-conversa",
    "company_id": "uuid-da-empresa",
    "customer_name": "Nome do cliente",
    "customer_phone": "5511999999999",
    "status": "active"
  }
}
```

O `atlas-core` utiliza:

```text
conversation.id
```

como:

```text
conversation_id
```

nos próximos workflows.

---

## Contrato: atlas-core → atlas-save-message

Para uma mensagem recebida:

```json
{
  "conversation_id": "uuid-da-conversa",
  "sender": "customer",
  "message": "Mensagem enviada pelo cliente",
  "message_type": "text",
  "direction": "inbound",
  "status": "received"
}
```

Para uma resposta do Atlas:

```json
{
  "conversation_id": "uuid-da-conversa",
  "sender": "atlas",
  "message": "Resposta gerada pelo Atlas",
  "message_type": "text",
  "direction": "outbound",
  "status": "generated"
}
```

---

## Contrato: atlas-core → atlas-knowledge

O `atlas-core` envia apenas os dados necessários para buscar informações relevantes.

Exemplo:

```json
{
  "company_id": "uuid-da-empresa",
  "message": {
    "content": "Quanto custa o corte de cabelo?"
  }
}
```

O `atlas-knowledge` retorna informações pertencentes exclusivamente à empresa identificada.

---

## Contrato: atlas-core → geração de resposta

O componente responsável pela geração da resposta deve receber contexto suficiente para produzir uma resposta adequada.

Esse contexto poderá incluir:

```text
configurações da empresa
serviços
base de conhecimento
mensagem atual
histórico relevante da conversa
instruções do sistema
```

A estrutura definitiva desse contrato será detalhada antes da implementação da camada de inteligência artificial.

---

## Contrato: atlas-core → atlas-send-message

Após a resposta ser gerada e armazenada, o `atlas-core` prepara a solicitação de envio.

Exemplo:

```json
{
  "company_id": "uuid-da-empresa",
  "conversation_id": "uuid-da-conversa",
  "message_id": "uuid-da-mensagem",
  "channel": "whatsapp",
  "recipient": {
    "phone": "5511999999999"
  },
  "message": {
    "type": "text",
    "content": "Resposta gerada pelo Atlas"
  }
}
```

O `atlas-send-message` é responsável apenas pela entrega ao canal externo.

---

## Regra de transformação de dados

Cada workflow deve receber apenas os dados necessários para cumprir sua responsabilidade.

O `atlas-core` pode:

- selecionar campos;
- renomear campos;
- combinar resultados de diferentes workflows;
- construir a entrada do próximo workflow.

O `atlas-core` não deve:

- duplicar a responsabilidade interna de outros workflows;
- acessar dados de outra empresa;
- ignorar validações necessárias;
- acoplar a lógica principal a um provedor específico.

---

## Princípio de contrato explícito

Todo workflow deve possuir:

```text
entrada definida
      ↓
validação
      ↓
responsabilidade específica
      ↓
saída definida
```

Mudanças futuras em um workflow devem preservar seu contrato sempre que possível.

Quando uma alteração incompatível for necessária, os workflows dependentes deverão ser revisados antes da implementação.