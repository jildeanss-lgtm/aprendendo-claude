# Ferramenta `registrar_pedido`

Fluxo separado (sub-workflow) que o Agente de IA chama. Arquivo para importar: `registrar_pedido.json`
(no n8n: criar um fluxo vazio, clicar na área de trabalho e colar com Ctrl+V).

Nome do fluxo no n8n: **Ferramenta - registrar_pedido**

## O que o fluxo faz
1. Lê as abas Configuração (taxa de entrega e bairros), Produtos e Clientes (só o telefone da conversa).
2. Confere os itens (existe, ativo, sem receita, não controlado, estoque suficiente), o cadastro e o bairro do cliente, a forma de pagamento e o troco.
3. Calcula subtotal, taxa e total em centavos (sem erro de arredondamento).
4. `confirmado` = false: devolve o resumo sem gravar nada.
5. `confirmado` = true: gera o próximo `PED-0000` (e `PAG-0000` com o mesmo número) e grava nas abas Pedidos (status "Novo", atendido por "IA"), Itens_Pedidos e Pagamentos (status "Pendente").

A gravação usa "USER_ENTERED": a data entra como data de verdade (o painel Vendas usa isso em "Pedidos hoje"),
os valores vão como texto com vírgula ("17,80") para a planilha em português entender como número, e o telefone vai com apóstrofo para ficar como texto.

O estoque NÃO é baixado automaticamente (decisão do Jildean em 09/10/2026, opção A).

## Ligação no Agente de IA ("Call n8n Workflow Tool")
Nome do nó: `registrar_pedido`

Descrição da ferramenta:
```
Calcula ou registra um pedido do cliente que está conversando agora. O telefone, os preços, a taxa de entrega e o endereço vêm do sistema: nunca informe valores. Use confirmado = false para receber o resumo calculado (nada é gravado) e mostrar ao cliente. Use confirmado = true somente depois que o cliente disser "sim" ao resumo, com os mesmos itens, e só uma vez por pedido. Não use para itens que exigem receita ou são controlados.
```

Campos (Workflow Inputs):
| Campo | Como preencher | Descrição para a IA |
|---|---|---|
| telefone | Expressão: `{{ $('Dados da conversa').first().json.telefone }}` | (não é a IA que preenche) |
| itens | ✨ | Lista em JSON com o código e a quantidade de cada produto, por exemplo: [{"id_produto": "P0001", "quantidade": 2}]. Use o id_produto que consultar_produtos retornou. |
| forma_pagamento | ✨ | Pix, Cartão na entrega ou Dinheiro. |
| troco_para | ✨ | Só para pagamento em dinheiro: o valor da nota que o cliente vai usar, por exemplo 100. Deixe vazio se não precisar de troco. |
| confirmado | ✨ | false para só calcular o resumo; true para gravar o pedido depois do "sim" do cliente. |
| observacoes | ✨ | Observação do cliente sobre a entrega, se houver. Pode ficar vazio. |
