# Teste Técnico — QA Lead Sênior | Seguros

**Candidata:** Jessica Sales Melo
**Cliente:** NFoque — programa *Nova Jornada de Sinistros*
**Data:** 30/09/2026

---

## Suspensão de julgamento antes de qualquer coisa

Dois números convivem neste programa que, juntos, explicam quase todo o resto: **homologação com 100% dos casos verdes** e **~40% de falha intermitente na automação que ninguém confia**. Os dois dizem a mesma coisa — *a frente de QA mede esforço e não risco*. A homologação verde mede que scripts escritos por cada squad rodaram da primeira à última linha dentro da própria fatia, contra simulador/ambiente próprio. Isso não mede o que quebrou em produção. E os 220 cenários E2E falham com frequência alta o bastante para que o time tenha desistido do sinal e aprendido a re-executar até passar — ou seja, o único ativo que *tocava* a jornada real está entregando ruído. É desse lugar que eu parto: não há estratégia, não há rastreabilidade, não há indicador, e o que foi automaticamente é lixo técnico. A boa notícia é que isso é corrigível e barato de corrigir se eu atacar na ordem certa.

---

# 1) Investigar antes de responder ao patrocinador

## 1.1 Como 100% de casos verdes convivem com o que aconteceu

O número "100% verde na homologação" **mede que os pré-casos planejados, rodados por cada squad isoladamente, terminaram como esperado**. Ele não enxerga nada do que quebrou em produção. O que ele não cobre, e que é exatamente o que falhou:

- **A jornada completa entre sistemas.** Cada squad testa o pedaço; ninguém percorre aviso → regulação → aprovação → oficina → pagamento atravessando os serviços e as 11 integrações. Uma falha de orquestração entre serviços é invisível para quem testa uma fatia.
- **O comportamento distribuído real** — ordem de mensagens, redelivery, idempotência, tempos de consumo, concorrência. Nada disso aparece num caso manual single-thread.
- **O contrato real da integração.** Duas integrações foram validadas contra simulador escrito pelo próprio time; simulado ≠ sistema real.
- **Volume e carga** — a virada de onda traz 1.870+ avisos; teste manual é 1, 2 casos.
- **Estado consistente entre sistemas** — que o status de um sinistro concorde entre o core legado, o novo serviço, o antifraude e a mensagem ao cliente.

Ou seja: o 100% verde responde a pergunta *"os meus casos passaram?"* e não a pergunta *"a produção está correta sob essas condições?"*. É esse o primeiro engano que eu vou desmontar na comunicação — o número verde deu **certeza falsa de cobertura**, e esse é o mesmo risco que a IA aplicada sem validação cria (seção 6).

## 1.2 Hipóteses para o quádruplo sintoma

Os quatro sintomas — 240 parados em regulação, 62 com mensagens contraditórias, antifraude com o triplo de casos, pagamento duplicado — têm **fisionomia de problema de integração/mensageria**, não de UI. No Itaú eu lidava com arquitetura orientada a eventos (SNS/SQS/DynamoDB/Lambda); é o mesmo padrão de sintoma.

- **Hipótese A (origem única, mais parcimoniosa): defeito na camada de eventos/orquestração da nova jornada.**
  A nova jornada publica/consome eventos para as integrações (oficina, pagamento, antifraude, canais). Se o consumidor **não é idempotente** (não tem chave de deduplicação), uma mensagem reenviada gera: pagamento em duplicidade (o mesmo sinistro pago duas vezes), dois casos de antifraude para o mesmo sinistro (explicando o triplo de casos = os mesmos sinistros reprocessados), e status contraditórios (aprovado e depois negado = processamento duplicado/fora de ordem na mesma máquina de estados). **Uma única causa explicaria os quatro sintomas.** A parada em regulação poderia ser o componente da mesma falha: um evento de "regulação iniciada" que nunca é publicado, ou um evento consumido mas o estado não persistido de volta — a máquina de estados não avança.
- **Hipótese B (origens independentes):** os 240 parados são um defeito de persistência/máquina de estados na própria journey (não avança, sem erro, sem notificação) — característica de transação que falha silenciosamente; as mensagens contraditórias e o pagamento duplicado são um defeito de idempotência/ordem de mensageria; o antifraude triplicado pode ser *fan-out* mal configurado (um evento despachado 3× para o tópico do antifraude) — relacionado com B-mensageria, mas não com A-parada.

**Como separar o compartilhado do independente:** cruzar os IDs. O primeiro dado a pedir é a interseção entre os 240 parados, os 62 contraditórios, os casos de antifraude considerados "a mais" e o pagamento duplicado. **Se os mesmos números de sinistro aparecem em mais de um sintoma, é origem compartilhada**; se são disjuntos, são defeitos independentes. Esse cruzamento é o primeiro passo do plano e custa um query de SQL (diferencial que eu domino).

