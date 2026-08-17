# Capítulo 4 · Monitoring

> "Cada métrica exposta deve servir a um propósito."

**TL;DR:** O cap 2 é receita; este é **critério de julgamento**. Não ensina a montar dashboard, ensina a avaliar sistema de monitoramento e a decidir o que merece virar métrica. Não existe "melhor ferramenta": existe o peso que você dá aos 5 usos do monitoring, e esse peso dita os tradeoffs. Métrica alerta, log explica. Counter monotônico é pré-requisito arquitetural do burn rate (cap 5). Config de monitoring é código. E alerta não testado é Schrödinger: você descobre se funciona no pior momento possível.

Autores: Jess Frame, Anthony Lenton, Steven Thurgood, Anton Tolchanov e Nejc Trdin, com Carmela Quinito. (Thurgood também assina o cap 2: o capítulo inteiro é escrito pra alimentar o SLO.)

---

## 1. Os 5 usos e os tradeoffs

Escopo declarado: monitoring inclui métricas, log de texto, log estruturado de evento, distributed tracing e introspecção de runtime, mas o capítulo foca em **métricas e log estruturado**, as duas fontes mais adequadas às necessidades fundamentais do SRE. (A parte de distributed tracing que este capítulo deixa de fora é o [o11yzada](../o11yzada), Tema 02 em diante — sinal completo, span, propagação de contexto.)

Ver também [o11yzada Tema 01](../o11yzada/01-observabilidade-sinais-e-correlacao.md) pra a distinção monitoring vs observability (known-unknowns vs unknown-unknowns) que complementa os "5 usos" abaixo — monitoring aqui é sobre *o que você já sabia perguntar*; observability é sobre o que você não sabia.

Os 5 motivos pelos quais SRE monitora (herdados do cap 6 do livro 1):

1. Alertar em condições que exigem atenção
2. Investigar e diagnosticar
3. Exibir informação visualmente
4. Ganhar insight de tendência de recurso/saúde pro planejamento de longo prazo
5. Comparar comportamento antes e depois de uma mudança, ou entre dois grupos de um experimento

**A frase que carrega o capítulo: a importância relativa desses usos é o que gera os tradeoffs** na escolha do sistema. Não existe "melhor ferramenta de monitoring", existe o peso que você dá a cada uso. Peso em alertar → otimiza velocidade e alerta. Peso em planejamento → otimiza retenção longa e agregação.

### Onde os usos que não são alerta aparecem na prática

Só o alerta te procura; o resto você tem que ir buscar. Por isso a impressão de que "monitoring é alerta". Não é:

- **Diagnóstico**: alerta disparou e você fica 40 min cavando gráfico. É o grosso do MTTR, e é onde a *métrica de debug* paga. Se a métrica que responde o "por quê" não existe, sobra tentativa e erro.
- **Antes/depois**: você já faz toda semana sem chamar assim (subiu deploy, fica 10 min olhando o gráfico). Formalizado vira canary (cap 16). Também é o que **prova** win de portfolio: "50% de redução de custo" é gráfico antes/depois, não sensação.
- **Capacidade**: tendência de meses, sem urgência. "O PVC enche quando?", "quantos nós até dezembro?". Alerta não responde isso, porque quando vira alerta já é tarde. Daí a exigência de retenção multimês.
- **Exibição**: dashboard pra quem não é você (cliente, diretoria).

Se na sua operação só o alerta existe, isso é diagnóstico de estágio: reativo. Os outros 4 usos são o que caracteriza operação proativa, exatamente o espaço que o error budget do cap 2 abre.

## 2. Features desejáveis: Speed e Calculations

### Speed

Duas grandezas no mesmo nome.

**Freshness.** Impacta quanto tempo o sistema demora pra te paginar, mas o efeito colateral é pior: dado lento te faz **agir sobre informação errada**. Durante resposta a incidente, se o intervalo entre causa (tomar uma ação) e efeito (ver isso refletido no monitoring) é longo demais, você assume que a mudança não teve efeito, ou deduz uma **correlação falsa**. Régua explícita do livro: **mais de 4 a 5 minutos de atraso** já impacta significativamente a velocidade de resposta.

