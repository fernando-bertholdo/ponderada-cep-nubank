# 3. Cenários de negócio (Parte 3 do enunciado)

> Criar cenários de negócios — 3 cenários indicando como os eventos podem aprimorar a eficiência das transações. Justificar.

## O que "eficiência da transação" quer dizer aqui

Uma transação bancária é eficiente quando chega ao resultado certo com o menor custo para todos os lados. Uso quatro medidas, e cada cenário mexe em pelo menos duas:

| Medida | O que desperdiça quando falha |
|---|---|
| **Correção** — a transação legítima passa e a fraudulenta não | dinheiro perdido, ou venda perdida |
| **Fricção** — quantos passos extras o cliente legítimo enfrenta | tempo e abandono |
| **Retrabalho** — contestação, devolução, chamado, pagamento duplicado | custo operacional depois do fato |
| **Latência de decisão** — tempo entre o fato e a ação | a janela de agir passa |

O argumento central, retomado do slide "Batch vs. CEP" da aula: no modelo em lote a decisão vem depois, em relatório. No modelo CEP a ação acontece em malha fechada, enquanto a transação ainda pode ser mudada.

Os cenários usam o formato de **cenário de qualidade do ATAM** (fonte, estímulo, ambiente, artefato, resposta e medida), como pede o material da aula. **Todos os números de meta são hipóteses de projeto**, não dados do Nubank.

---

## Cenário 1 — Pix protegido contra golpe, sem atrito para o cliente legítimo

**Contexto.** No golpe por engenharia social, o próprio cliente é convencido a transferir: um falso atendente liga, pede para instalar o app em outro aparelho ou aumentar o limite "para proteger a conta" e dita a chave de destino. Todas as senhas estão certas, porque é o cliente quem digita. Uma regra que olha só para o Pix não vê nada de errado.

