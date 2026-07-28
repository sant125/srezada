# Capítulo 2 · Implementing SLOs

> "Sem SLOs, não existe necessidade de SREs."

**TL;DR:** O cap 1 vendeu a filosofia; este é o manual de instalação (os autores dizem que é, em muitos sentidos, o capítulo mais importante do livro). A receita: escolhe SLIs na forma boas ÷ total, mede com o que já existe, transforma a performance atual num SLO honesto (arredondado pro lado da folga), define janela rolling de semanas inteiras, coleta as três assinaturas e escreve a error budget policy que dá dentes ao número. Depois mantém o SLO honesto: acareia ele com a infelicidade real do usuário (precision/recall), refina, e usa o budget como moeda única pra ranquear incidentes e decidir onde investir engenharia.

---

## 1. SLO com dentes vs SLO decorativo

O trabalho diário do SRE é defender o SLO no curto prazo e garantir que ele se sustenta no médio/longo. SLO é o que transforma confiabilidade de opinião em problema de engenharia com número. Daí a frase da epígrafe.

Três níveis de maturidade:

1. **Greenfield**: nada em produção ainda.
2. **Produção com monitoramento**: alerta quando quebra, mas zero objetivo formal, zero budget, meta não dita de 100%. (A maioria dos clientes de consultoria vive aqui.)
3. **SLO sem dentes**: o número existe numa wiki, ninguém sabe pra que serve, nenhuma decisão muda por causa dele.

Pra ter SLO de verdade, quatro condições simultâneas:

1. Todos os stakeholders aprovaram o SLO como adequado pro produto
2. Quem defende concorda que é atingível em circunstâncias normais, sem heroísmo
3. A organização se comprometeu a usar o error budget pra decidir e priorizar, formalizado numa **error budget policy** escrita
4. Existe processo pra refinar o SLO

Faltou uma, o SLO vira só mais um KPI de relatório: medido, plotado, ignorado.

Analogia de CKS: SLO sem error budget policy é **NetworkPolicy num CNI que não implementa policy**. O YAML tá aplicado, `kubectl get netpol` mostra bonitinho, nenhum pacote é dropado. A policy assinada é o CNI do SLO: é ela que faz o número ter consequência.

E no momento em que o SLO fica abaixo de 100%, o tradeoff velocity × confiabilidade precisa de **dono** com autoridade: CTO em empresa pequena, product manager em grande.

## 2. Por que 100% é o alvo errado (versão completa)

Reframe que sustenta tudo: o número do SLO é uma **estimativa do limiar de felicidade do usuário**. Acima, quase todo mundo satisfeito; abaixo, reclamação e churn. Felicidade é fuzzy e não se mede direto, então confiabilidade é o proxy mensurável dela.

**SLO ≠ SLA.** SLA é contrato comercial que dispara quando o usuário tá tão infeliz que você compensa (crédito, multa). É o que AWS/GCP te devem. SLO é sempre mais apertado: SLA é o chão jurídico, SLO é o alvo de engenharia. E o SLO prometido pro cliente tem que caber dentro do SLA do provedor, porque falha dele debita da sua conta.

Os 4 argumentos contra 100%:

1. **A matemática não deixa.** Mesmo com redundância, health check e failover, a probabilidade de componentes falharem simultaneamente nunca é zero.
2. **O usuário nunca experimenta 100% mesmo.** O wifi e o 4G dele caem mais que teu app; a cadeia entre serviço e usuário é longa e qualquer elo quebra. Cada nove extra multiplica o custo entregando utilidade marginal que tende a zero.
3. **100% sustentado = serviço congelado.** A fonte nº 1 de outage é mudança: feature, patch de segurança, hardware novo, scaling. Meta de 100% proíbe mexer, e serviço parado morre. Versão segurança: com meta implícita de 100%, aplicar patch de CVE crítica é "risco de downtime" sem cota aprovada; a "segurança" de não mexer vira insegurança real.
4. **Budget zero te condena a ser só reativo.** Error budget = 100% − SLO = zero. Qualquer erro (garantido de acontecer) é emergência, sempre, sem espaço estrutural pra trabalho proativo. Frase do livro: **"confiabilidade de 100% não é SLO de cultura de engenharia, é SLO de time de operações"**.

