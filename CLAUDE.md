# Projeto Drogajil: atendimento e vendas por WhatsApp

## Quem é o usuário
- Nome: Jildean, dono da farmácia **Drogajil**.
- Não é programador e nunca usou o n8n. Precisa ser guiado passo a passo, do início ao fim.

## Como trabalhar com o Jildean
- Responder sempre em **português simples**, sem jargão. Ao usar um termo técnico, explicar em uma frase.
- **Uma etapa por vez.** Dizer exatamente onde clicar e o que preencher, e **esperar ele confirmar** que deu certo antes de seguir.
- Se não houver certeza de como algo funciona na versão atual do n8n ou de outro serviço, **dizer isso e pedir um print da tela**, em vez de adivinhar.
- **Nunca** colocar senhas, chaves de API, chave Pix ou dados de clientes nos arquivos do repositório. Usar espaços marcados como `[[PREENCHER]]`.
- Antes de criar ou mudar muitos arquivos, **mostrar o plano e esperar aprovação**.

## Objetivo do sistema
Um atendente virtual com IA (Claude, da Anthropic) responde os clientes no WhatsApp, consulta produtos, preço e estoque numa planilha do Google, registra pedidos e aciona a entrega por motoboy. Casos sensíveis vão para uma pessoa.

### Ferramentas
- **n8n Cloud**: o orquestrador que liga tudo (conta já criada). Endereço da instância: `https://jildean.app.n8n.cloud`. Em 09/10/2026 estava no período de teste (12 dias restantes, limite de 1000 execuções).
- **Google Planilhas**: a base de dados (já criada).
- **API da Anthropic**: o modelo de IA do atendente.
- **WhatsApp Business Cloud API (Meta)**: o canal de atendimento. Fica por último.

## Regras da farmácia
- Atendimento das **08:00 às 22:00, todos os dias**.
- **Fora do horário:** a IA aceita e registra o pedido, mas a entrega só sai a partir das 08:00. Um fluxo agendado às 08:00 deve acionar as entregas pendentes.
- **Entrega** só no **Recanto das Emas** e no **Riacho Fundo**. Taxa única de **R$ 10,00**. Prazo de **30 a 40 minutos**.
- **Pagamento:** Pix, cartão na entrega ou dinheiro. A IA **nunca** confirma sozinha que um Pix caiu.
- **Medicamento que exige receita:** a IA pede foto da receita e transfere para o farmacêutico. Nada é registrado ou entregue antes da validação. **Controlados sempre** passam pelo farmacêutico.
  - **Decisão do Jildean:** o pedido **só é gravado na aba Pedidos depois que o farmacêutico aprovar** a receita. Antes disso, não existe linha em Pedidos.
- A IA **não** dá orientação de dose, diagnóstico ou troca de remédio. Em emergência, orienta ligar **192 (SAMU)**.
- **Transferências:** receitas e dúvidas de saúde vão para o **farmacêutico**. O resto (reclamação, troca, erro, pedido do cliente) vai para o **atendente**.

## A planilha (Google Planilhas, já pronta)
Nome: **"Drogajil: Sistema de Atendimento e Vendas"**

ID da planilha (para o n8n): `18baUg9uQS0S2nbfmkEorTYS0JNUQMLAVL5Id_uP0hbI`

Os cabeçalhos estão sempre na **linha 1**. **Não mudar os nomes das colunas.**