**Eventos envolvidos:** N02 `DispositivoRegistrado(novo)`, N05 `LimiteAlterado`, N06 `FavorecidoAdicionado`, N07 `PixIniciado`, T01 `TelemetriaSessao(colou_texto)`, T05 `ChamadaAtivaDuranteSessao` → C01 `SuspeitaGolpeEngenhariaSocial`.
Diagramas: [sequência](02-modelagem-uml.md#23-sequências) e [estados](02-modelagem-uml.md#24-máquina-de-estados-da-transação-pix).

| | Sem CEP | Com CEP |
|---|---|---|
| Quando se descobre | depois, quando o cliente reclama ou o MED é acionado | durante a sessão, antes do envio ao SPI |
| Quem recebe fricção | todo mundo acima de um valor fixo, ou ninguém | só a sessão que casou o padrão |
| Retrabalho | contestação, MED, atendimento, eventual ressarcimento | cliente que desiste ao ver o alerta não gera nada disso |

**Como os eventos aumentam a eficiência:**

- **Correção.** A correlação de quatro sinais de fontes diferentes separa o golpe do cliente que só está fazendo um Pix alto. Um Pix de R$ 4.800 para o aluguel, do aparelho de sempre, para uma chave já usada, passa direto.
- **Fricção seletiva.** O desafio aparece em poucas sessões, não em todas as transações acima de um valor. O cliente legítimo não paga pelo golpista.
- **Retrabalho evitado.** Cada golpe interrompido antes da liquidação é um MED, um chamado e uma contestação que não existem.

**Cenário de qualidade (ATAM):**

| Elemento | Valor |
|---|---|
| Fonte | cliente sob orientação de golpista |
| Estímulo | `PixIniciado` que completa o padrão C01 |
| Ambiente | operação normal, pico noturno |
| Artefato | motor CEP + orquestrador de decisão |
| Resposta | desafio de autenticação com alerta contextual, antes do envio ao SPI |
| Medida | decisão em até 150 ms no p99; ≥ 90% dos golpes do padrão desafiados antes do envio; ≤ 1% das sessões legítimas desafiadas *(metas hipotéticas)* |

**Justificativa.** O Pix liquida em segundos, e a devolução depende do MED, um processo posterior que nem sempre recupera o valor. A única janela em que prevenir custa pouco é a sessão, antes do envio. O banco do pagador não pode aplicar bloqueio cautelar no próprio cliente (a regra permite o bloqueio só na conta **recebedora**), então a ação possível é o **desafio**, e ela tem que chegar a tempo. Só um processamento que correlaciona eventos em tempo real alcança essa janela.

**Extensão do mesmo cenário, lado recebedor.** Quando o Nubank é a instituição **recebedora**, a regra C03 `ContaDePassagem` detecta a conta que recebe de várias origens e repassa quase tudo em minutos. Aí o bloqueio cautelar de até 72 h é permitido e segura o dinheiro antes do próximo salto, o que também facilita o rastreio em cascata do MED 2.0.

**Limite honesto.** O sinal T05 (ligação ativa) depende do que o sistema operacional expõe, e pode não estar disponível. O padrão foi desenhado para funcionar sem ele, com os outros quatro sinais.

---

## Cenário 2 — Autorização de cartão que usa o contexto do cliente

**Contexto.** O autorizador precisa responder à bandeira em pouco tempo. Uma regra estática ("compra fora da cidade de residência é suspeita") recusa o cliente que está viajando. Ao mesmo tempo, fraudadores testam cartões vazados com várias compras online de centavos antes da compra grande.

**Eventos envolvidos:** N01 `LoginRealizado`, T03 `LocalizacaoAproximada`, N11 `CompraCartaoSolicitada`, N12 `CompraCartaoNegada`, agregado A03 → C04 `TesteDeCartao`, C05 `ImpossibilidadeGeografica`, C06 `ContextoViagemConfirmado`, C07 `OportunidadeAjusteLimite`.
Diagrama: [sequência](02-modelagem-uml.md#23-sequências).

| | Sem CEP | Com CEP |
|---|---|---|
| Cliente viajando | recusa por localização; liga para a central | o app aberto na mesma cidade confirma o contexto, e a compra passa |
| Teste de cartão | as compras de centavos passam; a fraude grande vem depois | a 5ª tentativa em 10 min bloqueia o cartão virtual antes da compra grande |
| Negada por limite | venda perdida | oferta de ajuste em um toque; a venda acontece |

**Como os eventos aumentam a eficiência:**

- **Correção nos dois sentidos.** O mesmo mecanismo que aprova o viajante (C06) bloqueia o clone (C05). O ganho vem de usar os eventos do app como evidência a favor do cliente, não só contra.
- **Menos retrabalho.** Cada falsa recusa evitada é uma ligação para a central a menos e uma compra que não precisa ser refeita.
- **Receita recuperada.** C07 transforma uma recusa por limite em oferta. É um evento complexo "positivo": o padrão detectado é uma oportunidade, não um risco.

**Cenário de qualidade (ATAM):**

| Elemento | Valor |
|---|---|
| Fonte | bandeira / adquirente |
| Estímulo | pedido de autorização presencial em cidade fora do padrão do cliente |
| Ambiente | cliente com sessão ativa no app nas últimas 6 h |
| Artefato | autorizador + motor CEP + gêmeo digital |
| Resposta | aprovar sem desafio quando C06 casar; negar e bloquear quando C04 ou C05 casar |
| Medida | resposta dentro do prazo da bandeira em 99,9% dos pedidos; redução de 30% nas falsas recusas por localização em relação à regra estática *(metas hipotéticas)* |

**Justificativa.** Uma autorização de cartão só pode ser decidida enquanto a bandeira espera. Depois disso, a venda já foi aprovada ou perdida. O contexto que evita a recusa errada (o cliente abriu o app naquela cidade uma hora antes) já existe como evento: basta correlacioná-lo a tempo. O teste de cartão, por sua vez, só é visível como padrão de várias tentativas numa janela; cada tentativa isolada parece inofensiva.

---

## Cenário 3 — Transação que não se perde nem se duplica quando o SPI degrada

**Contexto.** A conexão com o SPI (sistema do Banco Central que liquida o Pix) pode ficar lenta ou devolver erros por alguns minutos. O cliente vê um erro genérico, tenta de novo e às vezes o primeiro Pix também é liquidado. Resultado: pagamento em duplicidade, chamado e pedido de devolução.

**Eventos envolvidos:** T06 `LatenciaSPIMedida`, T07 `ErroIntegracaoSPI`, T09 `LagConsumidor`, T11 `OrdemReenviada`, N07 `PixIniciado`, agregado A04 → C08 `DegradacaoSPIDetectada`, C09 `RiscoDePixDuplicado`.
Diagramas: [sequência](02-modelagem-uml.md#23-sequências) e [estados](02-modelagem-uml.md#24-máquina-de-estados-da-transação-pix).

| | Sem CEP | Com CEP |
|---|---|---|
| Detecção | alarme de erro isolado, muitas vezes ruído | padrão sustentado: erro + latência + fila por 3 janelas |
| Comportamento do produto | erro genérico; o cliente repete | "Pix em processamento"; retentativa automática com a mesma chave |
| Segundo Pix igual | vira outro pagamento | C09 segura e pergunta se o cliente quer mesmo repetir |

**Como os eventos aumentam a eficiência:**

- **Nenhuma transação duplicada.** A retentativa reutiliza a `idempotency_key`, e o C09 intercepta a repetição manual.
- **Nenhuma transação perdida.** O Pix fica em `EmRetentativa` em vez de falhar; quando o SPI normaliza, segue.
- **Menos atendimento.** O aviso proativo diz ao cliente o que está acontecendo antes que ele ligue.
- **Alarme com contexto.** SRE recebe um evento complexo com as evidências, não dezenas de alertas soltos.

**Cenário de qualidade (ATAM):**

| Elemento | Valor |
|---|---|
| Fonte | infraestrutura externa (SPI) |
| Estímulo | taxa de erro > 2% e p99 > 3 s por 3 janelas de 60 s |
| Ambiente | horário de pico |
| Artefato | gateway SPI + core Pix + motor CEP |
| Resposta | ativar modo degradado, retentativa idempotente, aviso ao cliente |
| Medida | 0 Pix duplicados por retentativa; detecção em até 3 min do início da degradação; ≥ 95% dos Pix em retentativa liquidados após a normalização *(metas hipotéticas)* |

**Justificativa.** Esse cenário mostra que CEP não serve só para fraude. Eventos técnicos, correlacionados, mudam o comportamento do produto no momento em que o problema acontece. Um único erro de SPI não quer dizer nada; um padrão sustentado de erro, latência e fila quer. A eficiência aqui é operacional: menos transação repetida, menos devolução, menos chamado.

---

## Comparação dos três

| | Cenário 1 — Pix | Cenário 2 — Cartão | Cenário 3 — SPI |
|---|---|---|---|
| Natureza dos eventos | negócio + técnico | negócio + técnico | técnico → negócio |
| Principal ganho | menos golpe, menos fricção | menos recusa errada, mais venda | zero duplicidade, menos atendimento |
| Janela de decisão | antes do envio ao SPI | enquanto a bandeira espera | durante a degradação |
| O que o lote faria | detecção depois do prejuízo | relatório de recusas no dia seguinte | análise do incidente depois dele |