Fechando com o cap 1: a regra dos 50% é matematicamente impossível com budget zero. O nível 2 de maturidade (monitoramento + 100% implícito) é o mecanismo que mantém ops sendo bombeiro pra sempre, e o rename-and-shame ganha raiz estrutural. SLO abaixo de 100% não rebaixa a régua, **abre o espaço pra engenharia existir**.

## 3. SLI: a régua padronizada

Tudo vira a mesma forma: **eventos bons ÷ eventos totais**. Requests HTTP com sucesso / total. Chamadas gRPC completadas em < 100ms / total. Buscas que usaram o corpus inteiro / total incluindo as que degradaram graciosamente. Até "minutos bons de usuário / total de minutos".

Por que essa forma ganha de qualquer outra:

1. **Escala 0 a 100% intuitiva**: zero = nada funciona, cem = nada quebrado. Qualquer stakeholder lê sem manual.
2. **O budget cai de graça da definição**: budget = 100% − SLO, e vira moeda contável. 3M requests em 4 semanas a 99,9% = verba de 3.000 erros. Incidente de 1.500 erros **custou 50% da verba**. Gravidade de outage deixou de ser opinião e virou porcentagem.
3. **Padronização habilita tooling**: todo SLI com numerador, denominador e threshold = UMA lógica de alerta, UM cálculo de budget, UM template de dashboard, estampado no parque inteiro.

### Spec vs implementation

**SLI specification** = o que importa pro usuário, independente de como medir ("fração de requests da home carregadas em < 100ms"). **SLI implementation** = a spec + uma forma concreta de medir. Spec é a interface, implementation é a classe concreta.

A mesma spec, três implementações, subindo a escada:

| Implementação | Enxerga | Ponto cego |
|---|---|---|
| Log do servidor | O que chegou no backend | Request que morreu antes (DNS, rede, LB) |
| Prober black-box (JS em browser numa VM) | Falha de rede, cert, DNS | Tráfego sintético: problema de subconjunto real (só mobile, só uma região) |
| Instrumentação no client (RUM) | A verdade do usuário | Custo: mexer no client + infra de coleta com requisitos de confiabilidade próprios (quem monitora o monitor?) |

Tradeoff triplo: **quality** (fidelidade à experiência real), **coverage** (fração dos usuários capturada), **custo**. Não existe implementação perfeita, existe a mais honesta que o orçamento paga hoje. A migração Zabbix → Prometheus foi subir essa escada: ping de host nem SLI é, é "máquina de pé", que mente.

### Primeira versão não precisa estar certa

Objetivo da rodada 1: **algo medido + feedback loop**. Nuance histórica: o livro 1 dizia pra não basear SLO na performance atual (te prende num SLO estrito demais); o Workbook aceita performance atual como ponto de partida legítimo, desde que exista processo de iteração rodando.

Aviso: **usuário calibra expectativa pelo entregue, não pelo prometido**. Entregou 99,999% em 10ms prometendo 99,9%, regressão pro nível prometido gera revolta igual. Caso Chubby (lock service do Google): tão confiável que os times pararam de tratar falha como possibilidade; a solução do SRE foi derrubar **de propósito** quando sobrava budget no trimestre, pra manter os dependentes honestos. Confiabilidade entregue em excesso vira contrato implícito.

### Como começar sem travar

UMA aplicação (não o produto inteiro). Define quem são os "usuários" dela (cuja felicidade tu otimiza). Lista as interações mais comuns. Desenha o diagrama de arquitetura com fluxo de request, fluxo de dados e dependências críticas. Regra de bolso: relevante mas **fácil de medir** ganha de perfeito e caro. Itera depois.

## 4. Os 3 tipos de componente e o cardápio de SLIs

Qualquer sistema se abstrai em 3 tipos, e cada tipo tem cardápio pronto:

