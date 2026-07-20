# Atlas Core

## Objetivo

Este documento descreve o funcionamento interno do Atlas.

Todo novo recurso deverá seguir esta arquitetura.

---

# Fluxo Principal

## Passo 1

Cliente envia uma mensagem pelo WhatsApp.

↓

## Passo 2

A mensagem chega ao Atlas.

↓

## Passo 3

O Atlas identifica qual empresa recebeu a mensagem.

↓

## Passo 4

O Atlas busca as informações dessa empresa.

↓

## Passo 5

O Atlas envia todo o contexto para a IA.

↓

## Passo 6

A IA gera uma resposta.

↓

## Passo 7

O Atlas registra a conversa no banco de dados.

↓

## Passo 8

A resposta é enviada ao cliente.

---

## Componentes envolvidos

- WhatsApp
- Atlas Core
- Banco de Dados
- Base de Conhecimento
- OpenAI