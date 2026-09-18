# 1. Catálogo de eventos (Parte 1 do enunciado)

> Identificar eventos simples e complexos de negócios e técnicos envolvidos, segundo o conceito de CEP.

## 1.1 Conceitos usados

Os três níveis vêm do material da aula (board do encontro, cartões "Simples → Agregado → Complexo" e "A Anatomia de um Evento"):

| Nível | Definição | Forma | Exemplo no banco |
|---|---|---|---|
| **1. Simples** | Mudança de estado isolada, indivisível | `e = (id, tipo, t_ocorr, v)` | `PixIniciado` de R$ 4.800 |
| **2. Agregado** | Sumarização de eventos simples em uma janela de tempo | `e = f({e_s \| t ∈ Δt})` | soma dos Pix enviados pelo cliente na última hora |
| **3. Complexo** | Padrão inferido a partir de **várias fontes distintas**, com correlação causal, temporal ou espacial | `e = CEP(e₁, e₂, …, eₙ; Δt)` | dispositivo novo + aumento de limite + favorecido novo + Pix alto em 30 min = suspeita de golpe |

**Negócio × técnico.** Evento de negócio é o que o cliente ou o produto reconhece (um Pix, uma compra). Evento técnico é o que a plataforma observa sobre si mesma ou sobre o canal (latência, erro, integridade do aparelho). O CEP cruza as duas naturezas: vários dos padrões mais úteis só aparecem quando um sinal técnico explica um fato de negócio.

**Regra ECA** (*Event-Condition-Action*): **ao** chegar o evento gatilho, **se** a condição sobre a janela for verdadeira, **então** executar a ação. É a forma como cada evento complexo abaixo é especificado.

## 1.2 Contrato formal do evento

Todo evento, de qualquer nível, segue o contrato apresentado em aula:

```
evento = (id, tipo, versão, entidade, origem, t_ocorrência, t_ingestão, atributos, qualidade, correlação)
```

Exemplo de `PixIniciado` serializado:

```json
{
  "id": "8f1c2a7e-4b0d-4f7e-9a51-3c2d7e11b0a9",
  "tipo": "pix.iniciado",
  "versao": "1.2.0",
  "entidade": { "tipo": "cliente", "id": "cli_00042" },
  "origem": "core-pix",
  "t_ocorrencia": "2026-09-18T21:47:03.120-03:00",
  "t_ingestao":   "2026-09-18T21:47:03.410-03:00",
  "atributos": {
    "valor": { "centavos": 480000, "moeda": "BRL" },
    "chave_destino_hash": "sha256:9b…",
    "tipo_chave": "aleatoria",
    "idempotency_key": "pix-cli_00042-20260918-000731",
    "canal": "app-android",
    "id_dispositivo": "dev_7731"
  },
  "qualidade": { "confianca": 1.0, "completo": true, "atrasado": false },
  "correlacao": "sess_5e02…"
}
```

Três campos fazem o CEP funcionar:

- **`t_ocorrencia` × `t_ingestao`.** O app pode enviar o evento com atraso por causa da rede móvel. As janelas usam o tempo de **ocorrência** (event-time), e a diferença entre os dois alimenta a métrica de atraso.
- **`correlacao`.** Liga os eventos da mesma sessão. Sem ela, o motor não sabe que o aumento de limite e o Pix saíram da mesma sequência de telas.
- **`idempotency_key`.** Garante que um reenvio não vire um segundo Pix (cenário 3).

Dados pessoais não trafegam em claro: a chave do destinatário vai como hash, e o cliente como identificador interno. *(Decisão de projeto, alinhada à LGPD.)*

## 1.3 Eventos simples de negócio (13)

| ID | Evento | Origem | Atributos principais |
|---|---|---|---|
| N01 | `LoginRealizado` | app / autenticação | dispositivo, localização aproximada, método (senha, biometria) |
| N02 | `DispositivoRegistrado` | autenticação | id do aparelho, `novo` (nunca visto para o cliente) |
| N03 | `DadosCadastraisAlterados` | serviço de conta | campo alterado (e-mail, telefone), canal |
| N04 | `SenhaAlterada` | autenticação | canal, se houve recuperação por e-mail |
| N05 | `LimiteAlterado` | serviço de conta | produto (Pix noturno, cartão), valor de → para |
| N06 | `FavorecidoAdicionado` | core Pix | hash da chave, se já houve Pix para ela |
| N07 | `PixIniciado` | core Pix | valor, chave destino, tipo de chave, `idempotency_key` |
| N08 | `PixLiquidado` | gateway SPI | id fim a fim, tempo até liquidar |
| N09 | `PixRecebido` | gateway SPI | valor, instituição de origem, conta recebedora |
| N10 | `PixCancelado` | core Pix | motivo (cliente desistiu, desafio não respondido) |
| N11 | `CompraCartaoSolicitada` | autorizador | valor, MCC, presencial ou online, localização do terminal |
| N12 | `CompraCartaoNegada` | autorizador | motivo (limite, suspeita, cartão bloqueado) |
| N13 | `DevolucaoMEDSolicitada` | autoatendimento / SPI | transação contestada, motivo |

