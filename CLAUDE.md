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
- O **prompt de sistema** aprovado está em `prompts/atendente.md` (versão revisada e aprovada pelo Jildean em 09/10/2026). Vai no campo System Message do AI Agent e **não pode ter nada que mude a cada mensagem** (como data e hora).
- **Data e hora vão na mensagem do usuário, não no prompt de sistema.** No AI Agent, "Source for Prompt (User Message)" = "Define below", em modo Expression:
  `[Data e hora atuais: {{ $now.setZone('America/Sao_Paulo').toFormat('dd/MM/yyyy HH:mm') }}]` + quebra de linha + `{{ $json.chatInput }}`. Quando ligar o WhatsApp, trocar `$json.chatInput` pelo campo do texto da mensagem.
- **Lição aprendida (09/10/2026):** o Claude (Haiku 5.5) guarda blocos de "pensamento" na memória da conversa, e eles só valem se o prompt de sistema for **idêntico** ao da hora em que foram criados. Com `{{ $now }}` no prompt de sistema, o texto mudava a cada minuto e a segunda mensagem dava erro "Invalid signature in thinking block ... system prompt differs". Mudar o prompt de sistema também quebra as conversas que já estão na memória: depois de editar o prompt, começar uma sessão nova no chat.
- **Decisão:** `registrar_pedido` tem dois modos. Com `confirmado` = false, o n8n calcula o resumo (subtotal, taxa, total) e devolve sem gravar; com `confirmado` = true, grava o pedido. A IA nunca faz contas.
- **Decisão:** a IA sempre chama `acionar_entrega` após registrar; o fluxo do n8n decide se a entrega sai agora ou a partir das 08:00.
- **Decisão:** pedido com receita ou controlado: a IA pede a foto, chama `transferir_farmaceutico` e não chama `registrar_pedido` nem `acionar_entrega`. A IA só sugere produto alternativo (mesmo princípio ativo) para itens sem receita.
- O telefone do cliente vem do WhatsApp e é passado pelo fluxo; a IA não pergunta o telefone.
- **Modelo:** durante a montagem (etapas 4 a 10) usar **Claude Haiku 5.5** para economizar créditos. Na etapa 11 (bateria de testes das regras) testar com **Claude Sonnet 5.5** e com Haiku 5.5, e o Jildean escolhe qual vai para produção. Preços (Anthropic, por milhão de tokens): Sonnet 5.5 US$ 2 entrada / US$ 10 saída; Haiku 5.5 US$ 0,10 / US$ 0,50.
- O Jildean usa o n8n com a **tradução automática do Chrome** ligada. Dar os nomes em português e o original em inglês entre parênteses. A tradução troca nomes (ex.: "Sonnet" vira "Soneto", "categoria" vira "tímpano"); o que ele digita não é traduzido.
- Os **cálculos de dinheiro** (subtotal, taxa, total) são feitos pelo **fluxo do n8n**, não pela IA, para evitar erro de conta.