## 1.3 Plano de investigação (o que levantar primeiro, com quem, que evidência)

**Ordem dos passos (primeiro o que tem maior poder de discriminar hipótese por menor custo):**

1. **Cruzamento por chave (SQL), hoje.** Pedir acesso de leitura ao store da nova jornada e ao log de mensageria. Query que cruza: sinistros parados em regulação × sinistros com mensagens contraditórias × sinistros que passaram pelo antifraude × o registro do pagamento duplicado. Resultado: separa Hipótese A de B com evidência dura.
2. **Logs de mensageria/integração e filas contíguas (DLQ).** Com SRE/Operação (no meu caso, era CloudWatch/Datadog): contar redeliveries por mensagem, leitura de dead-letter queue, payloads rejeitados. Quantifica o triplo de casos: são 3 eventos *diferentes* ou o *mesmo* evento reprocessado 3×? A duplicata de pagamento: existe um segundo attempt registrado com a mesma chave de idempotência? Se sim, consumidor não idempotente = causa raiz provável.
3. **Falar com, na ordem:**
   - a **squad que orquestra a jornada/state machine** (detém os 240 parados — perguntar como o status "regulação" é persistido e o que dispara o avanço);
   - quem **construiu os simuladores das 2 integrações** (perguntar o que o simulador *não* reproduz: tempo, redelivery, formato real de payload, resposta de erro);
   - o **dono da integração de pagamento + o core legado** (idempotência, contrato);
   - o **time de antifraude** (trazer as horas/lotes dos casos manuais e os IDs — cruzar com os eventos);
   - **SRE/Operação** (logs, DLQ, métricas da janela da virada).
4. **Evidência a pedir a quem:**
   - a SRE/Squad: contagem de redelivery, DLQ, timestamps de publicação × consumo;
   - ao antifraude: lista de sinistros e horários dos casos manuais (p/ cruzar com eventos);
   - ao jurídico/financeiro: o número do sinistro e da oficina do pagamento duplicado e se houve *segundo* laço de tentativa no log de pagamentos;
   - a Dev: o modelo da máquina de estados e o mapeamento evento → transição (para os 240 parados).

**Estimativa da dimensão real do estrago (inclusive onda 1):**

- Não parar nos 1.870/240 conhecidos. Dimensionar por amostragem ativa: query em produção por **máquinas de estado abertas/além do tempo esperado**, **mensagens com contagem de redelivery acima de limite**, **profundidade de DLQ**, **transações de pagamento com tentativa em duplicidade**, **same-sinistro processado >1× pelo antifraude**. Isso dá a *população* afetada, não só a reportada.
- **Onda 1 — o dado não existe, e eu não vou inventar um número.** Não há registo de defeitos e ninguém sabe quantos escaparam. Minhas opções: (a) dizer ao diretor que o número é *inauditável retroativamente* — honestidade de líder; (b) reconstruir um **proxy indireto**: incidentes/tickets de suporte no período da onda 1, chamados de service desk sobre sinistro, volume atípico de antifraude/correção manual naquela janela, e o "conhecimento tribal" cruzado entre os analistas. (c) O que importa mesmo é **meter hoje** o escape rate a partir daquela data; reconstrução histórica é esforço com retorno limitado. Vou dizer: *"não tenho como precificar com precisão o que escapou na onda 1, e prefiro te dar isso do que um número redondinho falso; o que posso fazer é te dar a dimensão da onda 2, que é a que está viva, e começar a medir escape de agora em diante."* Se insistirem em número, entrego o proxy com incerteza explícita, nunca como verdade.

## 1.4 Resposta ao diretor: onda 3 em seis semanas

**Minha recomendação: *segue, mas somente com condições verificáveis em portão explícito — e não segue no modelo atual.*** Não é um no-go absoluto (o prazo e o negócio são reais), mas é um não ao "seguir como está". O que os dados que já tenho (240 parados na janela inicial = ~13% de deadlock) dizem é que **liberar a maior onda — com pagamento a prestadores — sobre o mesmo processo que produziu a duplicata que já chegou ao jurídico é aceitar risco financeiro e regulatório que o programa não pode pagar duas vezes.**

**Condições para a onda 3 seguir (todas verificáveis, em tabela de portão):**