Prático: `scrape_interval` 60s + janela de `rate` 5m + `for` 10m no alerta = você decide o presente enxergando o passado. Cada camada empilha atraso.

**Velocidade de retrieval.** Query que varre volume grande demora. Solução recomendada: o sistema deve conseguir **criar e armazenar novas séries temporais a partir do dado que chega**, pré-computando resposta pra query comum. Isso é `recording rules`, e casa com o cap 2: SLI padronizado em numerador/denominador se pré-computa uma vez e serve todos os dashboards.

### Calculations

**1. Retenção multimês.** Sem visão longa não há análise de crescimento. Distinção de granularidade que resolve custo: pra planejamento, **dado sumarizado basta** (agregado, sem drill-down). Reter toda métrica detalhada responde "esse comportamento estranho já aconteceu antes?", mas pode ser caro de armazenar ou impraticável de recuperar. Alta resolução recente + agregado histórico = política de downsampling (Thanos/Mimir), com o critério vindo daqui.

**2. Counters monotônicos.** O trecho mais importante: métricas de evento ou consumo de recurso **idealmente devem ser counters que só incrementam**. Com counter, o sistema calcula funções de janela (req/s). E a amarra entre capítulos: **calcular essas taxas sobre janela mais longa (até um mês) dá os blocos de construção do alerta baseado em queima de error budget** (cap 5).

Encadeamento: cap 2 dá a razão boas/total → cap 4 exige que os dois lados sejam counters monotônicos → cap 5 usa `rate()` desses counters em janelas múltiplas. **Counter não é detalhe de implementação, é pré-requisito arquitetural do burn rate.** (Por que monotônico: sobrevive a restart via detecção de reset e permite calcular taxa sobre qualquer janela retroativamente. Gauge de "requests no último minuto" perde a história e não reagrega.)

**3. Percentis, porque operação trivial mascara comportamento ruim.** Suportar percentil (50, 95, 99) ao gravar latência mostra **se 50%, 5% ou 1% das requests estão lentas**; a média aritmética só diz, sem especificar, que o tempo está maior. A média inventa um usuário fictício (99 requests de 10ms + 1 de 10s = média ~110ms, que ninguém experimentou); o percentil descreve usuários reais. Fallbacks pra quem não tem suporte: somar segundos e dividir por requests (que é a própria média, plano B inferior), ou logar toda request e calcular percentil varrendo/amostrando o log. E pode valer gravar o dado bruto num sistema separado pra análise offline (relatório mensal, cálculo intrincado demais pro monitoring).

### Counter, gauge e histogram: quem é quem

Counter + `rate()` **não** vira histogram. São perguntas diferentes:

- **Counter**: "quantas vezes aconteceu?" `rate()` responde "com que frequência agora". Sem noção de grandeza.
- **Histogram**: "quantas vezes em cada faixa de grandeza?" (duração, tamanho). Precisa de vários counters, um por faixa.
- **Gauge**: valor que sobe e desce (goroutines, fila, memória). Não aceita `rate()`.

E o ponto que amarra: **histogram É um conjunto de counters**. O que sai no `/metrics`:

```
http_request_duration_seconds_bucket{le="0.1"}   800000   ← counter
http_request_duration_seconds_bucket{le="0.4"}   950000   ← counter
http_request_duration_seconds_bucket{le="+Inf"} 1000000   ← counter
http_request_duration_seconds_sum                 45000   ← counter
http_request_duration_seconds_count             1000000   ← counter (= o +Inf)
```

Todos monotônicos: por isso a insistência em counter. Ordem correta na query: `rate()` primeiro (por bucket), `histogram_quantile()` depois.

## 3. Interfaces e alertas

**Interfaces.** O sistema deve exibir série temporal em gráfico e estruturar dado em tabela ou outros charts (heatmap, histograma, escala log). O dashboard é a **interface primária** do monitoring, então o formato precisa mostrar claramente o dado que importa.

