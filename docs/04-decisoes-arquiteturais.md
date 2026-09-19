# 4. Decisões de engenharia (etapa "Decidir")

O Cartão de Missão pede, na etapa "Decidir": especificar o gêmeo digital, os ciclos de feedback e avaliar os trade-offs com ATAM e ISO/IEC 25010:2023. Cada decisão abaixo registra contexto, alternativas, escolha e consequência, no formato de um ADR curto.

## Requisitos não funcionais mensuráveis

O material da aula lista quatro requisitos que todo pipeline CEP precisa declarar. Metas são **hipóteses de projeto**.

| Característica ISO/IEC 25010:2023 | Métrica | Meta | Onde é medida |
|---|---|---|---|
| Eficiência de desempenho — comportamento no tempo | latência de decisão, do `t_ocorrencia` do gatilho à ação | p99 ≤ 150 ms no caminho síncrono | diferença entre timestamps do evento e da decisão |
| Eficiência de desempenho — capacidade | vazão sustentada sem degradação | 3× o pico histórico de eventos por segundo | teste de carga (k6 ou JMeter) |
| Confiabilidade — tolerância a falhas | taxa de perda de eventos | 0 eventos perdidos; ≤ 0,1% atrasados além do watermark | contagem de `id` na entrada × na saída |
| Adequação funcional — correção | acurácia do padrão | precisão ≥ 0,8 e revocação ≥ 0,9 no C01, medidas contra casos rotulados | rotulagem pela mesa de fraude |
| Segurança — confidencialidade | dado pessoal em claro nos tópicos | 0 campos identificadores sem hash ou pseudônimo | varredura do schema no CI |

---

### D1. Apache Kafka como barramento de eventos

- **Contexto.** Dezenas de produtores e consumidores, necessidade de reprocessar histórico e ordem por cliente.
- **Alternativas.** Fila tradicional (RabbitMQ); serviço gerenciado de streaming de nuvem; Kafka.
- **Decisão.** Kafka, com tópicos por domínio e partição por `id_cliente`.
- **Por quê.** O log é imutável e relegível: um consumidor novo, ou uma regra nova, pode ser testado reprocessando dias de eventos reais. A fila tradicional apaga a mensagem depois de consumida. A escolha também é coerente com o caso: o Nubank descreve publicamente que a maior parte da comunicação crítica entre seus microsserviços passa por Kafka ([05](05-referencias-e-uso-de-ia.md)).
- **Trade-off.** Ordem só existe dentro da partição. Um padrão que cruze clientes diferentes (C03, conta de passagem) precisa de um segundo particionamento, por conta recebedora, e aceita ordem aproximada.

### D2. Apache Flink como motor CEP

- **Contexto.** Janelas de três tipos, padrões com sequência e tempo, estado grande por cliente, garantia de processamento.
- **Alternativas.** Esper ou Siddhi (motores CEP dedicados, citados na aula); Kafka Streams; Flink.
- **Decisão.** Flink, com a biblioteca de CEP e `MATCH_RECOGNIZE` em SQL para os padrões.
- **Por quê.** Suporta event-time e watermarks nativamente, mantém estado por chave com checkpoint e oferece processamento *exactly-once* integrado ao Kafka. Esper e Siddhi têm linguagem de padrões mais expressiva, mas escalam pior com estado de milhões de clientes.
- **Trade-off.** Mais complexidade operacional que Kafka Streams. Regra escrita em SQL de padrão é menos legível para a mesa de fraude, por isso D6.

### D3. Event-time com watermark e atraso tolerado de 5 s

