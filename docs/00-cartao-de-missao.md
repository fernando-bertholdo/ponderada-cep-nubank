# 0. Cartão de Missão — Nubank

Preenchimento do template indicado no board da aula ("Instruções Operacionais: Como ler os seus Cartões de Missão"). Cada campo aponta para o documento que o detalha.

## Cenário & Contexto (onde e por quê)

**Domínio:** banco digital que opera só pelo app: conta, Pix, cartão de crédito e débito, limites e crédito.

**Onde o tempo real agrega valor:**

- **Pix.** A liquidação acontece em segundos, 24 horas por dia, e desfazer uma transferência exige o Mecanismo Especial de Devolução (MED), que é lento e nem sempre recupera o dinheiro. A proteção precisa agir **antes** do envio.
- **Autorização de cartão.** A bandeira espera a resposta em uma janela curta. Uma recusa errada custa a venda e a confiança do cliente; uma aprovação errada custa a fraude.
- **Operação.** Se a integração com o SPI degrada, o cliente vê erro, tenta de novo e pode gerar pagamento em duplicidade.

Nos três casos, um relatório do dia seguinte chega tarde demais para agir.

## 1. Identificar (negócio e sensores)

Mapear o contrato do evento. Diferenciar telemetria simples da regra ECA (*Event-Condition-Action*) do evento complexo.

- Contrato formal: `evento = (id, tipo, versão, entidade, origem, t_ocorrência, t_ingestão, atributos, qualidade, correlação)`.
- **24 eventos simples** (13 de negócio, 11 técnicos), **5 agregados** e **9 complexos**, com regra ECA para cada complexo.
- Detalhe em [01-eventos.md](01-eventos.md).

## 2. Modelar (arquitetura)

Desenhar tópicos no Kafka (produtores e consumidores) e as visões RM-ODP.

- Estática: diagrama de classes do modelo de eventos e diagrama de componentes com os tópicos.
- Dinâmica: três diagramas de sequência (um por cenário), máquina de estados da transação Pix e diagrama de atividade do pipeline CEP.
- As cinco visões RM-ODP resumidas em uma tabela.
- Detalhe em [02-modelagem-uml.md](02-modelagem-uml.md).

## 3. Decidir (engenharia e gêmeo digital)

Especificar o gêmeo digital, os ciclos de feedback e avaliar os trade-offs com ATAM (ISO/IEC 25010:2023).

- Gêmeo digital do cliente: perfil comportamental atualizado por eventos e realimentado pelas decisões.
- Seis decisões registradas, cada uma com alternativas e trade-off.
- Requisitos não funcionais mensuráveis: latência (percentis), vazão, perda e acurácia.
- Três cenários de negócio com estímulo, resposta e medida.
- Detalhe em [03-cenarios-de-negocio.md](03-cenarios-de-negocio.md) e [04-decisoes-arquiteturais.md](04-decisoes-arquiteturais.md).
