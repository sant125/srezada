# Capítulo 5 · Alerting on SLOs

> "Você pode receber 144 alertas por dia, todo dia, não agir em nenhum, e ainda cumprir o SLO."

**TL;DR:** Onde o SLO vira YAML. O cap 2 deu o error budget, o cap 4 deu o counter monotônico, e aqui os dois se encontram no **burn rate**: quão rápido, relativo ao SLO, o serviço queima a verba. O capítulo é uma escada de 6 tentativas — as 3 primeiras falham de propósito, cada uma de um jeito educativo (a 3ª é o `for:` que quase todo mundo usa hoje, e o livro recomenda **não** usar). A resposta final é **multiwindow, multi-burn-rate**: janela longa dá precision, janela curta dá reset time, e as duas juntas por `and`. Tabela pronta pra copiar na seção 5.

Autor: Steven Thurgood (o mesmo dos caps 2 e 4 — a linha de raciocínio é contínua).

---

## 1. As 4 réguas de qualquer estratégia de alerta

Objetivo: transformar SLI + error budget numa regra que notifique de um **evento significativo**, definido como o evento que **consome uma fração grande do error budget**. "Significativo" não é definição técnica, é econômica: não importa se foi 500 ou timeout, importa quanto custou da verba.

| Parâmetro | Pergunta |
|---|---|
| **Precision** | Dos eventos detectados, quantos eram significativos? (100% = todo alerta é real) |
| **Recall** | Dos eventos significativos, quantos foram detectados? (100% = nada escapa) |
| **Detection time** | Quanto tempo pra notificar? Tempo longo **impacta negativamente o budget**, porque a verba queima enquanto ninguém sabe |
| **Reset time** | Quanto tempo o alerta continua disparando **depois de resolvido**? Reset longo leva a confusão ou a **problema sendo ignorado** |

Precision e recall vêm do cap 2 (o detector de fumaça: "gritei à toa?" / "escapou incêndio?"). Detection e reset time são a contribuição nova deste capítulo — e **reset time é o parâmetro que quase ninguém considera** ao projetar alerta. Alerta que fica gritando depois do fim ensina o time a não acreditar em alerta.

**A tensão que faz o capítulo existir: os 4 puxam pra lados opostos.** Detecção rápida pede janela curta, que gera falso positivo. Precisão pede janela longa, que atrasa a detecção e piora o reset. As 6 iterações buscam segurar os quatro ao mesmo tempo.

Nota de vocabulário: "error budget" e "error rate" valem pra **todo SLI**, não só os com "erro" no nome. Latência acima do threshold é bad event igual. Budget = número de eventos ruins permitidos; error rate = razão de ruins sobre total.

## 2. As três tentativas que falham

### 1: error rate ≥ threshold do SLO

Janela curta (10 min), alerta se o error rate nela exceder o SLO. SLO 99,9%/30d → alerta se os últimos 10 min tiverem ≥ 0,1%.

```yaml
- alert: HighErrorRate
  expr: job:slo_errors_per_request:ratio_rate10m{job="myjob"} >= 0.001
```

A recording rule por trás (o cap 4 pagando: counter, `rate`, pré-computado):

```yaml
record: job:slo_errors_per_request:ratio_rate10m
expr: sum(rate(slo_errors[10m])) by (job)
      /
      sum(rate(slo_requests[10m])) by (job)
```

Se o job não exporta `slo_errors`/`slo_requests`, cria a série renomeando métrica existente (`record: slo_errors` / `expr: http_errors`).

Alertar quando o error rate recente **iguala** o SLO detecta um gasto de budget de **tamanho da janela ÷ período de report** = 10 min ÷ 30 dias = **0,02% do budget mensal**. Esse número é a sentença de morte.

**Prós:** detection time 0,6 s pra outage total; recall bom.
**Contra fatal:** precision baixíssima. 0,1% de erro por 10 min dispara consumindo 0,02% do budget. No extremo: **144 alertas/dia, todo dia, sem agir em nenhum, e o SLO cumprido.**

A melhor definição de alerta inútil que existe: tecnicamente correto, operacionalmente irrelevante.

### 2: aumentar a janela

Só me notifique se consumir **5% do budget de 30 dias** → janela de **36 horas**.

