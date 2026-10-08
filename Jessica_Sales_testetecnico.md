# Teste Técnico — QA Lead Sênior | Seguros

**Candidata:** Jessica Sales Melo  
**Cliente:** NFoque — *Nova Jornada de Sinistros*  
**Nível:** Sênior (QA Lead)  
**Data:** 24/09/2026

---

## Objetivo

Avaliar a capacidade do candidato de estruturar uma frente de Quality Assurance em um programa crítico e de grande porte, analisar risco sob pressão de cronograma e defender decisões de estratégia, governança, indicadores e liderança — este teste avalia raciocínio, priorização e comunicação, e não exige escrita de código.

## Informações Gerais

| Tempo estimado | Prazo para entrega |
|---|---|
| Até 2 horas | 2 dias corridos |

## Cenário

**Domínio de negócio: Seguros — programa de reconstrução da jornada de sinistros**

Não é esperado conhecimento prévio do mercado segurador. O contexto necessário está todo descrito aqui: quando o cliente sofre um sinistro (uma batida, um vazamento, um roubo), ele registra um aviso; a seguradora analisa o caso — essa etapa se chama regulação —, aprova ou nega, autoriza o reparo em uma oficina ou prestador credenciado e, por fim, paga. Cada uma dessas etapas passa por sistemas diferentes.

Você foi contratado como QA Lead do programa Nova Jornada de Sinistros, de uma seguradora com aproximadamente 2,1 milhões de apólices nos ramos automóvel e residencial. O programa tem quatorze meses de duração e substitui, em ondas, a jornada de sinistros que hoje roda no core legado. A nova jornada é construída em microsserviços e conversa com onze integrações: oficinas credenciadas, prestadores de assistência, motor de antifraude, meio de pagamento, o próprio core legado — que continua sendo a fonte da apólice e da cobertura —, portal do cliente, aplicativo, atendimento por WhatsApp e os envios regulatórios.

O programa tem quatro squads e cerca de 34 pessoas. A frente de QA, na prática, não existe: há três analistas de qualidade, um alocado em cada squad de desenvolvimento, sem coordenação entre eles e sem nenhum papel na quarta squad. Não há estratégia de testes formalizada, não há plano de testes, não há nenhum indicador de qualidade. O que existe é uma planilha de casos de teste mantida por uma pessoa e uma suíte automatizada com 220 cenários de ponta a ponta que falha de forma intermitente em cerca de 40% das execuções — o time já se acostumou a mandar rodar de novo até passar. Foi para estruturar essa frente do zero que você foi contratado.

INCIDENTE: a onda 2 entrou em produção na quinta-feira passada e, nos três dias seguintes, 1.870 avisos de sinistro passaram pelo novo fluxo. O resultado: 240 sinistros pararam no status de regulação e não avançam — não dão erro, não notificam ninguém, simplesmente ficam parados; 62 clientes receberam uma mensagem de sinistro aprovado e, em seguida, outra de sinistro negado; o time de antifraude está recebendo o triplo de casos para análise manual em relação ao previsto, sem explicação; e um pagamento foi feito em duplicidade para uma oficina credenciada, em valor alto, caso que já chegou ao jurídico.

O que se sabe até agora, sem conclusão fechada. A homologação da onda 2 terminou com 100% dos casos planejados executados e todos verdes. O ciclo de regressão foi reduzido de cinco para dois dias por pressão de cronograma, com aval verbal do gerente do programa. Duas das onze integrações não possuem ambiente de homologação disponível e foram validadas contra simuladores construídos pelo próprio time de desenvolvimento. Não existe nenhum teste que percorra a jornada completa entre sistemas: cada squad testa o seu pedaço. O critério de aceite das histórias, na prática, é a demonstração ter funcionado na review. E não há gestão estruturada de defeitos — bugs são relatados no canal do time, alguns viram cartão no board, outros se perdem na conversa; ninguém sabe dizer quantos defeitos escaparam para produção na onda 1.

A pergunta que está na sua mesa: a onda 3, que é a maior de todas e inclui o pagamento a prestadores, está planejada para daqui a seis semanas. O patrocinador do programa, um diretor, quer saber de você, por escrito, se pode seguir como está.