| Aba | Colunas |
|---|---|
| Configuração | parametro, valor, observacao |
| Produtos | id_produto, nome, principio_ativo, categoria, apresentacao, preco, exige_receita, controlado, ativo, estoque_atual (calculado) |
| Estoque | id_produto, produto (calculado), quantidade, estoque_minimo, situacao (calculado), atualizado_em |
| Clientes | telefone, nome, endereco, numero, complemento, bairro, ponto_referencia, cadastrado_em, observacoes |
| Pedidos | id_pedido, data_hora, telefone_cliente, nome_cliente, endereco_entrega, bairro, subtotal, taxa_entrega, total, forma_pagamento, troco_para, status, atendido_por, observacoes |
| Itens_Pedidos | id_pedido, id_produto, produto, quantidade, preco_unitario, subtotal_item |
| Pagamentos | id_pagamento, id_pedido, data_hora, forma_pagamento, valor, status, conferido_por, comprovante |
| Entregas | id_entrega, id_pedido, motoboy, endereco, bairro, status, saiu_em, entregue_em, observacoes |
| Atendimentos | id_atendimento, data_hora, telefone_cliente, assunto, resumo, transferido_para, status, id_pedido |
| Atendentes | nome, funcao, telefone_whatsapp, ativo, horario |
| Receitas | id_receita, data_hora, telefone_cliente, id_pedido, link_foto, tipo, status, farmaceutico, observacoes |
| Vendas | painel automático (não recebe dados) |
| Listas | opções das listas suspensas |

- Status de pedido (aba Listas): **Novo, Aguardando receita, Aguardando entrega, Saiu para entrega, Entregue, Cancelado.**
- Colunas marcadas como **"calculado"** têm fórmulas: o n8n **nunca** deve escrever nelas.
- A aba **Receitas** e os **dados de clientes** são dados de saúde protegidos pela **LGPD**.

### Dados de teste (preenchidos em 09/10/2026)
- **Produtos e Estoque:** catálogo de 163 produtos (`P0001` a `P0163`) em todas as categorias. Fica para uso real, mas os **preços são aproximados** e o Jildean deve conferir antes de vender. Cópia em `dados/produtos.csv`.
- **Dados fictícios, para apagar antes de abrir para clientes:** Clientes (10, telefones `5561900000001` a `5561900000010`), Pedidos (`PED-0001` a `PED-0007`), Itens_Pedidos, Pagamentos (`PAG-`), Entregas (`ENT-`), Atendimentos (`ATD-`), Receitas (`R0001` a `R0003`).
- **Atendentes:** nomes fictícios com telefone `[[PREENCHER]]`. O Jildean precisa colocar os números reais antes de testar transferências.
- Formato dos códigos usados: `PED-0001`, `PAG-0001`, `ENT-0001`, `ATD-0001`, `R0001`, `P0001`. Os fluxos do n8n devem seguir o mesmo formato.

## O atendente de IA no n8n
- Roda no nó **AI Agent**, com o modelo **Claude**.
- **Memória por conversa**, com chave = telefone do cliente.
- Sete ferramentas, com **exatamente** estes nomes:
  `consultar_produtos`, `consultar_cliente`, `salvar_cliente`, `registrar_pedido`, `acionar_entrega`, `transferir_farmaceutico`, `transferir_atendente`.
- O **prompt de sistema** do atendente já está escrito pelo Jildean. Pedir para ele colar quando for a hora.
- Os **cálculos de dinheiro** (subtotal, taxa, total) são feitos pelo **fluxo do n8n**, não pela IA, para evitar erro de conta.

## Situação atual
- **Já tem:** a planilha criada, o prompt do atendente escrito, a conta no n8n Cloud, a conta na Anthropic (platform.claude.com) com US$ 5 de crédito e a chave de API `n8n-drogajil` criada, sem prazo de validade (guardada só com o Jildean, nunca no repositório).
- **Credenciais no n8n:** `Anthropic Drogajil` (testada com sucesso em 09/10/2026) e a credencial do Google Planilhas (tipo "Google Sheets OAuth2 API", conta jildean.ss@gmail.com, "Account connected" em 09/10/2026; nome sugerido `Google Planilhas Drogajil`, antes chamada "Google Sheets account 2"). Existe também uma credencial antiga `Google Sheets account` com aviso "Needs first setup", criada pelo Assistant do n8n e ligada a 1 fluxo: apagar depois, com cuidado.
- **Ainda não tem:** conexão com o WhatsApp.
- **Próxima etapa:** 3B, fluxo de teste que lê a aba Produtos no n8n.
