# Fluxo `Entregas das 08:00`

Fluxo agendado (não é ferramenta do atendente). Arquivo para importar: `entregas_8h.json`.
Nome no n8n: **Entregas das 08:00**. Precisa estar **Publicado** para rodar sozinho.

**Fuso horário:** nas Configurações do fluxo, o fuso tem de ser `America/Sao_Paulo` (horário de Brasília);
senão o "08:00" do agendamento pode ser em outro fuso.

## O que faz
Todo dia às 08:00: lê Configuração, Entregas e Atendentes; pega as entregas com status "Pendente" e **sem motoboy**
(pedidos feitos fora do horário), coloca o motoboy do turno atual (aba Atendentes) e acrescenta na observação
"Liberada em dd/MM HH:mm para <motoboy>". Se rodar fora do horário de funcionamento, não faz nada.

Ainda não avisa o motoboy no celular (depende do WhatsApp).