## Desafio Técnico

Este teste não pede código. Entregue um documento de decisão: o que você faria, em que ordem e por quê. Não descreva teoria de qualidade de forma genérica — ancore cada resposta neste programa, com este time, estas integrações e este prazo. Dizer o que você deixaria de fora, e assumir o risco disso, vale tanto quanto dizer o que faria. O tempo sugerido está entre parênteses em cada item.

## 1) Investigar antes de responder ao patrocinador (~28 min)

Minha primeira conclusão é que o resultado de “100% dos casos planejados executados e aprovados” não representa 100% de cobertura dos testes. Esse indicador mede os casos executados em relação aos casos planejados, mas não demonstra, por exemplo, se os cenários planejados representavam os principais riscos do negócio, se contemplavam fluxos entre múltiplos sistemas, se as regras de exceção foram validadas, se duplicidade e retentativas foram consideradas, se as transições de estado foram verificadas, se houve validação de consistência de dados e se eventos e notificações foram validados.

**Referente aos 240 sinistros parados em regulação, eu levantaria algumas hipóteses de causa:**

1. Se o evento necessário para avançar o estado do sinistro está sendo publicado
2. Se o evento é publicado, mas não está sendo consumido
3. Se o consumidor descartou ou ignorou o evento
4. Se houve falha ou indisponibilidade de algum componente da arquitetura assíncrona, ou processamento do evento diferente do esperado
5. Se houve retry e qual foi a quantidade de tentativas
6. Se alguma mensagem foi direcionada para DLQ e se existe monitoramento
7. Se houve alteração de contrato de API ou evento entre serviços
8. Se houve problema de idempotência ou ordenação de eventos
9. Se o retorno do antifraude não está provocando a mudança para o status esperado.

O time como um todo precisaria consultar banco de dados, logs distribuídos e filas para acompanhar a timeline desses sinistros do início ao fim.

Nos incidentes de mensagem de aprovado e depois negado, existe uma possível inconsistência entre o estado transacional e a comunicação ao cliente. Eu levantaria hipóteses como:

1. Notificação enviada antes da conclusão definitiva.
2. Processamento assíncrono fora de ordem.
3. Retry de mensagem.
4. Consumers diferentes interpretando estados distintos.
5. Serviço de comunicação utilizando evento incorreto.
6. Eventos duplicados.

**Referente ao aumento de 3x nos casos enviados ao antifraude**

Assim como nos demais incidentes, eu levantaria possíveis causas:

1. Se houve alguma mudança nas regras de elegibilidade
2. Se campos obrigatórios estão chegando nulos
3. Se houve timeout e ele está sendo interpretado como risco.
4. Se existe feature flag ou configuração incorreta
5. Se o ambiente de homologação representa corretamente o comportamento real do motor antifraude
6. Se alguma integração está retornando fallback para análise manual

Os incidentes podem não ter exatamente a mesma causa raiz, mas podem compartilhar uma origem. Eu investigaria problemas no processamento assíncrono entre microsserviços, como duplicidade de eventos, eventos fora de ordem, retry, idempotência e observabilidade insuficiente. Em uma arquitetura orientada a eventos, os fluxos precisam ser testados também como jornada integradaneste cenário, aparentemente foram validados de forma isolada.

Para iniciar o plano de investigação, eu criaria um war room envolvendo QA, desenvolvimento, arquitetura, produto, responsável pelo core, antifraude, pagamentos, SRE/DevOps e negócio de sinistros . Reproduzir tudo manualmente no início seria inviável, eu começaria pela coleta de evidências olhando os logs, IDs dos sinistros, correlation IDs, eventos publicados e consumidos, DLQs, traces, versões dos serviços e alterações recentes. O objetivo é preservar o máximo de evidência possível antes de uma nova implantação.

Eu criaria grupos separados para cada incidente, evitando misturar os temas, e durante a investigação procuraria interseções entre eles. Também compararia produção e homologação se foi utilizado mock, ele reproduzia erros? Era possível simular timeout, duplicidade, indisponibilidade e comportamento assíncrono? Nem todo erro de produção será necessariamente reproduzível em ambiente de teste, por isso a limitação precisa ser registrada e o risco deve ficar explícito.