## 1.4 Eventos simples técnicos (11)

| ID | Evento | Origem | O que observa |
|---|---|---|---|
| T01 | `TelemetriaSessao` | SDK do app | ritmo de digitação, texto colado no campo de valor ou chave, tempo em cada tela |
| T02 | `IntegridadeDispositivo` | SDK do app | root/jailbreak, emulador, app adulterado |
| T03 | `LocalizacaoAproximada` | app / IP | cidade provável da sessão |
| T04 | `FalhaAutenticacao` | autenticação | senha ou biometria recusada |
| T05 | `ChamadaAtivaDuranteSessao` | SDK do app | indicador de ligação em curso enquanto o cliente opera o app *(hipótese: depende do que o sistema operacional expõe)* |
| T06 | `LatenciaSPIMedida` | gateway SPI | tempo de ida e volta de cada ordem |
| T07 | `ErroIntegracaoSPI` | gateway SPI | código de erro ou timeout |
| T08 | `TimeoutAutorizador` | autorizador | resposta à bandeira fora do prazo |
| T09 | `LagConsumidor` | observabilidade | atraso de um grupo de consumidores Kafka |
| T10 | `EventoQuarentenado` | validação de contrato | evento que falhou no schema e foi para a fila de quarentena |
| T11 | `OrdemReenviada` | core Pix | retentativa de envio ao SPI com a mesma `idempotency_key` |

## 1.5 Eventos agregados (5)

| ID | Agregado | Chave | Janela | Função |
|---|---|---|---|---|
| A01 | Valor de Pix enviado | cliente | deslizante de 1 h, passo de 5 min | soma e contagem de N07 |
| A02 | Falhas de autenticação | cliente | fixa (*tumbling*) de 10 min | contagem de T04 |
| A03 | Compras online de baixo valor | cartão | deslizante de 10 min | contagem de N11 com valor < R$ 5 e lojistas distintos |
| A04 | Saúde do SPI | instituição | deslizante de 60 s, passo de 10 s | taxa de T07, p99 de T06, tamanho da fila |
| A05 | Movimentação da conta recebedora | conta | sessão (fecha após 30 min sem evento) | entradas (N09) e saídas (N07) e tempo entre elas |

Os tamanhos de janela são **hipóteses de projeto**, a calibrar com dados reais.

## 1.6 Eventos complexos (9) e suas regras ECA

Limiares e janelas são **hipóteses de projeto**.

### De negócio — proteção

**C01 · `SuspeitaGolpeEngenhariaSocial`** (cenário 1)
- **Ao** `PixIniciado`
- **Se**, para o mesmo cliente, em até 30 min: `DispositivoRegistrado(novo)` **e** `LimiteAlterado(aumento)` **e** `FavorecidoAdicionado` para a chave do Pix **e** valor > p95 histórico do cliente (do gêmeo digital); reforçado por `ChamadaAtivaDuranteSessao` ou `TelemetriaSessao(colou_texto)`
- **Então** `DESAFIAR_AUTENTICACAO` com alerta contextual de golpe, antes do envio ao SPI

**C02 · `TomadaDeConta`** (*account takeover*)
- **Ao** `DadosCadastraisAlterados(e-mail ou telefone)`
- **Se**, em uma sessão: `SenhaAlterada` por recuperação **e** login a partir de `DispositivoRegistrado(novo)` **e** `IntegridadeDispositivo(emulador ou root)`
- **Então** suspender operações de saída, `NOTIFICAR_CLIENTE` pelos canais antigos, `ALERTAR_ANALISTA`

**C03 · `ContaDePassagem`** (conta "laranja", lado recebedor)
- **Ao** `PixIniciado` de saída a partir de uma conta que recebeu Pix recentemente
- **Se**, na janela de sessão A05: ≥ 5 Pix recebidos de origens distintas **e** ≥ 80% do valor repassado em menos de 10 min **e** conta aberta há menos de 30 dias
- **Então** `BLOQUEIO_CAUTELAR` dos valores recebidos (permitido pela regulação do Pix por até 72 h, apenas na conta **recebedora**) e `ALERTAR_ANALISTA`