| # | Condição | Como verifico | Riscos que estou aceitando se permitir seguir |
|---|---|---|---|
| C1 | Causa raiz da onda 2 confirmada e corrigida, com a verificação de que os 4 sintomas não reincidem (idempotência testada, DLQ zerada) | Regressão dirigida + canário em produção com os cenários exatos dos sintomas | Reincidência de duplicatas/paradas; novo caso jurídico |
| C2 | Teste de jornada completa entre sistemas (não existe hoje) criado e **verde estável** sobre a onda 3, incluindo o fluxo de pagamento a prestadores | Novo caso E2E ponta-a-ponta, N execuções limpas consecutivas | Liberar pagamento sem percorrer a jornada inteira = repetir o erro que gerou a duplicata |
| C3 | As 2 integrações sem homologação ganham defesa: contrato real + canário + monitoração de divergência (seção 2.3) | Contratos + canary métricas para os 2 fornecedores | Divergência silenciosa entre simulador e sistema real |
| C4 | Portões objetivos de homologação no lugar (entrada/saída em dados, não aval verbal) e critério de aceite escrito por história | Matriz de cobertura por risco assinada; segurança em pagamento | Pressão de prazo volta a cortar regressão por conversa |
| C5 | Gestão de defeitos numa ferramenta única e rastreada | Todo bug saiu do canal para o tracker; DLQ/estado aferido | Defeitos continuarão "se perdendo no canal" |

**O que me faria mudar de posição (para o hábil / para o não):**

- **Mudaria para seguir antes de 6 semanas** se C1 e C2 estiverem verdes *antes* do prazo e o canário de produção da onda 2 se estabilizar sem recorrência por ~2 semanas de tráfego real — aí eu passo a apoiar, pois as condições que mitiga o risco central já foram atendidas.
- **Mudaria para NÃO segue (no-go duro)** se, ao fechar as 6 semanas, C1 (causa raiz) ainda estiver aberta ou C2 (jornada ponta-a-ponta) se mostrar impossível de estabilizar a tempo — porque aí liberar pagamento a prestadores é liberar o único fluxo que já provou produzir perda financeira, sem rede.

Deixo claro a ele que minha função aqui é a de **guardião de portão**: eu posso acelerar quando as condições estiverem boas, e vou atrasar com a mesma energia quando não estiverem — e que cada condição tem *data de vencimento* minha, não hormonal de cronograma, para ele não depender de "aval verbal".

---

# 2) Estratégia de qualidade e plano de testes

## 2.1 Níveis de teste, cobertura, executor, momento

| Nível | O que cobre | Quem executa | Quando | Observação específica deste programa |
|---|---|---|---|---|
| **Unitário** | Lógica do serviço individual | Devs (in-sprint) | A cada PR, no pipeline | Não é responsabilidade QA, mas é portão — código novo entra vermelho? |
| **Contrato entre microsserviços + integrações** | Compatibilidade de schema/contrato entre os 11 serviços e as 11 integrações (incluindo o core legado) | Devs + QA (contratos) | A cada mudança de contrato, no CI | **NOVO**. É o que pega divergência que o "simulador do time" mascara. Versionar contrato. |
| **Integração (por serviço/integração)** | Cada serviço contra cada integração pública (oficinas, prestadores, antifraude, pagamento, WhatsApp, portal, app, regulatório, core) | QA | Por release, antes da homologação | As 2 sem homolog: ver 2.3 |
| **Jornada completa entre sistemas (E2E transversal)** | A viagem inteira: aviso → regulação → aprovação/negação → oficina/prestador → pagamento, atravessando todos os serviços + integrações (com simulador onde não há homolog) | QA (especialista) | Antes de cada onda e como regressão dirigida | **O teste que não existe e é a causa-tipo do incidente.** Cria no *modo contínuo*: mesmo com simulação de perna, ele pega ordem, idempotência e valor. |
| **Manual exploratório + UAT/business** | Jornadas complexas e de julgamento, negócio negando/aprovando caso a caso | QA + negócio | Durante homologação | Manual é onde mora o julgamento regulatório de um sinistro real — não se automatiza isso |
| **Não funcional (carga na virada de onda, segurança em pagamento, disponibilidade da jornada)** | Volume de virada, auth de integrações de pagamento, alta-disponibilidade | QA + SRE | Antes de cada onda / onda 3 | **NOVO e obrigatório na onda 3** (ver diferenciais) |

## 2.2 Critérios objetivos de entrada/saída (para discutir 2 vs 5 dias com dados)

**Entrada de um ciclo de testes/homologação de onda:**
1. Ambiente estável (sem incidente aberto de ambiente bloqueante) e dado de teste disponível com massa definida.
2. Smoke automatizado da jornada **verde estável** (N execuções limpas).
3. Nenhum defeito **Crítico** aberto no escopo; **Alto** com plano de correção datado.
4. Contratos das integrações afetadas disponíveis e validados.
5. Critério de aceite por história escrito e aceito antes de iniciar (não "funcionou na demo").

