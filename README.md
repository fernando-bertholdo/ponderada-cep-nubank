# Ponderada — CEP em um app bancário digital (caso Nubank)

**Disciplina:** Computação · Módulo 11 · Engenharia de Software · Inteli, turma 2026-2A-T13
**Encontro:** Processamento de Streaming de Dados e Eventos (T17.ESW7 PLN, 18/09/2026) · Prof. Reginaldo Arakaki
**Autor:** Fernando Bertholdo

## Enunciado

> A ponderada será realizada sobre CEP:
>
> 1. Considere um aplicativo digital bancário (ex. Nubank, BTG, Itaú, Bradesco — marcas de bancos brasileiros). Identificar eventos simples e complexos de negócios e técnicos envolvidos, segundo o conceito de CEP;
> 2. Elaborar uma modelagem estática e dinâmica usando UML destes eventos;
> 3. Criar cenários de negócios — 3 cenários indicando como os eventos podem aprimorar a eficiência das transações. Justificar.

## Como este trabalho está organizado

O board da aula marca o **Cartão de Missão** como template da ponderada: *Cenário & Contexto → 1. Identificar → 2. Modelar → 3. Decidir*. Os documentos seguem essa ordem, e cada parte do enunciado cai em um deles.

| # | Documento | Parte do enunciado | Etapa do Cartão de Missão |
|---|---|---|---|
| 0 | [Cartão de Missão (resumo)](docs/00-cartao-de-missao.md) | visão geral | todas |
| 1 | [Catálogo de eventos](docs/01-eventos.md) | **Parte 1** — eventos simples e complexos, de negócio e técnicos | Cenário & Contexto · Identificar |
| 2 | [Modelagem UML](docs/02-modelagem-uml.md) | **Parte 2** — modelagem estática e dinâmica | Modelar |
| 3 | [Cenários de negócio](docs/03-cenarios-de-negocio.md) | **Parte 3** — 3 cenários com justificativa | Identificar · Decidir |
| 4 | [Decisões de engenharia](docs/04-decisoes-arquiteturais.md) | apoio à Parte 3 | Decidir |
| 5 | [Referências e uso de IA](docs/05-referencias-e-uso-de-ia.md) | — | — |

Os diagramas estão em [`diagramas/src/`](diagramas/src) (PlantUML, fonte versionada) e [`diagramas/png/`](diagramas/png) (renderizados).

## Resumo em cinco linhas

1. Um app bancário gera eventos simples o tempo todo: login, dispositivo novo, alteração de limite, Pix iniciado, compra no cartão, latência do SPI.
2. Isolados, eles dizem pouco. Correlacionados em janelas de tempo, formam **eventos complexos** como *suspeita de golpe por engenharia social*, *teste de cartão* e *degradação do SPI*.
3. O Pix liquida em segundos e não tem estorno simples. Por isso a decisão precisa acontecer **antes** da liquidação, e um processamento em lote chega tarde.
4. A arquitetura proposta usa Kafka como barramento, um motor CEP com janelas e watermarks, e um **gêmeo digital do cliente** que fecha a malha: a decisão realimenta o estado.
5. Os três cenários mostram ganho de eficiência de formas diferentes: menos fricção para o cliente legítimo, mais compras aprovadas corretamente e menos transações duplicadas ou perdidas.

## Aviso sobre o caso

Nubank é usado como **caso ilustrativo**. Não há acesso a dados, regras ou arquitetura interna do banco. Os fatos públicos citados (uso de Kafka entre microsserviços, regras do Pix) têm fonte em [05](docs/05-referencias-e-uso-de-ia.md). Limiares, janelas e metas são **hipóteses de projeto**, marcadas assim no texto.