Ponto pouco aplicado: **audiência diferente precisa de visão diferente do mesmo dado**. Alta gestão quer algo bem distinto do SRE. E dentro de um conjunto de dashboards, exibir os mesmos tipos de dado de forma **consistente** tem valor de comunicação. Complemento: agregar a mesma métrica por dimensões diferentes em tempo real (tipo de máquina, versão, tipo de request), com o time **confortável em drill-down ad hoc** — fatiar o dado é como se acha correlação. (Na prática: fluência em `by`/`without`, não feature de ferramenta.)

**Alerts.** Três capacidades:

1. **Classificar** em categorias, pra resposta proporcional
2. **Severidade diferente**: taxa baixa de erro por mais de uma hora vira ticket; **100% de erro é emergência** com resposta imediata
3. **Supressão**, pra não distrair quem está de plantão

Os dois exemplos canônicos de supressão:

- Todos os nós com a mesma taxa alta de erro → alerta **uma vez** pela taxa global, em vez de um por nó
- Dependência sua com alerta disparando (backend lento) → não alerta também sobre a taxa de erro do seu serviço

Detalhe operacional que morde: garantir que o alerta **deixe de ser suprimido** quando o evento acaba. Supressão sem expiração é cegueira permanente. (Na prática: `group_by` + `inhibit_rules` do Alertmanager. O segundo exemplo é a hierarquia de dependência do cap 2: se a dependência crítica caiu, seu alerta é ruído derivado, não sinal novo.)

Escolha estrutural que fecha a seção: **o nível de controle exigido dita** se você usa serviço de terceiro ou opera o seu.

## 4. Logs vs métricas

Métrica é medição numérica de atributo ou evento, colhida em muitos pontos em intervalos regulares. Log é registro **append-only** de eventos — e o capítulo trata de **log estruturado**, que habilita query e agregação ricas, não texto puro.

| | Logs | Métricas |
|---|---|---|
| Granularidade | Alta, volume grande | Menor |
| Atraso | Inerente entre evento e visibilidade | Quase tempo real |
| Precisão | **Quase sempre mais preciso** | Aproximação |
| Uso típico | Root cause, relatório não urgente | Alerta e dashboard |

As três regras que saem daí:

1. **Alerta e dashboard usam métrica**, pelo tempo quase real
2. **Root cause tende a log**, porque a informação necessária frequentemente não existe como métrica
3. **Relatório não urgente usa log**, porque log quase sempre produz dado mais preciso

**A recomendação contraintuitiva (a mais valiosa da seção):** se você já alerta por métrica, é tentador adicionar alerta por log — por exemplo, pra ser notificado de um único evento excepcional. O livro recomenda **mesmo assim alertar por métrica**: incrementa um **counter** quando o evento acontece e alerta no valor dessa métrica. Motivo: mantém **toda a configuração de alerta num lugar só**.

Não é "log é ruim". É que a **superfície de controle do alerta deve ser única**. Evento raro vira counter, counter vira alerta, alerta vive junto com os outros. Mesmo princípio da padronização de SLI no cap 2: forma única habilita tooling único.

### Os 4 casos reais

**1. Mover informação de log pra métrica (App Engine).** *Problema*: status code HTTP era sinal importante pro cliente debugar, existia em log mas não em métrica. O dashboard só dava taxa **global** de erro, sem o código exato nem a causa. Debug virava: achar no gráfico global quando o erro ocorreu → ler log procurando linhas de erro → tentar correlacionar os dois. As ferramentas de log não davam noção de **escala** (aquele erro é frequente ou isolado?), e o log tinha muita linha irrelevante.

*Solução*: exportar o status code como **label** (`requests_total{status=404}` vs `requests_total{status=500}`). Justificativa de custo explícita: como o número de status codes distintos é limitado, **não** inflou o volume a tamanho impraticável.

*Resultado*: linha separada por categoria de erro, cliente formando hipótese rápido, e **threshold diferente pra erro de cliente e de servidor**, com alerta disparando com mais precisão.