## Situação atual
- **Já tem:** a planilha criada, o prompt do atendente escrito, a conta no n8n Cloud, a conta na Anthropic (platform.claude.com) com US$ 5 de crédito e a chave de API `n8n-drogajil` criada, sem prazo de validade (guardada só com o Jildean, nunca no repositório).
- **Credenciais no n8n:** `Anthropic Drogajil` (testada com sucesso em 09/10/2026) e a credencial do Google Planilhas (tipo "Google Sheets OAuth2 API", conta jildean.ss@gmail.com, "Account connected" em 09/10/2026; nome `Google Planilhas Drogajil`). Existe também uma credencial antiga `Google Sheets account` com aviso "Needs first setup", criada pelo Assistant do n8n e ligada a 1 fluxo: apagar depois, com cuidado.
- **Ainda não tem:** conexão com o WhatsApp.
- **Fluxos no n8n:** `Teste - ler produtos` (Trigger manually → Google Sheets "Get row(s) in sheet", aba Produtos). Fluxo completo executado com sucesso em 09/10/2026: os dois nós rodam e trazem 163 itens. O n8n acrescenta a coluna `row_number` (número da linha) nos dados lidos, e o preço vem como número (ex.: 8.9).
- **Lição aprendida no n8n:** quando aparecer "Problem saving workflow / Autosave failed", a ligação entre nós pode não ser salva e o fluxo roda só o primeiro nó. Solução que funcionou: apagar a linha, ligar de novo, Ctrl+S, F5 e conferir no painel Logs se todos os nós aparecem.
- **Fluxo `Atendente Drogajil`** (09/10/2026): Chat Trigger → AI Agent + Anthropic Chat Model (Claude Haiku 5.5) + Simple Memory (janela 20) + prompt em modo Expression. Teste sem ferramentas: cumprimento, data/hora, área de entrega, recusa de dose e emergência (192) corretos. Sem ferramentas ligadas, o modelo escreveu chamadas `<tool_call>` no texto; foi acrescentada uma linha no fim do prompt proibindo isso.
- **Etapa 5 concluída (09/10/2026):** ferramenta `consultar_produtos` (Google Sheets Tool, Get Row(s), aba Produtos, sem filtro, descrição manual). Lê os 163 produtos a cada consulta (cerca de 29 mil tokens por conversa com consulta; barato com Haiku, umas 20 vezes mais caro com Sonnet: se o Sonnet for escolhido, trocar por uma busca filtrada). Testes ok: preços da dipirona e do Dorflex, Gelol inativo com sugestão de Salonpas, Rivotril pede receita, data e hora corretas.
- **Etapa 6 (em andamento):** nó `Dados da conversa` (Edit Fields, Manual Mapping, campo `telefone` = `5561900000001` fixo para testes, "Include Other Input Fields" ligado) entre o Chat Trigger e o AI Agent. As ferramentas de cliente pegam o telefone com `{{ $('Dados da conversa').first().json.telefone }}`, nunca da conversa (segurança/LGPD). No WhatsApp, só esse nó muda.
- **Etapa 6 concluída (09/10/2026):**
  - `consultar_cliente`: Google Sheets Tool, Get Row(s), aba Clientes, filtro coluna `telefone` = expressão do nó `Dados da conversa`. Testado: reconheceu a Maria Aparecida Teste e só ela (sem o filtro, trazia todos os clientes).
  - `salvar_cliente`: Google Sheets Tool, Append or Update Row, aba Clientes, coluna correspondente `telefone` (expressão do `Dados da conversa`), nome/endereco/numero/complemento/bairro/ponto_referencia preenchidos pela IA (✨), `cadastrado_em` = expressão `$now` (dd/MM/yyyy HH:mm, fica como texto), coluna `observacoes` removida do mapeamento, Formato da célula = "Let n8n format" (telefone fica como texto). Testado com o telefone `5561900000011`: criou a linha 12 "Carlos Novo Teste", Riacho Fundo, telefone como texto.
  - O telefone de teste no `Dados da conversa` ficou `5561900000011` (cliente de teste novo, criado na etapa 6; apagar junto com os dados fictícios).
  - O Jildean apagou a `consultar_produtos` sem querer e recriou. Dica dada: Ctrl+Z desfaz; clicar em área vazia antes de apertar Delete.
- **Etapa 7 (em andamento):** plano aprovado em 09/10/2026; estoque **não** é baixado automaticamente (opção A). Fluxo `Ferramenta - registrar_pedido` (16 nós) importado no n8n em 10/10/2026 colando `fluxos/registrar_pedido.json`, credenciais escolhidas nos 7 nós de planilha. Detalhes e textos da ligação no agente em `fluxos/registrar_pedido.md`. Os nomes dos nós desse fluxo não podem ser trocados (o código procura os nós pelo nome).
- **7B (10/10/2026):** ferramenta `registrar_pedido` ligada ao agente ("Chame a ferramenta de fluxo de trabalho n8n", Fonte Database, fluxo `Ferramenta - registrar_pedido`, telefone por expressão, demais campos ✨).
- **Lição aprendida (10/10/2026):** com duas ferramentas chamadas na mesma resposta, o n8n remonta a mensagem do Claude em outra ordem e o Haiku 5.5 recusa ("Invalid signature in thinking block ... content before this block that was not present"). Solução: no nó do modelo, Opções → "Modo de Pensamento" desligado. Atenção para a etapa 11: o Sonnet 5.5 não aceita pensamento desligado do jeito normal (só `between_tools`), então pode ter o mesmo problema no n8n.
- **Lição aprendida (10/10/2026):** fluxo chamado por outro fluxo precisa estar **Publicado**, senão dá "O fluxo de trabalho não está ativo e não pode ser executado". O agente repetiu a chamada até "Número máximo de iterações (10) atingido".
- **Próxima etapa:** publicar `Ferramenta - registrar_pedido`, colar o novo código do nó `Calcular pedido` (aceita itens em texto) e testar o pedido de 2 Dorflex no Pix (esperado: R$ 27,80, PED-0008).