- **Request-driven**: usuário gera evento e espera resposta (Deployment + Service + Gateway)
- **Pipeline**: entra registro, transforma, deposita em outro lugar; do tempo real ao batch de horas (CronJob, consumer de fila). O Prometheus É um pipeline: coleta métricas remotas, gera séries temporais e alertas.
- **Storage**: aceita dado agora, devolve depois (StatefulSet + PVC, S3, CloudSQL)

Exemplo que carrega o capítulo: joguinho mobile. App fala com HTTP API; API grava estado num storage permanente; pipeline periódico gera league tables (dia, semana, all time) num segundo store, servido pro app e pro site; usuário sobe avatar customizado. Os 3 tipos num sistema bobo.

| Tipo | SLI | Pergunta que responde |
|---|---|---|
| Request-driven | Availability | Das requests, quantas com resposta de sucesso? |
| Request-driven | Latency | Quantas voltaram abaixo do threshold? |
| Request-driven | Quality | Quantas respostas saíram **sem degradação**? |
| Pipeline | Freshness | Dos dados, quantos atualizados há menos de X? |
| Pipeline | Correctness | Dos registros que entraram, quantos saíram com valor certo? |
| Pipeline | Coverage | Batch: dos jobs, quantos processaram acima do alvo? Streaming: dos registros, quantos processados dentro da janela? |
| Storage | Durability | Dos registros escritos, quantos conseguem ser lidos de volta? |

Destaques do cardápio:

- **Quality**: se o serviço degrada graciosamente, degradação vira coisa contável. User Data caiu, o jogo continua com avatar genérico: availability 100%, mas a fração degradada é SLI próprio. Antídoto contra "tá no ar" pela metade.
- **Freshness ponderada por acesso**: o ideal é contar **leituras de dado stale**, não existência de dado stale. Registro velho sem leitor não machuca ninguém; league table velha lida 1 milhão de vezes machuca 1 milhão de vezes. (Cache: chave vencida fora do hot path é irrelevante, no hot path é incidente.)
- **Durability agregada mente**: 1 bilhão de registros de 10 anos, tudo legível menos os de hoje = 99,9999% de durability e usuário puto, porque o que ele queria era exatamente o de hoje. A média esconde a fatia que importa. Velero: "backup existe" não é SLI; "restaurei o que o cliente precisava" é.

Overlap entre categorias é normal (request-driven com correctness, pipeline com availability); categoria é guia, não jaula. Regra: **5 ou menos tipos de SLI**, cobrindo só o crítico pro usuário. Passou disso, dilui sinal e vira enfeite de dashboard.

### Degraus múltiplos no mesmo SLI

Threshold único mente sobre a cauda: 90% das requests em 100ms e os outros 10% em **dez segundos** deixa o SLO "90% < 100ms" verde, batido, comemorado, com 1 em cada 10 usuários sofrendo. Solução: degraus. 90% < 100ms **e** 99% < 400ms, o segundo vigia a cauda. No Prometheus sai de graça: mesmo histogram, duas rules com `le` diferente. Vale pra qualquer SLI parametrizado.

## 5. Mão na massa: da spec pro número

Regra de ouro da primeira implementação: **o mínimo de engenharia que produz um número**. Se o log já existe mas prober leva semanas e instrumentar o client leva meses, usa o log. Só precisa de duas informações: status de sucesso/falha e tempo de resposta.

### As escolhas do exemplo