Lição: label é decisão de **cardinalidade**. Status code cabe.

**2. Melhorar log E métrica (Ads SRE, ~50 serviços).** *Problema*: log era **fonte canônica de verdade** pra compliance de SLO, e cada serviço tinha script de processamento cheio de caso especial:

```
SE o status HTTP estava no range (500, 599)
E o campo 'SERVER ERROR' do log está preenchido
E o cookie DEBUG não estava setado no request
E a url não continha '/reports'
E o campo 'exception' não continha 'com.google.ads.PasswordEx...'
ENTÃO incrementa o contador de erro em 1
```

Difícil de manter e usando dado indisponível pro sistema de métricas. Como **métrica dirigia o alerta**, às vezes o alerta não correspondia a erro que o usuário via, e cada alerta exigia um passo explícito de triagem pra saber se era user-facing, atrasando a resposta.

*Solução*: biblioteca acoplada à lógica do framework de cada app, decidindo **em tempo de request** se o erro impacta usuário. A instrumentação escreve a decisão no log **e** exporta como métrica ao mesmo tempo. Se a métrica mostra erro, o log tem o erro exato mais o dado do request pra reproduzir. Todo erro que impacta SLO e aparece no log também mexe na métrica de SLI, que o time então alerta.

*Resultado*: superfície de controle uniforme entre serviços, reuso de tooling no lugar de soluções custom, remoção do código de processamento específico (mais escalabilidade), e — com alertas **diretamente atrelados aos SLOs** — alertas mais acionáveis e **taxa de falso positivo caindo significativamente**.

Cap 2 pagando: falso positivo caindo é **precision subindo**, e a causa foi mover a decisão "isso impacta usuário?" do post-processing frágil pra dentro do request path. Decidir na origem, uma vez.

**3. Manter o log como fonte (o contra-exemplo).** *Problema*: o time olhava **entity IDs** afetados pra determinar impacto e root cause; o dado só existia em log e exigia query ad hoc durante o incidente (minutos pra montar a query + tempo de consultar).

*Decisão*: métrica **não** substituiria o log. Diferente do App Engine, o entity ID pode assumir **milhões de valores**, impraticável como label.

*Solução*: script pronto pras queries, com **o comando documentado no próprio e-mail do alerta**, copiável direto pro terminal.

*Resultado*: sumiu a carga cognitiva de montar a query certa; resultado mais rápido (embora não tanto quanto métrica); e plano B — rodar o script automaticamente quando o alerta dispara, mais um servidor pequeno consultando os logs periodicamente pra manter dado semi-fresco.

Antídoto contra o dogma do caso 1: **alta cardinalidade não vira label, vira automação de acesso**. E documentar o comando no alerta é runbook embutido no ponto de dor.

**4.** (Os casos acima cobrem os três padrões: mover pra métrica, unificar os dois, manter em log.)

## 5. Gerenciando o sistema de monitoring

Premissa: **seu sistema de monitoring é tão importante quanto qualquer outro serviço que você roda**, e merece o mesmo cuidado.

**Config como código.** Histórico de mudança, link da mudança pro ticket, rollback e lint mais fáceis (`promtool` pra validar sintaxe), code review obrigatório. Recomendação forte: config de monitoring também como código. Critério de escolha decorrente: sistema com **configuração intent-based** é preferível a sistema que só oferece UI web ou API CRUD-style (cita `grafanalib` pra componentes tradicionalmente configurados por UI).

Dashboard clicado na UI é config sem histórico, sem review e sem rollback. Provisionado por GitOps é infra. Mesma diferença entre `kubectl edit` e commit.

**Consistência.** Empresa grande equilibra: centralizado dá consistência, time individual quer controle do próprio design. O Google convergiu pra **um framework único rodado centralmente como serviço**: engenheiro que troca de time sobe a rampa mais rápido, debug colaborativo fica mais fácil, dashboards descobríveis e acessíveis (entender o dashboard do outro time acelera os dois lados).