```yaml
- alert: HighErrorRate
  expr: job:slo_errors_per_request:ratio_rate36h{job="myjob"} > 0.001
```

Detection time = (1 − SLO) ÷ error ratio × tamanho da janela → **2 min 10 s** pra outage de 100%. Precision melhor.

**Quebra em dois lugares:**

- **Reset time terrível**: outage de 100% dispara em ~2 min e **continua disparando por 36 horas** (Fig. 5-2: o error rate atual já é zero, a média de 36h segue acima do threshold). Incidente de 2 minutos, alerta de um dia e meio.
- **Custo**: taxa sobre janela longa é caro em memória e I/O pela quantidade de pontos.

Janela longa **contamina o presente com o passado** — e o efeito colateral é pior que barulho: ninguém sabe se o problema ainda existe.

### 3: duração (`for:`) — o que quase todo mundo usa

```yaml
- alert: HighErrorRate
  expr: job:slo_errors_per_request:ratio_rate1m{job="myjob"} > 0.001
  for: 1h
```

**Pró:** precision melhor, exige error rate sustentado.

**Contras graves:**

- **A duração não escala com a severidade.** Outage de **100%** alerta depois de 1 hora — o **mesmo** detection time de um outage de 0,2%. Nessa hora, o outage de 100% consumiu **140% do budget de 30 dias**: você é notificado quando já não há verba.
- **O timer reseta.** Se a métrica voltar pra dentro do SLO por um instante, o cronômetro zera. Um SLI que oscila entre furar e cumprir **pode nunca alertar**.

Fig. 5-3, a demonstração cruel: spikes de 100% de erro durando 5 min a cada 10 min, com `for: 10m` sobre janela de 5m. **O alerta nunca disparou** e o serviço consumiu 35% do budget — cada spike comeu ~12% do budget mensal.

Por isso o livro **não recomenda duração como critério de alerta baseado em SLO**. (Rodapé: `for:` curto pode filtrar ruído efêmero, mas os contras seguem valendo.)

**O motivo conceitual da falha: `for:` mede persistência, não custo.** Duas coisas diferentes.

## 3. Burn rate: o conceito central

> **Burn rate é quão rápido, relativo ao SLO, o serviço consome o error budget.**

Burn rate 1 = consumo que deixa exatamente **zero budget no fim da janela do SLO**. Com SLO 99,9%/30d, error rate constante de 0,1% usa exatamente toda a verba.

| Burn rate | Error rate (SLO 99,9%) | Tempo até exaustão |
|---|---|---|
| 1 | 0,1% | 30 dias |
| 2 | 0,2% | 15 dias |
| 10 | 1% | 3 dias |
| 1.000 | 100% | **43 minutos** |

Essa tabela é o coração do capítulo: burn rate converte "estou com 3% de erro" em "fico sem budget em X tempo", que é a única pergunta que importa pra decidir se acorda alguém.

**As duas fórmulas:**

- Tempo até disparar = (1 − SLO) ÷ error ratio × janela × burn rate
- Budget consumido no disparo = burn rate × janela ÷ período

**O uso inverso é como se trabalha na prática: fixa a janela e o gasto significativo, deriva o burn rate.** 5% do budget de 30 dias em 1 hora → burn rate **36** (5% de 30 dias = 36 horas de verba; gastar em 1 hora = 36x).

```yaml
- alert: HighErrorRate
  expr: job:slo_errors_per_request:ratio_rate1h{job="myjob"} > 36 * 0.001
```

**Prós:** precision boa, janela curta e barata, detection time bom, reset time **58 min** (melhor).
**Contras:** **recall baixo** — burn rate de **35x nunca alerta** e consome todo o budget em 20,5 horas. E 58 min de reset ainda é muito.

Threshold único cria ponto cego logo abaixo dele: rápido o bastante pra matar, lento o bastante pra passar sob o radar.

## 4. Tentativa 5: múltiplos burn rates

Vários burn rates com várias janelas, e a sacada de severidade: nem todo evento significativo merece acordar gente. Vale ticket pra incidente que passaria desapercebido mas esgotaria o budget se ignorado (10% em 3 dias) — pega o evento significativo, e como o ritmo dá tempo de endereçar, **não precisa paginar**.

| Budget consumido | Janela | Burn rate | Notificação |
|---|---|---|---|
| 2% | 1 hora | 14,4 | Page |
| 5% | 6 horas | 6 | Page |
| 10% | 3 dias | 1 | Ticket |