- **Availability/latency da API**: sucesso = tudo que não é 5xx. O 401 de quem errou a própria senha fica fora da conta (erro dele, não teu). Continua escolha de projeto: tem contexto onde enxurrada de 4xx é sintoma de bug teu no client. Fonte escolhida: **load balancer**, pelos dois motivos que formam o critério geral: a métrica já existe (custo zero) e fica um degrau mais perto do usuário que o log do app (o LB vê a request que morreu antes de alcançar um backend saudável).
- **Freshness**: o pipeline grava um **watermark** (timestamp da última atualização) a cada rodada. Implementação escolhida: checagem no client, dois counters ("pedi dado" / "dado tava fresco"), entregando a freshness ponderada por acesso.
- **Coverage**: o pipeline exporta "quantos deveria processar" e "quantos processei". Ponto cego admitido: é autorrelato; registro que o pipeline nem sabia que existia (fonte esquecida, misconfiguration) não entra no denominador. O pipeline corrigindo a própria prova.
- **Correctness**: como validar a saída sem reimplementar o pipeline? **Dado dourado** curado na mão dentro do game state DB, com saídas conhecidas, testado a cada execução. Unit test rodando em produção, contra o pipeline real, a cada run. Caveat: o dado curado precisa ser representativo do dado real, senão tu testa um mundo que não existe.

### Métricas e queries

Counter por backend/status e latência como histogram cumulativo (cada bucket conta requests ≤ aquele boundary):

```
http_requests_total{host="api", status="500"}
http_request_duration_seconds{host="api", le="0.1"}
http_request_duration_seconds{host="api", le="0.4"}
```

SLIs sobre 7 dias:

```promql
sum(rate(http_requests_total{host="api", status!~"5.."}[7d]))
/
sum(rate(http_requests_total{host="api"}[7d]))

histogram_quantile(0.9, rate(http_request_duration_seconds_bucket[7d]))
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[7d]))
```

### Histogram vs counter explícito

O histogram é um conjunto de counters **exatos nos boundaries**. Pergunta que bate num boundary ("quantas ≤ 400ms?") tem resposta exata, tá gravada. Pergunta entre boundaries ("≤ 450ms?") é respondida com **interpolação uniforme dentro do bucket**: chute educado. O `histogram_quantile` devolvendo p90 = 432ms é o mesmo chute ao contrário; a única verdade é "o corte caiu entre 400 e 800".

Alternativa: counter sob medida no threshold exato (`requests_slower_than_450ms`), classificação na hora, zero interpolação. Custo: mais configuração e **zero retroatividade**: mudou o alvo de 450 pra 300, o counter velho não diz nada sobre 300 e o novo só conta daqui pra frente. Histogram muda o threshold na query e recalcula o histórico inteiro na hora.

| | Exato | Retroativo |
|---|---|---|
| Counter explícito | Sim | Não |
| Histogram | Não (interpola) | Sim |

Na fase de **descobrir** o SLO iterando, retroatividade vale mais que exatidão: fica com o histogram. Quando o SLO estabiliza, mata o dilema: **alinha um boundary com o threshold** (SLO fechou em 450ms? adiciona `0.45` nos buckets; aquele bucket vira counter exato e a grade continua lá pro resto). E native histograms (Prometheus 2.40+, experimental) encolhem o erro de interpolação sem explodir cardinalidade.

### Performance atual → primeiro SLO

4 semanas da API do jogo: 3.663.253 requests, 3.557.865 com sucesso = **97,123%**. p90 = 432ms, p99 = 891ms.

Arredondamento sempre **pro lado da folga**: availability pra baixo (97,123% → 97%), threshold de latência pra cima (432 → 450ms, 891 → 900ms). Granularidade de ~50ms porque usuário não percebe menos que isso (a régua muda pra jogo real-time).

| SLO | Objetivo | Budget em 4 semanas |
|---|---|---|
| Availability | 97% | 109.897 falhas |
| Latency | 90% < 450ms | 366.325 lentas |
| Latency | 99% < 900ms | 36.632 lentas |

**97%** choca de propósito: 3 em cada 100 requests falhando. Mas é o número honesto, é o que o serviço FAZ hoje. Começar com 97% verdadeiro vale mais que 99,9% de fantasia, porque a distância entre real e desejado fica visível, medível e negociável.

## 6. A janela de medição

**Rolling window** é a memória do usuário: outage no dia 31 não é perdoado no dia 1º do mês seguinte. Regra que quase ninguém segue: janela em **número inteiro de semanas** (4 semanas, não "30 dias"), senão o período ora pega 4 fins de semana, ora 5, e o SLI oscila por composição de calendário, não por confiabilidade. Denominador estável = sinal limpo.