Como estimar os defeitos escapados da onda 1? Como não existe histórico estruturado, eu faria uma retrospectiva usando incidentes, Teams/Slack ou ferramenta de comunicação utilizada, cards criados depois da implantação, deployments emergenciais, acionamentos de rollback, commits de hotfix e problemas reportados pelo negócio.

Como seguir com a onda 3? Eu manteria as seis semanas como objetivo, mas não assumiria hoje que a liberação ocorrerá necessariamente nesse prazo. A liberação dependeria de condições objetivas, como:

- identificar a causa raiz do pagamento duplicado
- validar o mecanismo de idempotência nos pagamentos
- compreender a causa dos sinistros presos
- ter testes E2E da jornada crítica funcionando
- validar os contratos das integrações críticas
- garantir cobertura dos cenários críticos de antifraude
- não ter defeitos Severity 1 abertos
- caso ainda existam defeitos Severity 2, liberar somente mediante aceite formal de risco
- ter um plano de rollback
- ter observabilidade da onda definida

Se esses pontos não forem atendidos, minha recomendação seria postergar a onda, porque liberar pagamentos ainda sujeitos a duplicidade pode gerar impacto financeiro, jurídico e operacional significativamente maior.

### Dimensionamento do impacto com SQL e evidências de produção

Para dimensionar o impacto dos incidentes, eu trabalharia em conjunto com o time desenvolvimento, dados ou o responsável pelos bancos integrações, solicitando consultas que permitissem identificar a quantidade de sinistros afetados e os padrões envolvidos.

Como QA Lead, o meu papel seria definir quais evidências e informações precisariam ser levantadas para apoiar na investigação. As consultas técnicas ao banco de dados seriam executadas pelo time de desenvolvimento ou dados, enquanto eu utilizaria os resultados para dimensionar o impacto, identificar padrões, definir prioridades de teste.

Eu tenho conhecimento de SQL aplicado ao contexto de QA e validação de dados, mas nesse cenário eu não assumiria como responsabilidade do QA Lead executar consultas complexas diretamente em produção.

## 2) Estratégia de qualidade e plano de testes (~25 min)

A estratégia seria baseada em risco, deixando claro que QA não deve ser o único responsável por encontrar defeitos. A qualidade precisa existir desde o requisito até a produção.

Os níveis de teste seriam:

1. Testes unitários realizados pelo time de desenvolvimento e testes de componente realizados por desenvolvimento e QA
2. Testes de API realizados pelo QA e testes de integração
3. Testes E2E da jornada
4. Testes de regressão separados por risco: smoke test em todo deploy, regressão crítica para os fluxos de maior risco antes da release e regressão ampliada com funcionalidades secundárias antes das ondas.
5. Testes exploratórios para jornadas novas ou regras com alto grau de variabilidade
6. Testes não funcionais, incluindo performance de APIs, filas, processamento em lote, pagamento e segurança, como exposição de dados e autenticação

Para iniciar um ciclo de testes, eu definiria critérios de DOR e DOD.

DOR

- requisitos definidos
- critérios de aceite existentes
- dependências identificadas
- build implantada e estável
- smoke test aprovado
- massa de teste disponível
- integrações necessárias acessíveis ou alternativa formalizada
- casos críticos revisados
- ausência de impedimentos técnicos

DOD

- 100% dos cenários críticos executados;
- 100% dos cenários críticos aprovados;
- zero Severity 1;
- zero Severity 2 sem aceite formal;
- jornada E2E crítica aprovada;
- regressão crítica aprovada;
- taxa máxima de falha de automação acordada;
- observabilidade preparada;
- rollback definido.

Para integrações sem ambiente de homologação, eu avaliaria virtualização de serviço e mocks capazes de reproduzir timeout, erro e indisponibilidade.

