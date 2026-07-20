# Capítulo 1 · How SRE Relates to DevOps

> `class SRE implements interface DevOps`

**TL;DR:** DevOps diz O QUE uma organização de engenharia saudável faz (princípios, filosofia). SRE diz COMO (práticas concretas, com números). O capítulo apresenta os dois, prova que são aliados e não rivais, e fecha com a parte que ninguém lê: os incentivos organizacionais que decidem se tudo isso funciona ou desaba.

---

## 1. A ideia que sustenta tudo: perfeição é a meta errada

Três motivos pra 100% de disponibilidade ser o alvo errado:

1. Sistema que nunca cai é sistema em que ninguém mexe, porque toda mudança é risco. Produto parado morre.
2. Passado certo ponto, o usuário não percebe a diferença. O wifi e o 4G dele caem mais que seu app.
3. Cada "nove" extra custa muito mais caro que o anterior, em engenharia e em complexidade.

Então a engenharia madura inverte a pergunta. Em vez de "como nunca falhar?", pergunta: **quanto de falha é aceitável?** E escreve esse número num acordo, combinado entre todo mundo antes.

Analogia do skate: quem nunca cai é quem nunca tenta manobra nova. Você não planeja nunca cair, aceita uma cota de ralada como preço de evoluir. Ralou demais essa semana, segura a onda e volta pro básico. Essa autorregulação é exatamente o mecanismo do error budget.

## 2. Medir do ponto de vista do usuário (SLI)

"Servidor no ar" mente. A máquina pode estar de pé com o app quebrado: login de 20 segundos, metade das requisições em erro, funcionando pra um e quebrado pra outro. Quem importa é o usuário.

A régua certa: **de tudo que os usuários tentaram fazer, quanto de fato deu certo?**

É continha de padaria: numa hora chegaram 1000 requisições, 997 receberam resposta certa e rápida. 997 ÷ 1000 = 99,7% naquela janela. Coisas boas dividido pelo total.

O trabalho de engenharia de verdade é decidir o que entra na conta como "boa". Resposta de 8 segundos conta como sucesso? Não, o usuário já desistiu. Usuário que errou a própria senha conta como falha sua? Não, erro dele. Essa escolha é decisão de projeto, não verdade técnica.

Nome da régua: **SLI** (Service Level Indicator).

## 3. A meta combinada (SLO)

**SLO** (Service Level Objective) é o alvo sobre o SLI, combinado entre produto, engenharia e negócio, antes de qualquer incidente. Exemplo: 99,9% de sucesso numa janela móvel de 30 dias.

Nunca 100%. O SLO é promessa realista, não aspiração. E não adianta prometer 99,99% se o load balancer do provedor entrega 99,95%: falha do provedor debita da SUA conta, porque pro usuário é tudo "o app caiu".

## 4. A margem como verba (error budget)

**Error budget** = 100% menos o SLO. Não é limite pra temer, é **verba pra gastar** com deploy, migração de banco, experimento, chaos testing, manutenção.

| SLO (janela de 30 dias) | Verba de falha |
|---|---|
| 99% | 7h12min |
| 99,9% | 43,2 min |
| 99,95% | 21,6 min |
| 99,99% | 4,3 min |
| 99,999% | 26 segundos |

Cada nove a mais divide a verba por 10 e multiplica o custo por bem mais que 10. Em 99,999% um humano nem responde o pager antes do budget acabar: tudo precisa ser autocorretivo.

Dá pra contar por request em vez de tempo, que é mais justo: 100M req/mês a 99,9% = 100 mil requests podem falhar. Cair de madrugada com tráfego zero não pesa igual a cair no pico.

### Error budget policy: o fim da guerra dev vs ops

Documento assinado antes, por dev, SRE e gestão:

- **Tem saldo** → deploy sai, e ops não barra na base do "acho arriscado"
- **Estourou** → freeze de feature, o sprint vira confiabilidade (os action items acumulados) até a janela móvel recuperar

Ninguém discute opinião: todo mundo olha o mesmo dashboard. A disputa política virou aritmética.

Contraintuitivo importante: verba **sobrando** todo mês também é bug. Ou você está lento demais pra lançar e podia arriscar mais, ou está pagando por nines que ninguém percebe. Confiabilidade acima do SLO é velocity queimada de graça.

## 5. Vigiar o ritmo do gasto (burn rate)

O alarme burro olha a foto do instante: "erro > 1% por 5 min? Acorda alguém". Falha dos dois lados:

