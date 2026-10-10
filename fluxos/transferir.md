# Ferramentas `transferir_farmaceutico` e `transferir_atendente`

As duas ferramentas chamam o **mesmo** fluxo: **Ferramenta - transferir** (arquivo `transferir.json`, publicado).
O destino é fixo em cada ferramenta, a IA não escolhe. Não renomear os nós (o código procura os nós pelo nome).

## O que o fluxo faz
1. Lê Atendimentos, Receitas e Atendentes.
2. Grava em Atendimentos o próximo `ATD-0000` com assunto, resumo, `transferido_para` (Farmacêutico ou Atendente) e status "Transferido".
3. Se o destino é Farmacêutico e veio `tipo_receita` (Comum, Controlada ou Antimicrobiano; aceita "controlado", "antibiótico"),
   grava em Receitas o próximo `R0000` com status "Aguardando validação", sem pedido, e o farmacêutico do turno.
4. Responde para a IA avisar o cliente que uma pessoa vai continuar o atendimento.

Ainda não avisa a pessoa no celular e o `link_foto` fica "(foto enviada na conversa)": as duas coisas vêm com o WhatsApp.

**Decisão do Jildean (opção A):** depois de transferir, o robô fica quieto com aquele cliente enquanto houver atendimento
"Transferido"; volta a responder quando a pessoa marcar "Resolvido". Implementar na etapa do WhatsApp.

## Ligação no Agente de IA (duas vezes "Chame a ferramenta de fluxo de trabalho n8n", fluxo `Ferramenta - transferir`)

### `transferir_farmaceutico`
Descrição:
```
Transfere a conversa para o farmacêutico. Use para toda receita (depois de o cliente enviar a foto), todo medicamento controlado e qualquer dúvida de saúde: dose, interação, efeito colateral ou indicação de remédio. Informe o assunto, um resumo curto do que o cliente quer (produtos e quantidades, se houver) e o tipo de receita, se houver.
```
| Campo | Como preencher | Descrição para a IA |
|---|---|---|
| telefone | Expressão: `{{ $('Dados da conversa').first().json.telefone }}` | |
| destino | Fixo: `Farmacêutico` | |
| assunto | ✨ | Assunto em poucas palavras, por exemplo: Receita, Dúvida de dose, Interação. |
| resumo | ✨ | Resumo curto para o farmacêutico: o que o cliente quer, produtos e quantidades. |
| tipo_receita | ✨ | Se houver receita: Comum, Controlada (medicamento controlado) ou Antimicrobiano (antibiótico). Vazio se não houver receita. |
| link_foto | vazio (fica para o WhatsApp) | |

### `transferir_atendente`
Descrição:
```
Transfere a conversa para um atendente da farmácia. Use para reclamações, problemas com pedido já feito, cancelamentos, trocas, erros, pedidos de desconto, cliente insatisfeito ou que peça para falar com uma pessoa, e quando uma ferramenta falhar. Não use para assuntos de saúde ou receitas.
```
| Campo | Como preencher | Descrição para a IA |
|---|---|---|
| telefone | Expressão: `{{ $('Dados da conversa').first().json.telefone }}` | |
| destino | Fixo: `Atendente` | |
| assunto | ✨ | Assunto em poucas palavras, por exemplo: Reclamação, Cancelamento, Troca. |
| resumo | ✨ | Resumo curto para o atendente: o que aconteceu e o que o cliente quer, com o número do pedido se houver. |
| tipo_receita | vazio | |
| link_foto | vazio | |