Separar a responsabilidade e momento de execução: desenvolvimento executa unitários e componentes a cada mudança, QA e desenvolvimento mantêm testes de API/contrato no CI,  QA conduz integração e exploração durante a sprint, a jornada E2E crítica é executada em ambiente integrado antes de cada onda, negócio participa da homologação dos fluxos críticos com critérios definidos previamente. Testes não funcionais de carga, segurança e disponibilidade são executados antes da liberação das ondas de maior risco e repetidos quando houver mudança relevante de arquitetura.

Critérios de entrada e saída deixam de ser negociados apenas por prazo. Se a regressão precisar cair de cinco para dois dias, a decisão deve ser sustentada pelo risco coberto, suíte crítica selecionada, defeitos abertos, estabilidade da automação e impacto do que ficará sem executar. A redução pode ocorrer, mas o risco remanescente precisa estar explícito e ter aceite formal do responsável pelo negócio/programa.

Para as duas integrações sem ambiente de homologação, eu consideraria alternativas como virtualização de serviço com mocks ou stubs controlados, utilização de sandbox do fornecedor quando disponível, validação de contratos das APIs ou eventos e, em situações específicas, uma validação controlada após a implantação, utilizando feature flag ou liberação gradual quando a arquitetura permitir.

Minha recomendação seria utilizar virtualização de serviços combinada com validação de contrato. Os simuladores não deveriam reproduzir apenas o caminho de sucesso, mas também situações como timeout, indisponibilidade, respostas inválidas, duplicidade, lentidão e códigos de erro.

A implementação técnica dos testes de contrato ficaria principalmente com o time de desenvolvimento. Como QA, eu atuaria junto ao desenvolvimento e arquitetura identificando as integrações críticas, levantando os cenários que precisam ser protegidos e acompanhando os resultados dessas validações.

Mesmo com essas validações, permanece um risco residual, porque mocks e testes de contrato não reproduzem integralmente o comportamento do sistema externo em produção. Por isso, eu registraria esse risco e complementaria a estratégia com smoke pós-release, monitoramento reforçado, feature flag quando possível e plano de rollback.

A priorização dos testes seria baseada no impacto e na probabilidade do risco. Teriam cobertura mais profunda os fluxos de pagamento, mudança de status do sinistro, antifraude, consulta de apólice e cobertura, processamento assíncrono, notificações e a jornada crítica entre os sistemas.

Portal, aplicativo, WhatsApp, oficinas e prestadores teriam cobertura proporcional ao risco de cada funcionalidade. Cenários de baixo impacto, variações visuais e combinações muito raras teriam menor prioridade dentro da janela de seis semanas, com o risco restante registrado e conhecido pelo programa. O E2E crítico não substituiria os testes de API, contrato e componente. Ele existiria em pequeno número para validar as jornadas de maior risco de ponta a ponta, enquanto a maior parte da cobertura permaneceria em camadas mais rápidas, estáveis e com melhor capacidade de diagnóstico em caso de falha.

## 3) Governança: defeitos, critérios de aceite e indicadores (~22 min)

Para gestão de defeitos, eu manteria o chat como canal de comunicação rápida, mas todo defeito confirmado precisaria virar registro no board. O fluxo poderia ser: identificado/triagem > confirmado > priorizado > em correção > pronto para reteste > reteste > fechado.

O registro deveria conter, no mínimo: título claro, ambiente, passo a passo para reprodução, comportamento esperado, comportamento atual, evidências, logs quando houver, severidade, prioridade, impacto e versão. A severidade mede o impacto e a prioridade define a ordem de tratamento. Produto, QA e desenvolvimento deveriam participar da triagem. Também seria necessário estabelecer um SLA inicial por severidade.

Para os critérios de aceite, cada história deveria possuir critérios claros. Eu adotaria BDD com linguagem natural e padrão Gherkin quando aplicável.

Uma história só entraria em desenvolvimento quando atendesse ao DOR, com pontos como:

- objetivo claro
- regra de negócio conhecida
- critérios de aceite definidos
- integrações identificadas
- risco avaliado
- dependências conhecidas
- dados necessários identificados

No DOD, a história só estaria pronta quando:

- código concluído
- code review realizado
- testes unitários aprovados
- critérios de aceite testados
- testes de API executados, quando aplicável
- testes automatizados atualizados, quando aplicável
- evidências registradas
- defeitos críticos resolvidos
- documentação criada ou atualizada
- observabilidade incluída