**Saída (definição de pronto de homologação):**
1. 100% dos casos de **risco alto/crítico** no escopo executados e verdes *sem reexecução de mascaramento*.
2. Defeitos abertos ≤ fronteira por severidade (ex.: 0 crítico/alto em escopo; médios com data; baixos em backlog prioritizado).
3. Jornada ponta-a-ponta entre sistemas verde no ambiente de homologação/integrado.
4. Idade de defeitos dentro do SLA por severidade.
5. **Assinatura formal escrita** do responsável de negócio/negócio+arquitetura na matriz de cobertura por risco — substitui qualquer aval verbal.

**Redução 5 → 2 dias como objeto de decisão, não de pressão:** a pergunta deixa de ser "conseguimos em 2 dias?" e vira "**o que estamos dispostos a não testar, em que faixa de risco, e quem assina isso**". Eu trago a matriz de casos classificada por risco; o corte de 5 para 2 dias corta por *faixa de risco descartada*, com o residual documentado e assinado. Se a pressão mandar cortar, corta a faixa de menor risco explícita — nunca a duração de uma regressão com escopo intacto. E o custo desse corte fica visível: se um defeito escapa de uma faixa que foi cortada, a resposta ao patrocinador é "esta foi a decisão de 2 dias — eis o recibo assinado".

## 2.3 As duas integrações sem ambiente de homologação

**Opções:**
1. **Conseguir homologação real do fornecedor** (contatar vendor, ambiente de staging do parceiro). Melhor, mas depende de terceiro e de prazo.
2. **Contrato real + replay de tráfego**: validar o contrato e *replay* de payloads reais de produção capturados contra o serviço, junto com **canário em produção** em lote pequeno + monitoração de divergência.
3. **Simulador + testes de contrato**: continuar com simulador, mas blindá-lo com contract tests e *producer-consumer* — o simulador do time já funcionou como "tudo verde" que mentiu, então deixa de ser a única defesa.
4. **Shadow/canário em produção** com rollback automático e monitoração da divergência.

**Recomendação:** para as duas integrações (uma delas é pagamento, dado o incidente — **é a que mais importa**), recomendo **combinação: contrato real + simulador blindado para o dia a dia + canário em produção de baixo volume + monitoração de divergência e correlação de IDs**. Não libero a perna de pagamento real-valor em onda sem alguma forma de espelho/homolog do fornecedor.

**Risco residual da minha recomendação (é honesto declarar):** simulador contratado ≠ comportamento temporal do real (tempo de resposta, redelivery, formato real de payload de erro). O residual que permanece é que **divergência de comportamento só aparece em produção, e pode aparecer primeiro como incidente**. Mitigação que reduz, não elimina: canário pequeno + monitoração ativa dos fluxos de pagamento. E um go/no-go explícito no toque real-valor da integração de pagamento.

## 2.4 O que cobrir com profundidade vs o que cobre-se superficial/nenhum

**Critério (por risco, score):** `Score = (severidade da falha) × (verossimilhança dado a mudança) × (impacto de negócio/regulatório) × (exposição de integração e de dinheiro)`. Testar com profundidade o que move dinheiro, afeta cliente final, tem cap regulatório, ou cruza mais integrações. Onde o custo de testar excede o custo esperado do defeito, cobrir superficial ou declarar não-cobertura **com assinatura do risco**.

**Aplicado a este programa (nomeando):**
- **Profundidade:** fluxo de pagamento a prestadores/oficinas (onda 3), aprovação/negação e mensagens ao cliente, regulação (deadlock dos 240), integrações de pagamento e antifraude, envios regulatórios, correção de defeito da onda 2 (idempotência/ordem). Tudo que toca em dinheiro ou cliente.
- **Cobertura superficial:** telas de consulta/relatórios de baixo uso, mensagens de borda de WhatsApp, textos de notificação não-regulatórios, perfis de acesso administrativo não exposto.
- **Não coberto (declarado e assinado):** cenários de cancelamento/estorno em volume, as integrações expostas que não movem dinheiro, backlog de cosmética. Assumo explicitamente esse risco — é a restrição real de gente/tempo, e o preço de "testar tudo" é não testar nada direito.

---

# 3) Governança: defeitos, aceite, indicadores

## 3.1 Fluxo de gestão de defeitos

**Nascimento:** qualquer pessoa (QA, dev, negócio, suporte) detecta e registra. **Regra de ouro: "se não está na ferramenta, não é defeito."** O canal continua existindo como porta de entrada, mas **toda** menção no canal é triada no mesmo dia por mim/analista e convertida em registro estruturado — nada fica só em excesso de conversa.