**O conselho de maior alavancagem do capítulo: torne a cobertura básica sem esforço.** Se todos os serviços exportam um conjunto **consistente** de métricas básicas, você coleta automaticamente na organização inteira e provê dashboards consistentes. **Qualquer componente novo já nasce com monitoring básico**, e até time não-técnico usa esse dado. O rodapé diz como: biblioteca comum de instrumentação (OpenCensus, hoje OTel) **ou service mesh como Istio**.

Golden path puro. Em consultoria multi-cliente: chart/módulo base com scrape, dashboard e alerta padrão faz o cliente nº 16 nascer observável sem trabalho novo.

**Loose coupling.** Requisito de negócio muda, produção muda, e o monitoring precisa evoluir conforme os serviços mudam de padrão de falha. Componentes **fracamente acoplados**, com interfaces estáveis pra configurar e passar dado; separar quem **coleta**, **armazena**, **alerta** e **visualiza**. Interface estável = trocar componente por alternativa melhor fica fácil.

Contraste histórico explícito: **uma década atrás, sistema como o Zabbix combinava todas as funções num componente só**. Design moderno separa coleta e avaliação de regra (Prometheus server), storage de longo prazo (InfluxDB), agregação de alerta (Alertmanager) e dashboard (Grafana). O ganho da migração não é "Prometheus é melhor", é **poder trocar uma peça sem trocar o conjunto**.

Padrões abertos de instrumentação citados: **statsd** (daemon de agregação do Etsy, portado pra maioria das linguagens) e **Prometheus** (data model flexível, suporte a label, histograma robusto; formato sendo padronizado como **OpenMetrics**).

**Caso Borgmon → Monarch**, desacoplamento na prática: o sistema legado **combinava dashboard na mesma configuração das regras de alerta**. Na migração, moveram o dashboard pra serviço separado (Viceroy). Como o Viceroy não era componente de nenhum dos dois, o Monarch precisou de **menos requisitos funcionais**, e como o usuário podia graficar dado dos **dois** sistemas, deu pra **migrar gradualmente**. Desacoplar não foi elegância: foi o que tornou a migração incremental possível.

## 6. Metrics with Purpose

**Métrica de SLI é a primeira que você checa quando o alerta de SLO dispara**, e deve aparecer com destaque no dashboard, idealmente na landing page. Mas o dashboard de SLO mostra que você **está** violando, não **por quê**. O que mais exibir? Quatro categorias.

**Intended Changes.** Diagnosticar alerta de SLO é sair da métrica que **notifica** pra métrica que diz o que **causa**. Mudança recente é suspeito nº 1 (cap 2: a fonte nº 1 de outage é mudança). Monitore:

- Versão do binário
- **Flags de linha de comando**, especialmente as que ligam/desligam feature
- Versão da configuração dinâmica, se config é empurrada em runtime

Sem versionamento, monitore ao menos o **timestamp do último build/empacotamento**. Justificativa: correlacionar outage com rollout é muito mais fácil olhando um gráfico linkado do alerta do que vasculhando log de CI/CD depois do fato. (Rodapé: caso onde monitorar via log é atraente, porque mudança de produção é infrequente; log ou métrica, mas **exponha no dashboard**.)

Feature flag como métrica quase ninguém faz, e é o que dá resposta rápida ao "quando começou?".

**Dependencies.** Mesmo sem mudança sua, as dependências podem ter mudado ou quebrado: monitore as respostas das **dependências diretas**. Razoável exportar por dependência: tamanho de request e response em bytes, latência e response code. Ao escolher o que graficar, mantenha os **quatro golden signals** em mente. Labels quebram por response code, nome do método RPC e nome do peer job.

Otimização: instrumentar a **biblioteca cliente RPC de baixo nível** pra exportar essas métricas **uma vez**, em vez de pedir a cada client library. Mais consistência e **dependência nova monitorada de graça**. (É o argumento do sidecar de service mesh e da auto-instrumentação do OTel: instrumenta o transporte, ganha todo serviço novo.)