### Severidade, prioridade e acordo de tratamento

Severity 1 (Crítica): indisponibilidade ampla, corrupção/perda de dados, pagamento duplicado/indevido, falha que possa gerar impacto jurídico ou regulatório, ou bloqueio da jornada sem workaround. Tratamento imediato, war room, contenção em até 1 hora e correção/rollback prioritário.

Severity 2 (Alta): função crítica degradada ou bloqueada para parcela relevante dos clientes, mensagem de decisão incorreta, falha de integração crítica ou processamento que exige intervenção operacional significativa, com workaround limitado. Triagem no mesmo dia e plano de correção em até 24 horas; para release, exige correção ou aceite formal de risco.

Severity 3 (Média): defeito funcional com impacto moderado, restrito, com workaround viável e sem risco financeiro/regulatório imediato. Triagem em até 1 dia útil e planejamento de correção na sprint/release seguinte conforme prioridade.

Severity 4 (Baixa): problema cosmético, usabilidade menor, texto ou comportamento sem impacto relevante na jornada. Entra em backlog e é priorizado por produto.

A prioridade (P1 a P4) seria definida na triagem por QA, Produto e Desenvolvimento, considerando severidade, alcance, urgência de negócio, frequência e existência de workaround. Severidade não seria reduzida apenas para caber no cronograma.

Para os indicadores, eu usaria: cobertura de riscos críticos, taxa de aprovação da regressão crítica, defeitos abertos por severidade, lead time de defeitos críticos e flaky test rate.

### Indicadores de qualidade e decisão

1. Cobertura dos riscos críticos = riscos críticos com ao menos um teste aprovado / total de riscos críticos mapeados x 100. Fonte: matriz de risco + ferramenta de testes. Leitura: diária na janela de release e semanal fora dela. Público: QA Lead, Produto, Arquitetura e gerente do programa. Decisão: identificar risco crítico sem evidência suficiente e bloquear/condicionar a liberação.

2. Taxa de aprovação da regressão crítica = cenários críticos aprovados / cenários críticos executados x 100. Fonte: pipeline/gestão de testes. Leitura: por execução. Público: squads e release management. Decisão: permitir ou bloquear avanço para homologação/release.

3. Defeitos escapados para produção por severidade = quantidade de defeitos encontrados em produção que deveriam ter sido detectados antes da release, segmentados por Sev1-Sev4. Fonte: incidentes + board de defeitos. Leitura: por onda e mensal. Público: liderança do programa. Decisão: ajustar cobertura, processo e investimento nas áreas onde o escape está ocorrendo.

4. Lead time de defeitos críticos = tempo entre identificação e contenção/solução de Sev1/Sev2. Fonte: Jira/board + incident management. Leitura: por incidente e tendência mensal. Público: QA Lead, Engenharia e diretor em casos críticos. Decisão: avaliar capacidade de resposta e necessidade de reforço operacional/arquitetural.

5. Flaky test rate = testes com resultado inconsistente sem mudança funcional / total de testes automatizados executados x 100. Fonte: CI/CD e histórico de execuções. Leitura: semanal e por release. Público: QA/Engenharia. Decisão: retirar testes não confiáveis do gate, priorizar estabilização e medir recuperação da confiança.

6. Integridade da jornada E2E crítica = jornadas críticas aprovadas de ponta a ponta / jornadas críticas planejadas x 100. Fonte: suíte E2E + evidências de integração. Leitura: antes de cada onda. Público: QA Lead, Produto e patrocinador. Decisão: evidenciar se a cadeia entre sistemas está realmente validada.

Indicadores que eu não adotaria como meta principal neste momento: quantidade total de casos de teste, porque volume não demonstra cobertura de risco e percentual geral de testes aprovados, porque pode produzir 100% verde sobre uma seleção inadequada, exatamente como ocorreu na onda 2. Também evitaria usar quantidade de bugs encontrados por QA como meta individual, pois incentiva comportamento contrário à prevenção de defeitos.

