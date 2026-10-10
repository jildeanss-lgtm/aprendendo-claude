# Ferramenta `acionar_entrega`

Fluxo separado (sub-workflow) que o Agente de IA chama logo depois de `registrar_pedido`.
Arquivo para importar: `acionar_entrega.json` (fluxo vazio no n8n, clicar na área de trabalho, Ctrl+V).

Nome do fluxo no n8n: **Ferramenta - acionar_entrega**. Precisa estar **Publicado**.
Não renomear os nós (o código procura os nós pelo nome).

## O que o fluxo faz
1. Lê Configuração (horário de abertura e fechamento), Pedidos, Entregas e Atendentes.
2. Confere: o pedido existe (aceita "PED-0008", "ped-8" ou "8"), é do telefone da conversa, não está cancelado e ainda não tem entrega.
3. Dentro do horário: cria a entrega com o motoboy do turno (aba Atendentes: função Motoboy, ativo Sim, horário "08:00 às 15:00").
   Fora do horário: cria a entrega sem motoboy, com a observação "sai a partir das 08:00" (o fluxo das 08:00 da etapa 9 cuida dela).
4. Grava em Entregas (próximo `ENT-0000`, status "Pendente") e muda o status do pedido para "Aguardando entrega".

Ainda não avisa o motoboy no celular: isso depende do WhatsApp (etapas 12 a 14).

## Ligação no Agente de IA ("Chame a ferramenta de fluxo de trabalho n8n")
Nome do nó: `acionar_entrega`

Descrição da ferramenta:
```
Aciona a entrega por motoboy de um pedido que acabou de ser registrado. Use logo depois que registrar_pedido retornar o número do pedido, uma única vez por pedido. O sistema decide se a entrega sai agora ou a partir das 08:00; repasse ao cliente o que a ferramenta responder.
```

| Campo | Como preencher | Descrição para a IA |
|---|---|---|
| telefone | Expressão: `{{ $('Dados da conversa').first().json.telefone }}` | (não é a IA que preenche) |
| id_pedido | ✨ | O número do pedido que registrar_pedido retornou, por exemplo PED-0008. |