**Calendar window** é a lente do negócio: fecha com trimestre, planejamento, headcount. Custo: no meio do período o denominador ainda não existe, então decisão mid-quarter é especulação sobre quanto budget vai sobrar.

Tamanho da janela = tamanho da decisão que ela suporta. Curta: correção tática rápida (perdeu a semana, prioriza os bugs certos e salva a próxima). Longa: aposta estratégica (DB distribuído HA vs automatizar rollout/rollback vs stack duplicada em outra zona pede meses de dado). Régua do livro: **quantidade de dado proporcional ao tamanho da aposta de engenharia**.

Resposta do Google: não escolhe, empilha. **Rolling de 4 semanas** como janela oficial do SLO, resumo semanal pra priorização tática, relatório trimestral pra planejamento. (Consultoria: relatório de cliente em calendário porque casa com fatura e contrato; operação interna enxerga melhor em rolling.)

Se o SLO veio da medição, "bater a meta" no período é tautologia. O exercício útil é a **distribuição**: teve dia fora do SLO? Bate com incidente conhecido? Alguém agiu (ou devia ter agido)? E sem histórico nenhum, um health check remoto vagabundo (ping ou GET periódico de fora) já é fonte de dado válida pra primeira rodada.

## 7. As três assinaturas e a error budget policy

O SLO proposto só vale quando três partes assinam, e cada assinatura garante uma coisa diferente:

| Quem | O que a assinatura garante |
|---|---|
| Product manager | Abaixo desse threshold o usuário sofre de verdade, e vale gastar engenharia pra consertar |
| Devs do produto | Estourou o budget, o time **vai** agir pra reduzir risco até voltar |
| SRE/ops | Dá pra defender esse SLO sem heroísmo, toil excessivo ou burnout |

Fechou as três, o livro declara: "a parte difícil acabou". Com rodapé impagável: *"disclaimer: pode haver tarefas mais difíceis no seu futuro"*.

### Aprovar a policy é o teste diagnóstico do SLO

Cada recusa aponta um defeito diferente:

- **SRE recusa** (indefensável sem toil infinito) → relaxa o objetivo
- **Dev/PM recusam** (consertar confiabilidade nesse nível mata a velocity) → relaxa, com o aviso que o PM engole junto: SLO mais frouxo = SRE responde a menos situações; menos proteção é o preço
- **PM recusa pelo outro lado** (muito usuário sofre antes da policy disparar ação) → aperta
- **Ninguém converge** → itera no SLI/SLO: falta dado, falta recurso, ou o indicador tá errado?

### A escada de ações

Policy escrita = ação específica + dono quando o budget zera:

1. Bugs de confiabilidade das últimas 4 semanas viram prioridade máxima do time dev
2. Time dev foca **exclusivamente** em confiabilidade até voltar pro SLO, **com aval da alta gestão pra recusar demanda externa** (sem essa cobertura política a escada desaba: o time fica prensado entre a policy e a empresa cobrando feature)
3. Freeze de produção: certas mudanças param até o budget regenerar na janela móvel

Failure mode admitido pelo livro: budget zerou e os stakeholders acham que aplicar a policy "não cabe dessa vez". Diagnóstico: a aprovação era fake. Volta pro estágio de aprovação. Policy que só vale quando é conveniente é decoração.

### Papel escrito, publicado

Dois documentos em lugar visível, mesmo esqueleto de metadados: autores, revisores, aprovadores, data de aprovação e **data da próxima revisão**.

- **SLO doc** adiciona: objetivos, implementações de SLI, como o budget é calculado e consumido, e o **racional dos números**, confessando por escrito quando foi chute ad hoc. Motivo: o engenheiro de daqui a 2 anos não pode tratar teu chute como ciência e construir decisão em cima.
- **Policy doc** adiciona: ações de exaustão e **rota de escalonamento** pra quando alguém discordar do cálculo ou da adequação das ações no caso concreto.

Cadência de revisão acompanha maturidade: cultura nova revisa mensal, estabelecida cai pra trimestral ou menos.