## 4) A suíte automatizada em que ninguém confia (~18 min)

Como estratégia, eu faria uma triagem inicial para não descartar os 220 testes imediatamente. Classificaria os testes em: críticos e confiáveis, que seriam mantidos, críticos e flaky, que teriam correção prioritária, cenários de baixo valor e alta manutenção, que poderiam ser removidos, cenários duplicados, que seriam consolidados, e cenários que deveriam estar em camada inferior, que poderiam migrar para API ou componente.

Para decidir o que manter, eu avaliaria criticidade do fluxo, frequência de execução, histórico de defeitos encontrados, custo de manutenção, duração e dependências externas. Pressão de prazo não altera o resultado técnico, altera apenas quem assume explicitamente o risco para nova estratégia, eu seguiria uma pirâmide adaptada, concentrando maior volume em testes unitários, API, componentes e contrato, e menor volume em integração e E2E. Os testes E2E ficariam reservados às jornadas realmente críticas. A automação priorizaria regressões repetitivas, APIs críticas, regras estáveis e cenários negativos recorrentes. Funcionalidades recém-criadas, UX e comportamentos ainda instáveis permaneceriam inicialmente em testes manuais e exploratórios.

Eu adotaria uma combinação: recuperar o que tem valor e descartar/reimplementar o que não gera sinal confiável. Nas primeiras semanas, cada um dos 220 testes seria classificado por criticidade, estabilidade, camada correta, tempo de execução e dependência externa. Teste crítico e flaky entra na fila de estabilização; teste de baixo valor, duplicado ou excessivamente E2E é removido ou migrado para API/contrato.

Quality gates de automação: bloqueiam a entrega falhas reproduzíveis em testes unitários/componentes obrigatórios, contratos de integrações críticas, APIs críticas, smoke e E2E das jornadas de alto risco, além de qualquer falha relacionada a pagamento, idempotência, status ou comunicação de decisão. Testes flaky conhecidos, cenários de baixa criticidade e suítes exploratórias/longas apenas informam até serem estabilizados. Em caso de falha, a execução é considerada inconclusiva até haver diagnóstico. A exceção a um gate só pode ocorrer com evidência do defeito/instabilidade, impacto analisado, compensação definida, responsável nominal e aceite formal de risco.

## 5) Time, proatividade além do escopo e comunicação (~17 min)

Pensando nos três QAs hoje isolados em squads diferentes, eu criaria uma frente de QA do programa, mesmo que operacionalmente os profissionais continuem apoiando as squads.

QA 1 - foco em sinistros, core e regras de negócio

QA 2 - foco em integrações, APIs, antifraude e eventos

QA 3 - foco em canais, pagamentos, jornada E2E, automação e apoio à squad de dados

Como QA Lead, eu ficaria responsável pela estratégia, risco, governança, métricas, planejamento, integração entre as squads e comunicação executiva.

Implementaria uma abordagem de shift-left testing, incluindo QA desde discovery, refinamento, arquitetura, definição de critérios de aceite, planejamento e review de riscos. Para justificar ampliação do time, eu levaria ao diretor um argumento de risco: três profissionais precisam garantir quatro squads, onze integrações e uma jornada com impacto financeiro. A ausência de cobertura já se materializou em pagamento duplicado, sinistros bloqueados, aumento operacional de antifraude e comunicação incorreta. O custo adicional de capacidade de QA deve ser comparado com perda financeira, horas de incidente, retrabalho, atraso, risco jurídico e risco regulatório.

Eu priorizaria três melhorias:

1. Plano de rollback e estratégia de implantação: checklist, critérios de rollback, responsáveis, feature flags quando possível, plano de comunicação e smoke test pós-release
2. Rastreabilidade entre requisito, cenário, execução e defeito, utilizando uma ferramenta como Jira/Zephyr
3. Gestão de massa de teste: evitar dados reais de segurados em homologação, utilizar dados sintéticos, perfis controlados, regras de acesso e limpeza das massas, considerando LGPD