**Informação que carrega (obrigatória):** título claro, passos de reprodução, esperado vs atual, ambiente (homolog/prod), **severidade**, **prioridade**, evidência (log/screenshot/ID de sinistro), história/requisito vinculado, e o impacto de negócio. Lanço os campos na ferramenta para que rastreabilidade e métricas existam — esse é o dado que alimenta o escape rate da seção 3.3.

**Classificação:**
- **Severidade** (impacto técnico): Crítico (bloqueia jornada / perda financeira / violação regulatória / deadlock — ex.: os 240, a duplicata), Alta (funcionalidade principal degradada sem workaround), Média (função acessória, com workaround), Baixa (cosmética/usabilidade).
- **Prioridade** (urgência de negócio + agenda): define a *ordem* de correção, decidida no triage.

**Decisão do que entra na correção (triagem):** um **triage diário de 15 minutos**, QA Lead + tech lead da squad + (se dinheiro/regulatório) negócio. Decisões: corrigir agora / agendar / backlog / aceitar como "não-defeito" (com justificativa registrada). **Quem decide:** QA não decide sozinho — decide o *conjunto* com dev; QA é quem **susta o portão** e registra a decisão. Nada é corrigido "porque deu na review".

**Acordo de prazo por severidade (SLA):**
- **Crítico:** trava a linha; correção em até ~4h úteis; **bloqueia homologação/release** até resolver.
- **Alta:** correção ainda no sprint corrente ou +1; precisa do canário/monitoração.
- **Média:** próximo sprint, com data.
- **Baixa:** backlog priorizado, sem data garantida.

**Conviver com o canal:** o canal é a *detecção precoce* e informal; a ferramenta é o *registro formal*. Implantarei que ninguém considera sabido um problema que esteja só no chat — o time amarra "relatar no canal" → "abrir no tracker" como dois gestos do mesmo ato. E há regra de **contenção**: a duplicata/parada não é "resolvida" quando a última resposta do chat for boa — só quando o registro no tracker estiver fechado com causa raiz.

## 3.2 Critérios de aceite/pronto no lugar de "funcionou na demo"

**Definition of Ready (antes de pegar a história):**
- Critério de aceite **escrito, testável e sem ambiguidade** (quando? com que dado? que resposta?).
- Integrações envolvidas nomeadas e com **contrato disponível**.
- Massa de dado necessária identificada (e, se de segurados, reais → mascarada, seção 5).
- Requisitos não-funcionais aplicáveis (se toca em carga/segurança) anotados.
- Método de teste (nível + env) declarado.

**Definition of Done (para liberar a história):**
- Código revisado + unitário/contrato verdes no CI.
- Critérios de aceite demonstrados **contra evidência (teste automatizado ou execução rastreada)**, não por demo.
- Teste de integração/contrato da trilha afetada verde.
- **Zero defeito Crítico/Alto aberto na história** com justificativa se houver residual.
- Dado/valores validados via SQL (assert de estado, não só tela).
- Rollback/retorno considerado para a mudança (na esteira do 5.2).

A "demo funcionou" deixa de ser critério: vira demonstração de *resultado*, acompanhada do registro de execução e dos asserts.

## 3.3 Quatro a seis indicadores

| # | Indicador | Fórmula | Fonte | Frequência | Público | Decisão concreta que sustenta |
|---|---|---|---|---|---|---|
| 1 | **Defeitos escapados (Escape Rate)** | `(defeitos de produção)/(defeitos totais do período)` | Tracker/Jira | Semanal | Diretor + gestores + squads | Decidir **go/no-go da onda e se o portão de homologação foi honesto**; é a mais ligada a este incidente |
| 2 | **Cobertura de jornadas críticas entre sistemas** | `(cenários E2E* transversais executados)/(jornadas críticas mapeadas)` | Plano de teste + execução | Por onda | QA + PM | Decide se a onda 3 tem rede para **liberar pagamento/mensageria** |
| 3 | **Taxa de estabilidade da automação (flaky)** | `(falhas de cenários por instabilidade)/(execuções da suite)` | Suite + Allure/JUnit | Semanal | Time QA/Dev | Decide o que **barra release** (se > limiar, trava) e o que se **descarta/estabiliza** do lote de 220 |
| 4 | **Idade de defeitos abertos por severidade vs SLA** | `média de idade por severidade; % fora do SLA` | Tracker | Diário | QA + squads + PM | Decide prioridade da correção e **se segura o release**; mostra riscos que "moram no canal" |
| 5 | **Tempo médio até detecção (MTTD) de incidente em produção** | `tempo entre a falha ocorrer e alertar/a identificação` | Monitoração + incidentes | Semanal | SRE + QA | Decide se a **monitoração/observabilidade está pegando** (os 240 ficaram mudos — sem esse número repetimos) |
| 6 | **Contenção de defeitos por fase** | `(defeitos achados antes da homologação)/(total)` | Tracker | Por onda | QA Lead | Decide se o *shift-left* (teste no refinamento, contrato no CI) **está funcionando** ou se ainda se descobre tudo tarde |