```yaml
expr: (
    job:slo_errors_per_request:ratio_rate1h{job="myjob"} > (14.4*0.001)
    or
    job:slo_errors_per_request:ratio_rate6h{job="myjob"} > (6*0.001)
  )
severity: page

expr: job:slo_errors_per_request:ratio_rate3d{job="myjob"} > 0.001
severity: ticket
```

Critério page vs ticket: se vai **esgotar o budget em horas ou poucos dias**, notificação ativa; senão, ticket pro próximo dia útil. (Rodapé, do livro 1: **page e ticket são as duas únicas formas válidas de fazer um humano agir**. Alerta em e-mail que ninguém lê não é alerta.)

**Prós:** adapta à criticidade, precision boa, e **recall bom pela janela de 3 dias** — que fecha o buraco do 35x.
**Contras:** mais números pra gerenciar; reset time **ainda pior** pela janela de 3 dias; e exige **supressão**, porque as condições não são exclusivas — 10% gasto em 5 min **também** significa 5% em 6h e 2% em 1h: um evento, **três notificações**, a menos que o sistema impeça (`inhibit_rules`, cap 4).

## 5. Tentativa 6: multiwindow, multi-burn-rate (a recomendada)

O defeito residual é sempre o mesmo: **janela longa não sabe se o problema ainda está acontecendo**. Solução: adicionar uma **janela curta** que verifica se o budget **ainda está sendo consumido** no momento do disparo. Só alerta se as duas concordarem.

Guideline: **janela curta = 1/12 da janela longa**.

Fig. 5-6, o mecanismo em 4 momentos (15% de erro por 10 min, janelas de 5m e 60m):

1. Erros começam → a curta cruza o threshold **imediatamente**
2. A longa cruza depois de **5 min** → **o alerta começa a disparar** (precisa das duas)
3. Erros param → a curta cai abaixo do threshold **5 min depois** → **o alerta para**
4. A longa só cai 60 min depois, mas isso não importa mais: o `and` já resolveu

**Janela longa dá precision** (é sustentado e custou budget). **Janela curta dá reset time** (ainda está acontecendo?). Cada uma paga uma dívida diferente.

```yaml
expr: (
    job:slo_errors_per_request:ratio_rate1h{job="myjob"} > (14.4*0.001)
    and
    job:slo_errors_per_request:ratio_rate5m{job="myjob"} > (14.4*0.001)
  )
  or
  (
    job:slo_errors_per_request:ratio_rate6h{job="myjob"} > (6*0.001)
    and
    job:slo_errors_per_request:ratio_rate30m{job="myjob"} > (6*0.001)
  )
severity: page

expr: (
    job:slo_errors_per_request:ratio_rate24h{job="myjob"} > (3*0.001)
    and
    job:slo_errors_per_request:ratio_rate2h{job="myjob"} > (3*0.001)
  )
  or
  (
    job:slo_errors_per_request:ratio_rate3d{job="myjob"} > 0.001
    and
    job:slo_errors_per_request:ratio_rate6h{job="myjob"} > 0.001
  )
severity: ticket
```

**Os parâmetros recomendados (SLO 99,9%) — a tabela pra copiar:**

| Severidade | Janela longa | Janela curta | Burn rate | Budget consumido |
|---|---|---|---|---|
| Page | 1 hora | 5 minutos | 14,4 | 2% |
| Page | 6 horas | 30 minutos | 6 | 5% |
| Ticket | 3 dias | 6 horas | 1 | 10% |

(O YAML final acrescenta um nível que não estava na tentativa 5: **ticket com burn rate 3 em 24h/2h**. Quatro níveis no total.)

**Prós:** framework flexível controlando o tipo de alerta pela severidade e pelas necessidades da organização; precision boa; recall bom pela janela de 3 dias. Dispara só depois de 2% de budget consumido, mas para de disparar **5 min** depois em vez de 1 hora.
**Contra único:** muitos parâmetros, regras difíceis de gerenciar → ver Alerting at Scale.

Veredito: multiwindow multi-burn-rate é a abordagem mais apropriada na maioria dos casos.

## 6. Serviços de baixo tráfego

Tudo acima assume volume suficiente pra dar sinal com significado.