Na quarta squad, sem QA dedicado, eu usaria cobertura compartilhada por risco e defenderia capacidade adicional temporária. O argumento para o diretor seria financeiro e operacional: a lacuna já produziu um pagamento duplicado, 240 sinistros bloqueados, 62 comunicações conflitantes e aumento de 3x no antifraude. Reforçar QA custa menos do que repetir incidentes com impacto jurídico, financeiro, regulatório e reputacional.

## 6) Uso de IA no seu dia a dia (~10 min)

No dia a dia, eu utilizaria IA como acelerador de análises e não como autoridade sobre qualidade. Tenho experiência utilizando ferramentas como Copilot, Claude, Devin e outras soluções de IA. Neste programa, poderia utilizá-las em frentes como análise de requisitos, geração inicial de cenários, análise de defeitos, investigação de flaky tests, geração de massa sintética e apoio com a documentação.

Mesmo assim, eu não confiaria na resposta sem validação. A IA não deve interpretar sozinha regras de negócio, concluir causa raiz, definir se uma release está segura ou determinar a cobertura de testes. A responsabilidade e a decisão permanecem humanas. Também é importante não confundir volume de cenários com qualidade: a IA pode gerar muitos testes rapidamente, mas quantidade não substitui estratégia.

Os principais riscos são alucinação, interpretação incorreta e exposição de dados. Tudo o que pode afetar uma decisão precisa ser validado. Em relação à LGPD, não enviaria CPF, nome, endereço ou dados financeiros para ferramentas não aprovadas. Em segurança, também não compartilharia tokens, credenciais ou informações internas. A IA acelera o processo, mas o QA questiona, valida e assume a decisão.

Concluindo a análise, o incidente da onda 2 evidencia uma deficiência estrutural na forma como a qualidade está sendo conduzida no programa. Temos QA fragmentado, ausência de visão E2E, automação sem confiabilidade e pouca rastreabilidade. Nas seis semanas anteriores à onda 3, eu estruturaria uma operação de QA inicialmente focada na redução dos riscos críticos: compreender e conter os incidentes da onda 2, impedir duplicidade de pagamento, estabelecer a jornada crítica e testes E2E, formalizar a gestão de defeitos, implantar cobertura orientada a risco e preparar observabilidade e rollback para a entrada da onda 3.

O sucesso da frente de QA não deve ser medido apenas pela quantidade de testes executados, mas pela capacidade do programa de compreender seus riscos e impedir que defeitos críticos cheguem à produção.

Usos concretos neste programa: na análise de requisito, eu pediria à IA que identificasse regras ausentes, estados e exceções a partir de histórias já escritas; usaria a resposta como checklist para refinamento, e não como regra de negócio. Para casos de teste, pediria variações positivas, negativas, limites, retries, timeouts, duplicidade e concorrência, revisando tudo contra critérios de aceite e arquitetura. Para defeitos, poderia usar dados anonimizados para agrupar padrões e sugerir correlações, mas causa raiz só seria aceita com evidência técnica em logs, traces, banco e código. Para flaky tests, usaria IA para resumir histórico de falhas e comparar mensagens/stack traces, sem permitir que ela classifique automaticamente uma falha como “flaky” e libere a pipeline.

Na geração de massa, eu utilizaria apenas dados sintéticos e regras aprovadas, sem dados reais de segurados. Na documentação e comunicação executiva, a IA poderia melhorar estrutura e clareza, mas números, riscos, decisões e compromissos seriam revisados por mim.

A falsa sensação de cobertura aparece quando a IA gera dezenas de casos semelhantes e o time interpreta quantidade como proteção. A IA pode inventar regras inexistentes, reproduzir estados que não foram descritos. Por isso, IA seria acelerador de trabalho, não fonte de verdade nem aprovadora de release.

## Cláusula de Autoria e Uso de IA

Ao entregar este teste o candidato declara que a solução foi desenvolvida por ele, refletindo seus conhecimentos e experiência.

Não é permitido utilizar IA generativa (ChatGPT, Claude, Copilot, Gemini ou similares) para produzir respostas, códigos, arquitetura ou textos.

O código e as respostas serão discutidos na entrevista técnica.

Indícios de plágio, uso de IA ou incapacidade de defender tecnicamente a solução poderão resultar em desclassificação.