- **Contexto.** O app está em rede móvel. Um `LimiteAlterado` pode chegar depois do `PixIniciado` que ele precede.
- **Alternativas.** Processar pela ordem de chegada (*ingestion-time*); esperar indefinidamente; event-time com watermark.
- **Decisão.** Event-time, watermark com atraso tolerado de 5 s *(hipótese)*. Evento mais atrasado vira `EventoAtrasado` e pode gerar correção, mas não reabre a decisão.
- **Trade-off.** É o conflito clássico **latência × completude**. Esperar mais pega mais padrões e atrasa a decisão. Como o caminho síncrono tem orçamento de 150 ms, o atraso tolerado só vale para os eventos **anteriores** ao gatilho, que normalmente já chegaram.

### D4. Duas velocidades de decisão e fail-open controlado

- **Contexto.** Algumas decisões precisam acontecer antes da transação (C01, C04, C05, C06); outras podem vir depois (C03, C07).
- **Decisão.** Dois caminhos. **Síncrono**: o core Pix e o autorizador esperam a resposta do orquestrador, com timeout. **Assíncrono**: os demais consumidores leem `cep.eventos-complexos.v1` no próprio ritmo. Se o caminho síncrono estourar o timeout, a transação segue com regras estáticas e dentro dos limites (*fail-open* controlado).
- **Trade-off (ATAM).** Ponto de sensibilidade entre **disponibilidade** e **segurança**. *Fail-closed* protege mais e derruba o Pix de todos quando o antifraude cai. *Fail-open* sem limite deixa a porta aberta. O meio-termo aceita um risco limitado por valor durante a falha e registra cada decisão tomada em modo degradado para revisão.

### D5. Gêmeo digital do cliente em malha fechada

- **Contexto.** "Valor alto", "dispositivo novo" e "horário estranho" só fazem sentido em relação ao histórico de cada cliente.
- **Decisão.** Manter, por cliente, um estado atualizado a cada evento: dispositivos confiáveis, favorecidos habituais, p95 de valor, horários típicos, estado de risco. As regras consultam esse estado, e o resultado de cada decisão (o cliente confirmou, cancelou, contestou) realimenta o estado.
- **Por quê é gêmeo e não sombra.** Pelo espectro visto em aula: o *modelo digital* não tem fluxo automático; a *sombra digital* é atualizada em um sentido só; o *gêmeo digital* tem vínculo bidirecional, e a decisão atua sobre o mundo real. Aqui a ação (desafiar, bloquear, ajustar limite) muda o comportamento do cliente, e esse comportamento volta como evento.
- **Trade-off.** O estado é dado pessoal sensível. Retenção limitada, pseudonimização e acesso restrito são obrigatórios. E o gêmeo envelhece: o perfil de um cliente muda (mudou de cidade, trocou de celular), o que exige monitorar *concept drift*, como a aula aponta no ciclo MLOps.

### D6. Contrato versionado e regras ECA como dado, não como código

- **Contexto.** Muitos produtores evoluindo em ritmos diferentes; limiares que a mesa de fraude precisa ajustar sem deploy.
- **Decisão.** Todo tópico tem schema no Schema Registry, com compatibilidade retroativa obrigatória; evento inválido vai para quarentena. As regras ECA ficam num repositório versionado, com limiares como parâmetros, e cada evento complexo registra a versão da regra que o gerou.
- **Trade-off.** Mais governança na mudança de schema. Em troca, um produtor não quebra o motor, e toda decisão pode ser explicada: qual regra, qual versão, quais evidências.

---

## Riscos e pontos de sensibilidade (resumo ATAM)

| Ponto | Atributos em tensão | Tratamento |
|---|---|---|
| Timeout do caminho síncrono | disponibilidade × segurança | D4, fail-open limitado por valor |
| Tamanho do atraso tolerado | latência × completude | D3, só para eventos anteriores ao gatilho |
| Sensibilidade da regra C01 | fricção × proteção | limiar ajustável (D6) e medido por precisão e revocação |
| Estado do gêmeo | personalização × privacidade | D5, pseudonimização e retenção limitada |
| Partição por cliente | ordem × padrões entre clientes | D1, segundo particionamento para C03 |