### Dashboards e reports

Report de compliance: vários serviços, quantos objetivos bateram no trimestre ("3 de 4"), tendência vs trimestre anterior e vs mesmo trimestre do ano passado (sazonalidade). Dashboard de tendência: budget sendo consumido na janela; um evento único comendo ~15% do budget do trimestre em 2 dias aparece como degrau no gráfico. E a moeda em uso: "esse outage consumiu 30% do budget trimestral", "top 3 incidentes do tri por budget consumido". Num relatório de cliente, isso comunica infinitamente mais que "uptime foi 99,96%".

## 8. O detector de mentira do SLO (precision e recall)

Problema estrutural: o SLO não corrige a própria prova. Definido a partir da medição, tu passa por construção. O loop de melhoria exige **fonte externa de verdade**: infelicidade real de usuário, vinda de fora da telemetria.

Cardápio de fontes, do barato pro caro: outages contados manualmente, tickets de suporte, ligações, post em fórum, sentimento em rede social, sampling de felicidade no produto, entrevista cara a cara. Começa pelo barato e itera. O mais barato nem é técnico: **o PM já conversa com cliente sobre preço e funcionalidade; pede pra incluir confiabilidade no roteiro**. Custo zero, sinal direto da fonte.

### O acareamento: ticket × budget

Gráfico: tickets por dia × budget perdido naquele dia. Duas testemunhas, o usuário e a régua. Dia normal, as duas contam a mesma história (Spearman, correlação de ranking, quantifica sem assumir linearidade). Outlier é o dia em que **discordam**, e a direção aponta o furo:

- **Régua gritou, usuário quieto** (10% de budget, 5 tickets): cheiro de **precision** furada, a régua sente o que o usuário não sente. Investiga antes de cravar: madrugada? segmento que sofre calado? usuário que não abre ticket, cancela?
- **Usuário gritou, régua quieta** (40 tickets, zero budget): **recall** furado na certa, dor real invisível pra régua.

### As duas perguntas do detector de fumaça

- **Recall: escapou incêndio?** Dos problemas que machucaram usuário de verdade, quantos o SLI pegou. Ex: 10 problemas reais no mês, SLI capturou 6 → recall 60%, quatro incêndios escaparam.
- **Precision: gritei à toa?** Do que o SLI capturou, quanto era real. Ex: disparou 8 vezes, 6 batem com problema real → precision 75%, dois gritos à toa.

Tradeoff cruel: sensibilidade pra cima = recall sobe, precision desce (grita com torrada queimada). Pra baixo = precision sobe, recall desce (dorme no incêndio). Não existe regulagem que maximiza os dois, existe o equilíbrio que teu contexto aguenta. Idêntico ao tuning de regra de Falco: regra barulhenta ninguém escuta mais (e o recall efetivo vira zero, porque alerta ignorado não detecta nada), regra cirúrgica demais deixa o `sh` do atacante passar.

E o livro normaliza: gap de cobertura é **esperado**. O serviço muda, o SLI/SLO muda junto. Manutenção, não vergonha.

### As 4 saídas, como triage de 2 perguntas

Pergunta 1: o defeito é na **linha de corte** (o número do SLO) ou na **régua** (o SLI)? Pergunta 2: o conserto cabe agora?

1. **Muda o número**: pro incidente perdido, calcula que SLO teria notificado naquelas datas e dá replay do candidato contra o histórico inteiro de SLI (o histogram retroativo pagando de novo), conferindo que ele não sai gritando com torrada. Não adianta subir recall destruindo a precision. Simétrico pra falso positivo: relaxa.
2. **Muda a régua**: nenhum número salva medição no lugar errado; log de servidor nunca vê a request que morreu no DNS. Duas alavancas: **quality** (servidor → LB → client) e **coverage** (GET simples → health handler que exercita o sistema de verdade → teste executando o JS do client).
3. **SLO aspiracional**: usuário precisa de 99,9%, tu entrega 99,5%, fechar o gap leva trimestres. Adotar já = fora do SLO pra sempre = policy disparando 24/7 = policy vira ruído e morre. Então: SLO atual segura os dentes, o alvo roda em observação do lado, **isento por escrito na policy** ("medido e acompanhado, não dispara ação"). Audit mode de SLO: AppArmor em complain, seccomp com log, webhook em dry-run, regra de Falco entrando como NOTICE. Observa antes de dar dentes.
4. **Itera por ROI**: nas primeiras rodadas, **barato e rápido primeiro**, porque o conserto barato reduz a incerteza da métrica atual e diz se o caro vale o preço antes de tu pagar.