Caso feio: dependência com **API estreita**, onde tudo passa por um único RPC (`Get`, `Query` ou coisa igualmente inútil) e o comando real vem como argumento. Um ponto único de instrumentação não dá conta: você observa **variância alta de latência** e um percentual de erro que pode ou não indicar que parte dessa API opaca falha por completo. Se é dependência crítica: exportar métrica sob medida que **desempacota os requests** pra chegar no sinal real, ou pedir aos donos uma reescrita expondo API mais ampla com RPCs e métodos distintos.

**Saturation.** Monitorar o uso de **todo recurso** de que o serviço depende. Com limite duro: RAM, disco, quota de CPU. Sem limite claro mas ainda exigindo gestão: file descriptors abertos, threads ativas em pool, tempo de espera em fila, volume de log escrito.

Por linguagem: **Java** — heap e metaspace, mais métricas conforme o tipo de GC. **Go** — número de **goroutines**. (Direto pra operator em Go: goroutine leak em reconcile loop é o bug clássico, e `go_goroutines` já vem no `/metrics` do client_golang.)

Além do alerta em evento significativo (cap 5), pode ser preciso alertar ao **aproximar exaustão**: quando o recurso tem limite duro, e quando cruzar um threshold de uso já **degrada performance**. E tenha métrica pra **todo** recurso, mesmo os bem gerenciados: são vitais pra planejamento de capacidade.

**Status of Served Traffic.** Adicione métrica ou label que permita quebrar o tráfego servido **por status code** (a menos que o SLI já inclua). Recomendações: pra HTTP, monitorar **todos** os response codes, mesmo os que não dão sinal pra alerta, porque alguns são disparados por comportamento incorreto do cliente. Se você aplica rate limit ou quota, monitorar o agregado de **requests negados por falta de quota**. O gráfico ajuda a ver quando o volume de erro muda perceptivelmente durante mudança de produção.

Coerência com o cap 2: 4xx fica fora do SLI (erro do usuário), mas continua **medido**, porque enxurrada de 4xx é sintoma de bug seu no client. Não alerta, mas observa.

### Implementing Purposeful Metrics

**Cada métrica exposta deve servir a um propósito.** Resista à tentação de exportar métricas só porque são fáceis de gerar. Pense em **como serão usadas**: design de métrica (ou a falta dele) tem consequências.

A distinção fina:

- **Métrica de alerta**: idealmente muda **dramaticamente** só quando o sistema entra em estado de problema, e **não muda** em operação normal. (Sinal de alto contraste. Métrica que oscila em operação normal é geradora de falso positivo por construção.)
- **Métrica de debug**: não tem esse requisito; existe pra dar **insight sobre o que está acontecendo quando o alerta dispara**, apontando o aspecto do sistema que potencialmente causa o problema.

Loop de melhoria: **ao escrever um postmortem, pense em qual métrica adicional teria permitido diagnosticar mais rápido**. É o cap 10 conectando aqui, e é o mesmo mecanismo do detector de mentira do cap 2: usar o incidente real como fonte externa de verdade pra corrigir a instrumentação.

Critério que amarra o capítulo: **toda métrica serve a exatamente um destes três propósitos — alertar, debugar ou planejar capacidade.** Nenhum deles? Não exporta: é ruído com custo de armazenamento.

## 7. Testing Alerting Logic

Confissão honesta: no mundo ideal, código de monitoring e alerta teria os mesmos padrões de teste que código de desenvolvimento. Os devs do Prometheus discutem unit test pra monitoring, mas **não existe sistema amplamente adotado** pra isso. *(Nota: hoje existe `promtool test rules`, que não existia à época do livro — a lacuna já está parcialmente fechada.)*

No Google, testam com uma **DSL que cria série temporal sintética**, escrevendo assertions sobre os valores da série derivada ou sobre o **status de firing e a presença de label** de alertas específicos.