- Te acorda às 3h por um soluço de 30 segundos que se resolveu sozinho (30s dentro de 43 min de verba é nada)
- Dorme no ponto com o vazamento lento: erro pequeno e constante que nunca cruza o threshold mas come a verba inteira em duas semanas

A pergunta certa não é "tem erro agora?", é: **nesse ritmo de gasto, quando minha verba acaba?** Igual salário: se o dinheiro precisa durar 30 dias e no dia 2 já foi metade, não precisa de contador pra saber que tá ferrado. Importa o ritmo, não o valor de hoje.

**Burn rate** = taxa de erro ÷ (1 − SLO)

- Burn rate 1 = gastando a verba no ritmo exato de zerar no fim da janela. Vida normal.
- Burn rate 14,4 com SLO 99,9% (erro de 1,44%) = a verba de 30 dias morre em ~2 dias. Acorda gente.

O alerta esperto faz duas perguntas antes de tocar o pager: quão rápido tá o gasto, e há quanto tempo sustentado (o filtro anti-soluço):

| Burn rate | Sustentado por | Ação |
|---|---|---|
| ≥ 14,4 | 1h | Page (queimou ~2% da verba numa hora) |
| ≥ 6 | 6h | Page |
| ≥ 1 | 3 dias | Ticket, resolve amanhã com café |

### No Prometheus

Counter é hodômetro: só cresce, nunca zera. O app soma +1 a cada requisição num contador e +1 a cada erro noutro. `rate()` é a subtração do hodômetro entre dois instantes: valor de agora menos valor de 5 min atrás.