## 3.4 Dois indicadores que eu NÃO adotaria e por quê

1. **"Número de casos de teste executados / volume de casos"** — métrica de vaidade: premia volume, não risco. É fácil de inflar (mil forenses fáceis), não decide nada, e reproduz exatamente o erro do "100% verde" que já mentiu aqui.
2. **"Cobertura de código percentual como alvo/portão"** — dá falsa sensação de cobertura: 90% de linhas num serviço não diz nada sobre ordem de mensagens, idempotência ou a jornada entre sistemas — é onde este incidente nasceu. Line coverage como meta estimula *gaming* (testar código fácil e verde) em vez de risco.

---

# 4) A suíte automatizada em que ninguém confia

## 4.1 O que fazer com os 220 cenários

**Diagnóstico antes de decidir — triagem por cluster de falha, não decisão às cegas.** Rodo a suite hoje, capturo as falhas reais e as **clasifico**:

- **Falha ambiental do teste** (espera fraca, seletor quebrado, dado de teste compartilhado/concorrente, ambiente instável) → o *teste* é o defeito, não o produto. Tendem a ser a maioria dos 40%.
- **Falha genuína de produto/contrato** mascarada pelo "re-executar até passar" → o time pode estar *engolindo defeito real*.

**Decisão por cenário (combinação recuperar + descartar graciosamente):**
- **Manter** (recuperar): cenários de **jornada crítica** que, uma vez estabilizados, têm sinal direto no risco de negócio (pagamento, regulação, aprovação/mensagem).
- **Quarentena** (estabilizar sob custo): valor altos mas flaky — ganham isolamento de dado (*unique IDs*, massa própria), esperas corretas, e **definição de estável = N execuções limpas consecutivas sem reexecução**.
- **Descartar/reescrever**: redundantes, sem valor de risco, ou impossíveis de estabilizar sem reescrita completa e baixo retorno — custo de manter supera o benefício.

**Justificativa pelo que mais importa (confiança):** o ativo vale não pelas linhas, mas pelo **sinal**. 220 cenários que mentem 40% valem *menos que 30 que nunca mentem*. O esforço de recuperação é direcionado só ao que sustenta decisão; desistir do resto é recuperar *confiança* — o efeito colateral que ninguém mede mas que quebrou este time (já desistiram do sinal). **Regra anti-cultura:** reexecutar é permitido *uma* vez e **sempre registrado e investigado**; reexecutar silenciosamente até passar = falha de processo, não de sorte.

## 4.2 Estratégia de automação dali em diante

- **Pirâmide correta, em vez de "tudo E2E":**
  - **Unitário + contrato** (base): rápido, em todo PR, **devs executam** no CI. É o volume barato que pega divergência de contrato.
  - **Integração/API**: em CI por deploy, quebrando contrato das 11 integrações. **QA + devs**.
  - **Poucos E2E de jornada crítica** (topo, raros): execução em release/canário, **QA especialista**. Menos cenários, mais sinal, zero flaky tolerado.
- **O que fica manual/exploratório:** jornadas de julgamento (aprovação negocial de um sinistro real, casos únicos de negócio), UX e cenários negativos por empresa específica — onde automação daria falso sinal e o manual dá visão. **Nunca automatizar o que flakiness corromperia.**
- **Automatizar o que:** (1) é repetível, (2) tem alto risco, (3) tem custo baixo de falso negativo, (4) não depende de julgamento. Não automatizar o julgamento.

## 4.3 O que barra vs o que informa — e como fazer respeitarem

**Barra release (bloqueia — RED é RED, sem reexecução):**
- Unitário/contrato vermelho no CI.
- **E2E da jornada crítica** (inclusive a transversal) vermelho.
- Contrato da integração que move dinheiro (pagamento) vermelho.
- Segurança em pagamento falhou.
- **Estabilidade: flaky acima do limiar** decide-se na mesma linha — se a suite não é estável, o portão nem roda.

**Informa (não bloqueia sozinho):** cobertura % (informativo), tendências de performance (informa, não trava), regressão completa de faixa não-crítica, código cobertura, achados exploratórios.