## 9. O SLO como motor de decisão

### A matemática que humilha a intuição

Dois incidentes do joguinho, na moeda do budget (verba: 109.897 erros):

- **Release ruim**: API nova cuspindo 100% NullPointerException por 4h até o revert. 14.066 erros = **13% do budget**.
- **Servidor do state DB morre** (singly homed), restore do backup leva 20h. ~72.000 erros = **65% do budget**.

Intuição: o DB é o monstro. Frequência inverte: **1 falha de servidor em 5 anos** vs **2-3 releases ruins por ano**. Custo anual esperado: releases = 26 a 39% de budget/ano; DB = 65 ÷ 5 = 13%/ano. **Deploy ruim custa o dobro.** Automatizar rollback/canary ganha de investigar o servidor, mesmo o servidor sendo o incidente mais assustador. Risco = probabilidade × impacto, e o budget dá a unidade comum pra comparar incidentes de natureza completamente diferente. (Responde com número a escolha que o cap 1 deixou aberta: rollback automatizado vs datastore replicado.)

Caso extremo fora da escada: **emergência declarada**, com aval da alta gestão pra despriorizar TODA demanda externa até critério de saída (dentro do SLO + passos contra repetição: monitoring, testes, remover dependência perigosa, rearquitetar).

### A matriz das 8 combinações

Três eixos: bateu o SLO? toil alto? usuário feliz?

| SLO | Toil | Satisfação | Ação |
|---|---|---|---|
| Bateu | Baixo | Alta | Gasta o budget: acelera release, ou dá alta e vai cuidar de outro serviço |
| Bateu | Baixo | Baixa | Aperta o SLO |
| Bateu | Alto | Alta | Falso positivo de alerta? Dessensibiliza. Senão: afrouxa temporário (ou descarrega toil) e conserta produto/mitigação automática |
| Bateu | Alto | Baixa | Aperta o SLO |
| Estourou | Baixo | Alta | Afrouxa o SLO |
| Estourou | Baixo | Baixa | Aumenta sensibilidade do alerta |
| Estourou | Alto | Alta | Afrouxa o SLO |
| Estourou | Alto | Baixa | Descarrega toil e conserta produto/mitigação automática |

Chaves de leitura (valem mais que decorar as linhas):

- **Satisfação é o juiz supremo**: sempre que ela discorda do número, quem tá errado é o SLO. Bateu + infeliz = aperta. Estourou + feliz = afrouxa. É o detector de mentira aplicado ao próprio SLO.
- **Toil é o termômetro de sustentabilidade**: bateu na base do heroísmo não é confiabilidade, é hemorragia disfarçada de sucesso.
- **A única emergência de engenharia real é a última linha**: tudo ruim ao mesmo tempo, aí sim conserta-o-produto.
- **A linha esquisita** (estourou, toil baixo, infeliz): tudo ruim e ninguém reagindo = alerta dormindo, sobe a sensibilidade.

### Alta do serviço

Serviço rodando liso, quase sem supervisão: move pra tier menos hands-on, mantém incident response e oversight de alto nível, libera o SRE pra quem precisa. Inverso positivo do "direito de devolver o pager" do cap 1: lá o serviço inoperável perde o SRE de castigo, aqui o maduro **ganha alta**. É o mecanismo que faz SRE escalar sem crescer linear.

## 10. Tópicos avançados