A conta do problema: **10 requests/hora**. **Um** request falho = error rate horário de **10%**. Pra SLO 99,9%, isso é burn rate de **1.000x** e pagina na hora, porque consumiu **13,9% do budget de 30 dias**. O cenário permite **sete** falhas em 30 dias — e request único falha por motivo efêmero e desinteressante, que não vale resolver como outage sistêmico.

A pergunta que decide: **qual o impacto de um único request falho?** Se são requests one-off de alto valor, sem retry, meta altíssima pode ser apropriada e pode fazer sentido investigar cada falha — mas nesse caso o alerta chega **tarde demais** de qualquer forma.

**1. Gerar tráfego artificial.** Sintetiza atividade pra checar erro e latência. Vantagem: mais sinal e **reaproveita a lógica de monitoring e os valores de SLO existentes**; as peças provavelmente já existem (black-box prober, teste de integração). Desvantagens: serviço que merece SRE tem superfície de controle grande — idealmente o sistema é **projetado** pra ser monitorado assim; você sintetiza só uma **fração** dos tipos de request (pior se stateful); e o risco maior — se o problema afeta usuário real mas **não** o tráfego artificial, os requests sintéticos bem-sucedidos **escondem o sinal real**.

**2. Combinar serviços.** Agrupar requests de vários serviços de baixo tráfego num grupo de nível mais alto detecta evento significativo com mais precisão e menos falso positivo. Exige serviços **relacionados** (microserviços do mesmo produto, tipos de request do mesmo binário). Desvantagem: falha completa de um serviço individual pode **não** contar como significativa. Mitigação: escolher serviços com **failure domain compartilhado** (banco comum), e manter alertas de período longo que eventualmente pegam as falhas de 100% individuais.

**3. Mudar serviço/infra pra reduzir o impacto de uma falha.** Cliente com **retry, backoff exponencial e jitter**; fallback path capturando o request pra execução posterior (servidor ou cliente). Útil em alto tráfego e **ainda mais** em baixo: permite mais eventos falhos no budget, mais sinal, e mais tempo de resposta antes de virar significativo.

**4. Baixar o SLO ou aumentar a janela.** Se um número pequeno de erros consome budget, você **realmente** precisa acordar alguém? Se não, o usuário ficaria igualmente feliz com SLO menor, e o engenheiro só é notificado de outage sustentado. Implementar é trivial depois de negociado (99,9% → 99%: coloca o valor novo nos sistemas). Custo: **é decisão de produto** — mexe em expectativa de comportamento e em quando aplicar a error budget policy, e isso pode importar mais que evitar alertas de sinal baixo. Análogo: **aumentar a janela** do alerta.

Na prática o Google usa **combinação**: tráfego falso quando dá boa cobertura, cliente modificado pra falha efêmera machucar menos, agregação de serviços com failure mode compartilhado, e threshold de SLO proporcional ao impacto real de um request falho.

## 7. Metas extremas de disponibilidade

**Meta muito baixa.** Serviço com 90%: a tabela manda paginar em 2% do budget em 1 hora, mas um outage de **100%** consome só **1,4%** nessa hora — **o alerta nunca poderia disparar**. Budget em período longo exige tunar os parâmetros.

**Meta muito alta.** Pra 99,999% mensal, outage de 100% esgota o budget em **26 segundos** — menor que o intervalo de coleta de muitos sistemas, sem contar o tempo end-to-end de gerar e entregar o alerta por e-mail/SMS. Mesmo indo direto pra remediação automatizada, o problema pode consumir a verba antes da mitigação.

E a conclusão filosófica: ser notificado de que restam 26 segundos não é **estratégia ruim** — só **não serve pra defender o SLO**. A única forma de defender esse nível é **projetar o sistema pra que a chance de outage de 100% seja extremamente baixa**. Exemplo: rollout pra **1% dos usuários** queima budget na mesma proporção, e você passa a ter **43 minutos** em vez de 26 segundos (→ cap 16, Canarying Releases).

**Acima de certo nível de confiabilidade, alerta deixa de ser a ferramenta e arquitetura de deploy passa a ser.** Canary não é luxo de empresa grande: é o único mecanismo que funciona quando o budget se mede em segundos.

## 8. Alerting at scale