**Como fazer a regra ser respeitada num programa que já cortou regressão por pressão:**
1. **Colocar no processo escrito, não na autoridade.** O portão vira parte do Definition of Done e do release checklist formal, não um "QA bravo".
2. **Prefixo com o sponsor.** O diretor assina que RED de pagamento/segurança segura a liberação — ele mesmo passa a ser o guardião do portão, o que inverte o incentivo de "atropelar QA".
3. **Conversa sobre o dado, não sobre poder.** Quando um RED aparecer, discute-se o *sinal específico* (que cenário, que impacto) e não "por que o QA está travando". A conversa deixa de ser pessoal.
4. **Custo da violação visível.** Cada RED liberado à força fica registrado com quem decidiu — e quando escapar, a resposta ao patrocinador é o recibo da decisão. A onda 2 é o exemplo vivo do custo de não ter portão.

---

# 5) Time, iniciativa além do escopo e comunicação

## 5.1 Organização dos 3 analistas + a 4ª squad (dados)

O modelo atual "um analista por squad-de-dev" dilui 3 pessoas em 4 frentes e deixa o ponto de maior risco — **dados/processamento** (os 240 parados vivem aqui) — sem qualidade alguma. Reorganizo em **capítulo/centro de excelência**, eu como QA Lead respondendo pela estratégia, teste, governance e indicadores de todo o programa, com os 3 distribuídos por risco, não por squad:

- **Analista 1 — Especialista de jornada/integração/API (e E2E transversal).** Dona das jornadas ponta-a-ponta entre sistemas (as que pegaram o incidente) e dos contratos/11 integrações. É a posição mais crítica e a razão central do que quebrou.
- **Analista 2 — Embutida na squad de maior risco da onda 3 (pagamento/prestadores + regulatório).** In-sprint, do refinamento ao DoD, com foco em dinheiro e conformidade.
- **Analista 3 — Rota/cobertura das demais squads + a squad de dados.** Cobre as duas frentes restantes por prioridade de risco e responde à squad de dados (estado do sinistro, qualidade da fonte), com apoio de automação/contrato.

**Qualidade desde o início, não no fim da esteira:** os 3 entram no **refinamento** (3 Amigos com dev e produto), análise de **ambiguidade de requisito** antes de escrever caso, e participação em **revisão de arquitetura** de integração — não só "testar no fim". A prevenção vive no requisito; a detecção que salvou aqui veio cedo demais.

**Ampliar o time?** Recomendo **+1** — especificamente automação/engenharia de testes de integração-API, pareado com a squad de dados. **Argumento para o diretor, em termos dele:** o incidente custou 240 sinistros presos, 62 clientes recebendo mensagens contraditórias e **uma duplicata de valor alto que já está no jurídico**. Mais uma pessoa custa uma fração de uma duplicata dessas, e a onda 3 é a maior, com dinheiro a prestadores. Trago como mitigação de *risco financeiro e regulatório*, não conforto de time.

## 5.2 Três melhorias que eu conduziria por iniciativa própria (em ordem)

Nada disso foi pedido. Eu faria, por **impacto × urgência × dependência**:

1. **Rastreabilidade requisito → caso de teste → execução → defeito, numa ferramenta única** (Jira + Zephyr ou similar). *Por quê:* é o **facilitador de tudo** — sem ele não há escape rate, não há portão honesto, não há resposta ao diretor com número. É a peça que desbloqueia as seções 1 e 3 e custa configuração, não contratação. Maior multiplicador.
2. **Governança de massa de teste e dados de segurados (LGPD).** Hoje a massa é criada à mão com **dados reais de segurados**. Isso é risco legal e reputacional que pode custar mais que o programa inteiro (multa de LGPD). Não é "nice to have": é não-negociável assim que há dado pessoal. Implantar mascaramento/dados sintéticos e regra de nunca colar dado real de segurado em prompt de IA (seção 6).
3. **Plano de rollback/retorno da virada de onda.** O incidente provou que virada de onda falha e **não havia rede**. Antes de qualquer onda nova — não dá para voltar? — criar mecânica de retorno/rollback + critério de acionamento + monitoração de divergência. É proteção operacional imediata contra a classe de falha que já queimou o programa uma vez.

Escolhi essas três porque todas tiram risco **hoje** sem esperar 6 semanas, e criam a infraestrutura de governança que as seções 1–4 precisam. (Deixaria de fora, conscientemente, a documentação das regras de sinistro — importante, mas de retorno mais lento que as três acima e que não paga a onda 3.)

## 5.3 Comunicação ao diretor patrocinador (até 6 linhas)