**User journeys.** O SLO ideal mede ação de usuário, não endpoint: buscar produto → carrinho → checkout. Não mapeia nos SLIs existentes (multi-step; no log não dá pra saber se ele falhou no passo 3 ou foi distraído por vídeo de gato, exemplo literal do livro). Regra: identifica o que importa PRIMEIRO, resolve como medir depois (join de logs, instrumentação no client). Medido, vira só mais um SLI. Melhora recall sem sacrificar precision.

**Bucketing.** Nem toda request vale igual: check de notificação ≠ request de billing do anunciante. Label no SLI, SLO por label: Premium 99,99% / Free 99,9%; interativo 90% < 100ms / download de CSV só precisa começar em 5s. Per-cliente é ruidoso (cliente com pouca request ou tá 100% ou tá péssimo por azar amostral), mas o agregado "quantos clientes dentro do SLO agora" é sinal útil. Consultoria multi-cliente: SLO por tier de contrato é literalmente o modelo de negócio.

**Dependências.** Dependência crítica (caiu, tu caiu) precisa de garantia ≥ à da ação que depende dela, e o time dela é dono do SLO dela como se fosse produto. A armadilha da matemática: 99,9% numa zona, duplica em duas, 99,9999%? Só se independentes, e quase nunca são: dependências comuns, failure domain compartilhado, shared fate, control plane global. GKE zonal → regional compra independência de zona, mas um `kubectl apply` errado replica nas 3 zonas em segundos: regional não salva de shared fate. Quando a dependência de OUTRO time estoura teu budget, duas escolas: (a) não congela, a culpa não é tua; (b) congela mesmo assim. O livro fica com a (b): **usuário não sabe de quem é a culpa, só sabe que caiu**. Decide teu critério e escreve na policy.

**Relaxar de propósito.** Sobrou budget: degrada deliberadamente (latência artificial) e mede o impacto na métrica de negócio (conversão). Prêmio: ligar matematicamente latência a venda, o dado mais valioso que existe pra priorizar engenharia. O livro chama de Rubicão: atravessa pensando muito, se atravessar. Pegadinha de interpretação: +50ms sem queda de conversão pode ser "latência não importa" ou "usuário não tem alternativa **ainda**"; quando o concorrente chegar, ele vai sem avisar. Mesmo espírito do Chubby, só que pra calibrar a régua. Expectativa evolui: revisita regularmente.

## 11. Detector de entrevista

1. "Qual o SLO do serviço principal e onde tá documentado?" (uptime genérico ou ninguém sabe = KPI decorativo)
2. "O que aconteceu da última vez que o error budget estourou?" (freeze ou sprint de confiabilidade = real; "nunca estourou" ou resposta vaga = sem dentes)
3. "Como o número do SLO foi escolhido?" (racional documentado vs chute que virou lei)
4. "Me descreve o pior incidente do ano em % de budget consumido" (quem usa a moeda responde na hora)

## 12. Conexões com meu stack

- **Prometheus**: bucket do histogram alinhado com o threshold do SLO (counter exato + grade flexível); recording rules padrão numerador/denominador estampadas nos 15+ clientes; native histograms no radar
- **Grafana**: painel de budget restante na rolling de 4 semanas; relatório mensal de cliente em "% de budget por incidente" no lugar de uptime cru
- **CKS**: SLO aspiracional é audit mode (AppArmor complain, seccomp log, Falco NOTICE, webhook dry-run); precision/recall é o mesmo jogo do tuning de regra de Falco
- **Ambiente MSP multi-cliente**: bucketing por tier de contrato; SLO prometido cabendo dentro do SLA de AWS/GCP, porque falha do provedor debita do meu budget
- **GKE zonal → regional**: compra independência de zona, não de shared fate (config ruim replica nas 3 zonas em segundos)
- **TenantForge / DeployGuard**: webhook em dry-run antes de enforce = SLO aspiracional em forma de código; o operator visto como pipeline tem freshness (estado real vs desejado) e correctness (reconcile certo) como SLIs naturais

---

*Próximo: Capítulo 3, SLO Engineering Case Studies, a receita deste capítulo aplicada por empresas reais.*
