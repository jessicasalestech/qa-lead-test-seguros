<!-- seletor de idioma -->
[🇺🇸 English](README.md)

# QA Lead Sênior | Seguros — Teste Técnico

**Documento de decisão** para o programa *Nova Jornada de Sinistros* na **NFoque**.

Análise sênior de como estruturar uma frente de Quality Assurance para um programa crítico e
de grande porte sob pressão de cronograma — investigação, risco, estratégia, governança,
indicadores, automação e comunicação de liderança. Não envolve código; é um documento de decisão.

## Entregáveis

| Arquivo | Formato |
|---|---|
| [`Jessica_Sales_testetecnico.md`](Jessica_Sales_testetecnico.md) | Markdown |
| [`Jessica_Sales_testetecnico.docx`](Jessica_Sales_testetecnico.docx) | Word |

## O que contém

1. **Investigar antes de responder ao patrocinador** — hipóteses para os quatro sintomas
   simultâneos (240 sinistros parados, 62 mensagens contraditórias, antifraude triplicado,
   pagamento duplicado), por que uma homologação 100% verde convive com falhas em produção,
   um plano de investigação baseado em evidências e a decisão em portão para a onda 3.
2. **Estratégia de qualidade e plano de testes** — níveis de teste (incluindo os testes de
   jornada entre sistemas e de contrato, que não existem hoje), critérios objetivos de
   entrada/saída, as duas integrações sem ambiente de homologação e o critério de cobertura por risco.
3. **Governança** — fluxo de defeitos, critérios de pronto no lugar de "funcionou na demo" e
   seis indicadores (cada um ligado a uma decisão concreta).
4. **Automação em que ninguém confia** — triagem dos 220 cenários E2E instáveis, nova
   estratégia de automação e quais resultados barram release versus apenas informam.
5. **Time e comunicação** — reorganização de 3 analistas de QA em 4 squads, melhorias
   proativas além do escopo e a comunicação ao patrocinador.
6. **IA em QA** — onde usar, onde validar, onde cria falsa sensação de cobertura e os riscos
   de LGPD/segurança.

## Sobre

**Jessica Sales Melo** — QA Engineer Sênior (7+ anos) em bancário, serviços financeiros,
e-commerce e seguros; ISTQB CTFL; automação em Playwright, Cypress, Selenium, Appium, Robot
Framework, Rest Assured; arquiteturas orientadas a eventos na AWS, CI/CD e BDD.

*Entrega de teste técnico de processo seletivo — documento de decisão, não código de aplicação.*