Como é processo multiestágio, pede múltiplas famílias de teste. Modelo de **três camadas** (Figura 4-1: Binary → Monitoring Infrastructure → AlertManager → Page/Ticket):

1. **Binary reporting**: as variáveis de métrica exportadas mudam de valor sob certas condições, como esperado
2. **Monitoring configurations**: a avaliação de regra produz o resultado esperado, e condições específicas produzem os alertas esperados
3. **Alerting configurations**: os alertas gerados são **roteados pro destino predeterminado**, com base nos valores dos labels

Plano B quando não dá pra testar sinteticamente: criar um **sistema rodando que exporta métricas conhecidas** (número de requests e de erros) e usá-lo pra validar séries derivadas e alertas.

A justificativa é a frase mais assustadora do capítulo: **é muito provável que suas regras de alerta não disparem por meses ou anos depois de configuradas**, e você precisa de confiança de que, quando a métrica cruzar o threshold, os engenheiros certos serão alertados com notificação que faz sentido.

Alerta não testado é **Schrödinger**: você descobre se funciona exatamente no pior momento. E a camada 3 é a mais esquecida: regra dispara certo e o page vai pro time errado, ou pro canal que ninguém olha às 3h. **Firing correto com roteamento errado é indistinguível de não ter alerta.**

## 8. Conclusão

Como o SRE responde pela confiabilidade dos sistemas em produção, ele precisa ser **intimamente familiar com o sistema de monitoring e suas features**. Sem isso, pode não saber onde olhar, como identificar comportamento anormal ou como achar a informação necessária **durante uma emergência**.

Você provavelmente vai combinar métrica e log, com o mix exato altamente dependente de contexto. O essencial é **coletar métricas que servem a um propósito específico**: planejamento de capacidade, auxílio ao debug, ou notificação direta. Uma vez no lugar, o monitoring precisa ser **visível e útil** — e **teste o setup**. Bom sistema de monitoring paga dividendos, e vale investimento substancial em escolher a solução e iterar até acertar.

## 9. Detector de entrevista

1. "Como vocês testam se um alerta novo realmente dispara e chega em quem deve?" (as três camadas; camada 3 é a que quase ninguém testa)
2. "A configuração dos dashboards e alertas vive onde?" (Git = infra; UI = config sem histórico nem rollback)
3. "Um serviço novo sobe hoje. Ele nasce com monitoring?" (cobertura básica sem esforço vs cada time reinventando)
4. "Qual a última métrica que vocês adicionaram depois de um postmortem?" (loop de melhoria vivo vs dashboard fóssil)
5. "Vocês alertam em cima de log?" (se sim, pergunta como gerenciam a config de alerta em dois lugares)

## 10. Conexões com meu stack

- **Prometheus**: counter monotônico como pré-requisito do burn rate (cap 5); `recording rules` como resposta ao problema de retrieval; `rate()` antes de `histogram_quantile()`, sempre
- **Alertmanager**: `group_by` e `inhibit_rules` são os dois exemplos de supressão do livro; garantir que a supressão expira quando o evento acaba
- **Grafana**: provisionamento por código (GitOps/Terraform) no lugar de dashboard clicado; visão diferente por audiência (SRE vs relatório de cliente)
- **Zabbix → Prometheus**: o livro descreve exatamente essa transição como o caso de loose coupling; o ganho é trocar peça sem trocar conjunto
- **OTel / Istio**: o rodapé aponta os dois como o caminho pra "cobertura básica sem esforço"; instrumentar o transporte dá dependência nova monitorada de graça
- **Ambiente MSP multi-cliente**: módulo base de observabilidade = cliente nº 16 nasce observável; golden path aplicado a monitoring
- **Go (TenantForge / DeployGuard)**: `go_goroutines` como métrica de saturação obrigatória em operator; leak em reconcile loop é o bug clássico
- **Postmortem**: fechar todo incidente com "que métrica teria diagnosticado mais rápido?" e abrir PR da instrumentação

---

*Próximo: Capítulo 5, Alerting on SLOs — onde counter, `rate()` e error budget se encontram no burn rate.*