Com 100 microserviços (ou um serviço com 100 tipos de request), parâmetro custom por serviço acumula toil e carga cognitiva que **não escala**. Recomendação forte: **não** especifique janela e burn rate por serviço — decidiu os parâmetros, **aplica em todos**. (Única exceção: mudança temporária durante outage em andamento, quando você não quer ser notificado naquele período.)

A técnica: **agrupar tipos de request em buckets** de requisitos parecidos.

| Classe | Disponibilidade | Latência @90% | Latência @99% |
|---|---|---|---|
| CRITICAL | 99,99% | 100 ms | 200 ms |
| HIGH_FAST | 99,9% | 100 ms | 200 ms |
| HIGH_SLOW | 99,9% | 1.000 ms | 5.000 ms |
| LOW | 99% | — | — |
| NO_SLO | — | — | — |

- **CRITICAL**: os tipos mais importantes (login)
- **HIGH_FAST**: core interativo, alta disponibilidade e baixa latência (clicar e ver quanto o inventário de anúncio rendeu no mês)
- **HIGH_SLOW**: importante mas tolerante a latência (gerar relatório de campanhas de anos)
- **LOW**: precisa de alguma disponibilidade, mas outage é quase invisível (polling de notificação que pode falhar por longos períodos)
- **NO_SLO**: completamente invisível ao usuário (dark launch, alpha explicitamente fora de SLO)

Argumento final: cinco buckets dão **fidelidade suficiente pra proteger a felicidade do usuário** com menos toil que um sistema mais complicado e caro que provavelmente mapeia melhor a experiência real. **Aproximação de propósito**, porque o custo de gerenciar o modelo perfeito é maior que o ganho.

É a versão escalável do bucketing do cap 2, e a resposta pra consultoria multi-cliente: 5 classes reaproveitadas em 15 clientes, não 15 configurações artesanais.

## 9. Conclusão

Com SLOs significativos, entendidos e representados em métricas, dá pra configurar alerta que notifica o on-call **só quando há ameaça acionável e específica ao error budget**. As técnicas vão de "alerta quando o error rate passa do SLO" até múltiplos níveis de burn rate e janela, e na maioria dos casos o **multiwindow multi-burn-rate** é o mais apropriado pra defender o SLO.

## 10. Detector de entrevista

1. "Os alertas de vocês usam `for:` ou burn rate?" (`for:` como critério de SLO = o cap 5 não chegou lá)
2. "Quanto tempo um alerta continua disparando depois do problema resolvido?" (reset time é o parâmetro que ninguém mede)
3. "O que decide se algo vira page ou ticket?" (tempo até exaustão do budget vs 'sensação de gravidade')
4. "Com 40 serviços, quantas configurações de alerta diferentes existem?" (buckets vs artesanato por serviço)
5. "Como vocês alertam num serviço de pouco tráfego?" (se a resposta é 'do mesmo jeito', existe page por request único falho)

## 11. Conexões com meu stack

- **Prometheus**: as 4 recording rules por SLI (`ratio_rate5m`, `1h`, `30m`, `6h`, `2h`, `24h`, `6h`, `3d`) pré-computadas — e o cap 4 explica por que isso é obrigatório, não otimização
- **`for:` nos alertas atuais**: revisar todo alerta de SLO que usa duração; `for:` mede persistência, não custo
- **Alertmanager**: `inhibit_rules` resolvendo o problema das três notificações simultâneas da tentativa 5; roteamento page vs ticket como severidades distintas
- **Multi-cliente WH1**: os 5 buckets (CRITICAL → NO_SLO) como padrão único aplicado em todos, em vez de tunar por cliente; o SLO muda, os parâmetros de alerta não
- **Serviços internos de baixo tráfego**: blackbox_exporter como tráfego artificial (já existe, é só reaproveitar), com o cuidado de que sonda verde pode esconder dor real
- **Canary / Argo Rollouts**: pra SLO muito alto, o alerta não defende — deploy progressivo defende. Rollout de 1% troca 26 s por 43 min de budget
- **DeployGuard / TenantForge**: SLI de webhook (admission negada indevidamente) e de reconcile (tempo até convergência) entram nas mesmas recording rules

---

*Próximo: Capítulo 6, Eliminating Toil — com a Parte I fechada, o foco sai da medição e vai pro trabalho que consome o SRE.*
