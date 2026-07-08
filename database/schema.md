# Atlas Database Schema

## Companies

Representa cada empresa cliente do Atlas.

---

## Customers

Clientes que conversam com a empresa.

---

## Services

Serviços oferecidos pela empresa.

---

## Conversations

Cada atendimento realizado.

---

## Messages

Mensagens enviadas e recebidas durante uma conversa.

## Tabelas implementadas

- companies
- services
- knowledge
- conversations
- messages

## Relacionamentos

companies.id -> services.company_id

companies.id -> knowledge.company_id

companies.id -> conversations.company_id

conversations.id -> messages.conversation_id

## Objetivo das tabelas

### companies
Armazena as empresas cadastradas que utilizam o Atlas.

### services
Armazena os serviços oferecidos por cada empresa, como nome, duração e preço.

### knowledge
Base de conhecimento da empresa utilizada pelo Atlas para responder perguntas dos clientes.

### conversations
Armazena cada conversa iniciada entre um cliente e o Atlas.

### messages
Armazena todas as mensagens pertencentes a uma conversa, tanto do cliente quanto do Atlas.