> Diretor, preciso ser direto: a homologação da onda 2 marcou 100% verde, mas ela não testa a jornada entre sistemas — e foi aí que quebramos. Tenho 240 sinistros presos, 62 clientes com mensagem contraditória e uma duplicata que já está no jurídico; causa raiz em investigação, mas sem condição de liberar a onda 3 no mesmo modelo. A onda 3 pode seguir em 6 semanas, desde que fechemos 5 portões objetivos (causa raiz corrigida, teste de jornada ponta-a-ponta verde estável, as 2 integrações sem homolog blindadas, critérios de aceite escritos e gestão de defeitos numa ferramenta). Quero acelerar quando as condições estiverem boas — e vou segurar com a mesma convicção quando não estiverem. Proponho nos falarmos na terça para validarmos o plano e os prazos.

---

# 6) Uso de IA no meu dia a dia

**Onde eu me apoiaria, neste programa — e o que eu pediria de fato:**

- **Análise de requisitos (ambiguidade e risco):** pedir a um Claude/Copilot/Gemini para "localizar aceite ambíguo, lacunas de borda/negativa, menções a regulatório ou a integração sem contrato explícito" em histórias. **Onde confio e onde valido:** confio em gerar *perguntas* e *hipóteses* (o que me faz enxergar o que eu não vi), mas **valido cada sugestão com negócio** — IA não conhece a semântica de sinistro desta casa.
- **Geração/revisão de casos de teste:** gerar Gherkin e casos negativos a partir do aceite — **confio no esqueleto, valido contra o contrato real** e contra a jornada. Nunca gero "muitos casos" só para parecer coberto.
- **Análise de padrões em base de defeitos:** pedir clusters de defeito (por componente, severidade, sintoma) em cima de **dado agregado e anonimizado** — idem o risco: já sei a lição de que "muitos testes" ≠ cobertura. **Confio na clusterização, valido as conclusões com o time**.
- **Investigação de cenários instáveis de automação (os 220/40%):** dar *log de falha* (sem dado de segurado) e pedir hipóteses de causa (espera, dado compartilhado). **Confio em gerar direções de investigação, valido rodando o experimento** — IA não é quem decide o que o cenário "deve esperar".
- **Criação de massa de teste:** geração **sintética**, nunca real. Melhor caso de uso seguro.
- **Documentação e comunicação ao patrocinador:** esboço inicial; **reviso integralmente** — a voz e a decisão final (como na seção 5.3) são minhas.

**Onde a IA cria falsa sensação de cobertura:** o mesmo erro do "100% verde". LLM que **gera muitos casos testáveis, bem formatados, e que "passam"** produz a ilusão de que a jornada está coberta quando, na verdade, nenhum percorre a integração entre sistemas de verdade. Geração em volume é a nova "demo que funcionou" — precisa de portão duplo: vínculo com critério de aceite real + execução que escape ao "verde por reexecução".

**Riscos que eu observaria (e como me protejo):**
- **Alucinação:** IA "sabe" comportamento de integração que não existe na jornada. Regra: nada que ela produz entra num portão sem rastreabilidade ou execução real.
- **LGPD / dados de segurados em prompt:** colar sinistro/política de segurado real em um modelo externo é violação. **Regra dura: nunca colo dado pessoal de segurado em prompt; massa é sintética/mascarada; os 11 integrações têm IDs e payloads que tratamos como sensíveis.**
- **Dependência/atrofiamento do risco:** time que delega o "por que quebraria?" à IA deixa de pensar sobre o próprio risco. Mantenho a IA como ferramenta de *gerar e questionar*, nunca de *assinar*. A pergunta de risco é a do QA senior, ancorada em SQL, contrato e jornada real.

---

## Síntese da decisão ao patrocinador

**Resposta direta: a onda 3 não segue "como está". Segue em seis semanas sob cinco condições verificáveis, com o portão de pagamento e de jornada ponta-a-ponta como inegociáveis, ou não segue.** O 100% verde original não contava histórias de integração; a nova defesa sim. Eu assumo a posse dos portões, das datas de vencimento e dos recibos escritos de cada decisão de risco — e trago os números (escape, estabilidade da automação, cobertura de jornada, idade de defeito, MTTD) para que, daqui para frente, nenhuma decisão de liberação dependa de aval verbal.

---

*Documento elaborado como teste técnico — QA Lead Sênior | Seguros. Todos os valores (1.870/240/62/triplo/duplicata) são os fornecidos no cenário; as decisões assumem que a causa raiz pode ser compartilhada (mensageria/idempotência) ou independente (estado + mensageria), a ser resolvida pelo cruzamento de IDs no passo 1 da investigação.*