# 5. Referências e uso de IA

## Material da aula

Board do encontro "Processamento de Streaming de Dados e Eventos" (Prof. Reginaldo Arakaki, 18/09/2026), no Miro. Partes usadas:

- **Plano da aula:** Parte 1, eventos e eventos complexos; Parte 2, eventos complexos como pilares de inovação; Parte 3, projeto Sindusfarm e cenários; dúvidas sobre a ponderada.
- **Cartões conceituais:** "O Contrato Formal de Evento", "Tempo, Ordem e Tolerância", "Simples → Agregado → Complexo".
- **Deck "O Sistema Nervoso Digital":** anatomia da arquitetura, contrato de evento, event streaming, CEP, arquitetura de referência, espectro modelo, sombra e gêmeo digital, MLOps, governança RM-ODP e ATAM.
- **Deck "CEP: A Arquitetura do Agora":** anatomia de um evento, correlação temporal, método em 3 passos (mapeamento, modelagem, decisão), aplicação a finanças e e-commerce, matriz de padrões.
- **Deck "Módulos 2 & 3 — Cards Miro CEP":** ciclo de mapeamento de eventos, requisitos não funcionais mensuráveis (ISO/IEC 25010), batch × CEP, atividade prática de pipeline CEP.
- **Deck "SINDUSFARM Mission Briefing":** instruções dos Cartões de Missão, marcadas no board como template da ponderada; critérios de sucesso (fundamentos e visões, governança, execução, evidência).

## Referências indicadas no board

- PRESSMAN, R. S.; MAXIM, B. R. *Engenharia de Software: uma abordagem profissional*.
- BASS, L.; CLEMENTS, P.; KAZMAN, R. *Software Architecture in Practice*.
- ISO/IEC 25010 — qualidade de produto de software.
- ISO/IEC 12207 — processos de ciclo de vida de software.
- ISO/IEC 10746 — RM-ODP (*Reference Model of Open Distributed Processing*).

## Autoestudos do card

- *Real-Time Stream Processing with Apache Kafka: Design Patterns, Use Cases, and Performance Evaluation.* ResearchGate. https://www.researchgate.net/publication/382933056_REAL-TIME_STREAM_PROCESSING_WITH_APACHE_KAFKA_DESIGN_PATTERNS_USE_CASES_AND_PERFORMANCE_EVALUATION
- STOPFORD, B. *Designing Event-Driven Systems.* Confluent. https://www.confluent.io/resources/ebook/designing-event-driven-systems/
- *Apache Kafka Fundamentals / Event Streaming* (vídeo). https://youtu.be/-RDyEFvnTXI

## Fatos públicos usados sobre o caso

| Fato | Fonte |
|---|---|
| A maior parte das interações críticas entre microsserviços do Nubank passa por mensagens Kafka | Building Nubank, *Microservices at Nubank, an overview*. https://building.nubank.com/microservices-at-nubank-an-overview/ |
| Limite padrão de R$ 1.000 para Pix no período noturno (20h às 6h); aumento leva no mínimo 24 h | Serasa, *Limite Pix: regras…* https://www.serasa.com.br/premium/blog/o-pix-tem-limite-entenda-as-regras-de-transferencias-eletronicas/ |
| Bloqueio cautelar de até 72 h, aplicável só na conta recebedora, por suspeita de fraude (Resolução BCB nº 147/2021) | Blog BB, *Bloqueio cautelar Pix*. https://blog.bb.com.br/bloqueio-cautelar-pix/ · Serasa. https://www.serasa.com.br/blog/bloqueio-cautelar-pix/ |
| MED — Mecanismo Especial de Devolução; MED 2.0 com rastreio em cascata | Banco Central, *Guia de implementação do MED*. https://www.bcb.gov.br/content/estabilidadefinanceira/pix/Guia_MED.pdf · PagBank. https://blog.pagbank.com.br/mecanismo-especial-de-devolucao-pix |

Tudo o que não está nessa tabela (tópicos, regras, limiares, janelas, metas) é **proposta deste trabalho**, não descrição do Nubank.

## Declaração de uso de IA generativa

Usei um assistente de IA (Claude, da Anthropic) como ferramenta de apoio em três frentes:

1. **Leitura do material da aula.** Extração do conteúdo do board do Miro (texto dos cartões e das imagens dos slides) para consulta.
2. **Rascunho.** Primeira versão do catálogo de eventos, dos diagramas PlantUML e dos textos.
3. **Verificação de fatos.** Busca das fontes públicas da tabela acima.

A escolha do caso e dos formatos, a revisão final antes da entrega e a responsabilidade pelo conteúdo são minhas. Os fatos sobre o Pix e o Nubank foram conferidos nas fontes citadas; os limiares e metas estão marcados como hipóteses.