**C04 · `TesteDeCartao`** (cenário 2)
- **Ao** `CompraCartaoSolicitada(online)`
- **Se** A03 ≥ 5 em 10 min, em lojistas distintos
- **Então** `CANCELAR`, bloquear o cartão virtual, `NOTIFICAR_CLIENTE` com opção de gerar um novo

**C05 · `ImpossibilidadeGeografica`** (cartão clonado)
- **Ao** `CompraCartaoSolicitada(presencial)`
- **Se** houve compra presencial com o mesmo cartão em cidade cuja distância exige deslocamento maior que o tempo decorrido, **e** não há sessão do app coerente com nenhuma das duas
- **Então** `CANCELAR` e `ALERTAR_ANALISTA`

### De negócio — eficiência (eventos "positivos")

**C06 · `ContextoViagemConfirmado`** (cenário 2)
- **Ao** `CompraCartaoSolicitada(presencial)` fora da cidade habitual
- **Se** `LoginRealizado` e `LocalizacaoAproximada` do app na mesma cidade nas últimas 6 h, a partir de dispositivo confiável
- **Então** `APROVAR` sem desafio adicional

**C07 · `OportunidadeAjusteLimite`** (cenário 2)
- **Ao** `CompraCartaoNegada(motivo = LIMITE)`
- **Se** cliente sem atraso, dentro da política de crédito, e a compra coerente com o perfil
- **Então** `NOTIFICAR_CLIENTE` com oferta de ajuste de limite em um toque

### Técnicos

**C08 · `DegradacaoSPIDetectada`** (cenário 3)
- **Ao** fechamento de cada janela de A04
- **Se** taxa de `ErroIntegracaoSPI` > 2% **e** p99 de `LatenciaSPIMedida` > 3 s **e** fila crescendo, por 3 janelas seguidas
- **Então** `ATIVAR_MODO_DEGRADADO` (retentativa com a mesma chave, aviso ao cliente) e `ALERTAR_ANALISTA` de SRE; ao normalizar, emitir `DegradacaoSPIEncerrada`

**C09 · `RiscoDePixDuplicado`** (cenário 3)
- **Ao** `PixIniciado`
- **Se** existe outro `PixIniciado` do mesmo cliente, mesma chave e mesmo valor, nos últimos 60 s, com `idempotency_key` diferente, **e** o primeiro está em retentativa
- **Então** segurar o segundo e perguntar ao cliente se quer mesmo repetir

## 1.7 Mapa de correlação

Quais eventos simples alimentam quais complexos:

| Evento simples | C01 | C02 | C03 | C04 | C05 | C06 | C07 | C08 | C09 |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| N01 LoginRealizado | | ● | | | ● | ● | | | |
| N02 DispositivoRegistrado | ● | ● | | | | ● | | | |
| N03 DadosCadastraisAlterados | | ● | | | | | | | |
| N04 SenhaAlterada | | ● | | | | | | | |
| N05 LimiteAlterado | ● | | | | | | | | |
| N06 FavorecidoAdicionado | ● | | | | | | | | |
| N07 PixIniciado | ● | | ● | | | | | | ● |
| N09 PixRecebido | | | ● | | | | | | |
| N11 CompraCartaoSolicitada | | | | ● | ● | ● | | | |
| N12 CompraCartaoNegada | | | | | | | ● | | |
| T01 TelemetriaSessao | ● | | | | | | | | |
| T02 IntegridadeDispositivo | | ● | | | | | | | |
| T03 LocalizacaoAproximada | | | | | ● | ● | | | |
| T05 ChamadaAtivaDuranteSessao | ● | | | | | | | | |
| T06 LatenciaSPIMedida | | | | | | | | ● | |
| T07 ErroIntegracaoSPI | | | | | | | | ● | ● |
| T11 OrdemReenviada | | | | | | | | | ● |

O padrão que a aula enfatiza aparece na tabela: **nenhum evento complexo é um evento simples isolado**. Cada um depende de várias ocorrências na janela (C04, C08), de várias fontes (C01, C02, C05, C06, C09) ou do estado acumulado no gêmeo digital (C03, C07). Cinco dos nove cruzam eventos de negócio com eventos técnicos.

Os eventos que não aparecem na tabela (N08, N10, N13, T08, T09, T10) não disparam padrões: servem para fechar o ciclo de estados da transação ([diagrama de estados](02-modelagem-uml.md#24-máquina-de-estados-da-transação-pix)) e para medir a própria plataforma.