```promql
sum(rate(http_requests_total{code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

Tradução: crescimento dos erros (status 5xx, erro do nosso lado) dividido pelo crescimento do total. A fração da padaria, calculada ao vivo, o dia inteiro, sozinha. Joga num painel do Grafana com a janela do mês e a verba sendo gasta fica visível em tempo real.

Você dorme em paz não porque não tem erro (erro tem sempre), mas porque a matemática tá de guarda e só grita quando a promessa feita ao usuário entra em risco de verdade.

## 6. O que SRE faz o dia todo: matar toil

SRE não é bombeiro. Bombeiro apaga o incêndio de hoje e amanhã apaga o mesmo incêndio de novo, pra sempre, e quanto mais incêndio, mais herói parece. SRE, no terceiro incêndio igual, fala "peraí" e vai reescrever a fiação do prédio. Um trabalha NO sistema, o outro trabalha SOBRE o sistema.

Resolver o problema **uma vez, com software, pra ele nunca mais voltar.**

**Toil** = trabalho operacional manual, repetitivo, automatizável, sem valor duradouro (plantão, ticket, tarefa de mão).

### A regra dos 50%

No máximo metade do tempo do time em trabalho operacional. A outra metade é obrigatoriamente engenharia: a automação, o rollback confiável, a ferramenta que elimina a tarefa manual.

E o capítulo inverte a leitura: **não é teto, é garantia.** Se o operacional passa de 50%, o time tem o direito de parar de aceitar coisa nova e ir automatizar até voltar pro equilíbrio. Sem essa regra o fogo sempre ganha da fiação, porque fogo é urgente e fiação nunca é.

Efeito colateral (Murphy-Beyer): com o tempo você automatiza tudo que dá e sobra só o resíduo inautomatizável, que passa a dominar o trabalho se o time não absorver mais serviços ou mudar de problema.

### Plantão como pesquisa de campo

"Wisdom of production": cada page que te acorda é o sistema contando onde dói de verdade. Essa dor vira a fila de prioridade do que automatizar no próximo ciclo. O loop: opera, sente onde dói, escreve software que remove a dor, opera algo maior.

Por isso SRE escala e ops tradicional não: o time que só apaga fogo cresce linear com a empresa; o time que automatiza continua do mesmo tamanho.

> Conexão pessoal: o TenantForge É esse movimento. Provisionar tenant na mão (namespace, NetworkPolicy, RBAC, quota) é toil clássico. O operator cristaliza isso em software que faz sozinho. Operator de Kubernetes é filosofia SRE compilada em Go.

## 7. class SRE implements interface DevOps

**DevOps é a interface.** Declara os princípios abstratos mas não diz como implementar. Resumo em CALMS (Culture, Automation, Lean, Measurement, Sharing):

- Sem silos entre dev, ops, rede, segurança
- Acidente é falha de sistema, não de indivíduo (punir gera esconderijo de informação; o lucro está em acelerar recuperação, não em prevenir tudo)
- Mudança boa é pequena e frequente (daí CI/CD)
- Cultura vale mais que ferramenta: cultura boa contorna ferramenta ruim, o contrário quase nunca
- Medição objetiva como terreno comum entre áreas

**SRE é a classe concreta.** Implementa cada método com prática e número: SLO, error budget, regra dos 50%, postmortem blameless, tooling único pra dev e SRE (ferramenta divergente entre times é receita de comportamento catastrófico).

Diferença de escopo: DevOps é amplo, organizacional, sensível a contexto, mas silencioso sobre como operar um serviço na segunda de manhã. SRE é estreito, orientado a serviço, opinativo. Síntese do livro: SRE acredita nas mesmas coisas por razões ligeiramente diferentes (apoia CI/CD pela prática operacional, não pelo business case).

## 8. A política: incentivos (a parte que ninguém lê)

Ideia central da seção final, meio cínica e totalmente verdadeira: **gente faz o que o sistema recompensa, não o que o cartaz na parede pede.** Todo o resto do capítulo morre se a estrutura de recompensa remar contra.

### Lei de Goodhart: métrica que vira meta vira jogo

Paga o dev por feature lançada, ele empurra porcaria rápida. Paga o ops por uptime, ele bloqueia todo deploy, porque deploy é o inimigo do número dele. Ninguém é vilão: os dois são perfeitamente racionais dentro da regra que receberam. A guerra dev vs ops não é briga de personalidade, é briga de contracheque.

Antídoto: o error budget troca dois números opostos por UM número compartilhado. Os dois times ganham e perdem juntos.

### Não caçar culpado

Punir quem errou parece justiça, mas o efeito real é matar informação: gente esconde incidente, maquia causa, aponta dedo. Sistema complexo só melhora com informação honesta sobre onde quebra. Três práticas:

1. **Postmortem blameless**: o erro é do sistema que deixou o humano errar
2. **Autoridade pra consertar**: quem pode mexer no código e na config não precisa culpar quem mexe
3. **Direito de devolver o pager**: produto operacionalmente irrecuperável perde o suporte SRE, e o time de produto volta a carregar o próprio plantão. Isso inverte o poder: no modelo antigo ops engole qualquer coisa jogada por cima do muro; com direito de recusa, dev passa a ter interesse próprio em construir coisa operável. (Versão freela: quem nunca pode demitir cliente ruim vira refém do pior cliente.)

Ressalva de escala: startup de 20 pessoas com produto único não tem como "largar o produto". O substituto é priorização: trocar o SE suporta pelo QUANDO suporta. Sistema bem construído entra na fila primeiro.

### Paridade de estima

SRE na mesma escada de carreira, mesmo salário, mesma avaliação que dev. Senão os bons engenheiros migram pra dev e o time regride pra NOC tradicional. É por isso que vaga de SRE em empresa séria paga como vaga de dev: confiabilidade feita por cidadão de segunda classe sai de segunda classe.

### Rename-and-shame

O anti-pattern que encerra o capítulo: a empresa renomeia o time de ops pra "SRE", não muda nada (sem error budget, sem 50%, sem direito de recusa, sem paridade) e seis meses depois culpa o time porque nada melhorou. O crachá é a parte barata; todo o resto deste capítulo é a parte cara.

## 9. Detector de entrevista

Perguntas pra descobrir se a vaga é SRE de verdade ou ops renomeado:

1. "O que acontece aqui quando o error budget estoura?"
2. "O time pode recusar o onboarding de um serviço?"
3. "Quanto do tempo do time vai pra projeto vs operacional?"

Resposta vaga = ops com crachá novo. Resposta concreta = lugar que leva a sério.

## 10. Conexões com meu stack

- **Prometheus/Grafana**: SLI como recording rule, painel de budget restante na janela móvel de 30d
- **Alertmanager**: multiwindow multi-burn-rate no lugar de threshold cru de erro
- **TenantForge / DeployGuard**: eliminação de toil em forma de operator e admission webhook
- **Ambiente MSP multi-cliente**: SLO por cliente vale mais que uptime genérico; falha do provedor debita do meu budget, então o SLO prometido tem que caber no SLA da AWS/GCP

---

*Próximo: Capítulo 2, Implementing SLOs, onde a mecânica de medição que antecipei aqui é o assunto oficial.*
