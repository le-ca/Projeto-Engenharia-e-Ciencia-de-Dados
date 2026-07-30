# Sistema de Monitorização de Preços e Inteligência Competitiva
### Plano de Execução Detalhado — do início ao projeto completo

> **Antes de começares — lê isto.**
> O objetivo não é "perfeito", é **completo e no ar**. Num projeto de portefólio, o perfeccionismo é uma armadilha: há sempre mais um teste, mais um *refactor*, mais uma *feature*, e muita gente fica presa a polir para sempre um projeto que nunca chega a estar à mostra. A regra de ouro: chega primeiro a um sistema que corre de ponta a ponta com um link clicável, **e só depois poles**. Um projeto a 85% que está no ar vale mais, no CV, do que um a 100% que nunca sai do teu computador.
>
> **Como usar este documento:** trabalha fase a fase, de cima para baixo. Cada passo tem uma ação concreta; cada fase tem um critério de *"Concluído quando"* — só avanças quando ele estiver cumprido. Vai escrevendo notas das decisões e problemas à medida que acontecem: vais precisar delas no relatório (Fase 10).

---

## Estrutura de pastas sugerida

Cria isto logo no início (Fase 0), para teres onde encaixar tudo o resto:

```
price-intel/
├── src/
│   ├── scrapers/          # um módulo por site alvo
│   │   ├── base.py        # classe base comum (headers, delays, retry)
│   │   └── loja_x.py
│   ├── transform/         # normalização, validação, matching
│   ├── analysis/          # deteção de anomalias, posicionamento, sazonalidade
│   ├── storage/           # modelos SQLAlchemy, repository pattern
│   ├── api/               # FastAPI
│   ├── dashboard/         # Streamlit
│   └── alerts/            # email / webhook
├── tests/
│   ├── fixtures/          # HTML de exemplo guardado localmente
│   └── ...
├── migrations/            # Alembic
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
├── .env.example           # exemplo SEM segredos reais
├── .gitignore             # tem de incluir .env
└── README.md
```

---

## FASE 0 — Planeamento e Preparação
**Objetivo:** definir o âmbito exato e preparar o terreno legal e técnico. *(≈ 2-3 dias)*

1. **Fecha o nicho, e estreito.** Escreve numa frase o que vais monitorizar — por exemplo: *"preços de [categoria específica] de 1 loja de referência vs 3 concorrentes diretos no mercado português"*. Estreito é melhor: mais exequível, conta melhor a história numa entrevista, e é a forma de um serviço vendável.
2. **Escolhe 3 a 5 lojas/concorrentes alvo.** Prefere sites com informação pública sem *login*. Anota o URL de cada um.
3. **Escolhe 10-20 produtos piloto.** Não tentes cobrir tudo de início. Escolhe produtos que existam em várias das lojas (para poderes testar o *matching*).
4. **Verifica robots.txt e os termos de serviço de cada site.** Vai a `https://site.com/robots.txt` e confirma que as páginas que queres não estão proibidas. Lê os termos de serviço à procura de cláusulas sobre *scraping*/uso automatizado. **Guarda um print ou nota do que encontraste** — vais precisar na secção ética/legal do relatório. Se um site proíbe explicitamente, troca-o por outro.
5. **Cria o repositório Git** (GitHub/GitLab) com a estrutura de pastas acima e um `.gitignore` que **inclui o `.env`** desde o primeiro *commit*.
6. **Escreve um documento curto de requisitos** (`docs/requisitos.md`): o que o sistema deve fazer, que métricas produz (diferença de preço %, histórico, tendência), que alertas gera e quando.

> **Concluído quando:** tens o repositório criado, a estrutura de pastas, o nicho e os produtos definidos por escrito, e a verificação legal documentada.

---

## FASE 1 — Ambiente e Infraestrutura Base
**Objetivo:** um ambiente reproduzível onde tudo corre igual na tua máquina e em produção. *(≈ 2-4 dias)*

7. **Configura o ambiente Python com Poetry** (recomendado por fixar versões): `poetry init`, depois `poetry add requests beautifulsoup4 playwright sqlalchemy alembic pydantic pandas fastapi uvicorn streamlit`. Alternativa mais simples: `venv` + `requirements.txt`.
8. **Instala e configura PostgreSQL localmente.** Se quiseres otimizar para séries temporais de preços, considera **TimescaleDB** (extensão do PostgreSQL). Cria uma base de dados vazia para o projeto.
9. **Configura Docker + Docker Compose** com três serviços: `app`, `db` (PostgreSQL/Timescale) e `redis`. Isto garante reproduzibilidade e impressiona quem avaliar o repositório. Testa com `docker compose up` até tudo arrancar.
10. **Cria o `.env`** (para credenciais e configurações) e um `.env.example` sem segredos reais. Confirma que o `.env` **nunca** vai para o Git.
11. **Configura *linting* e formatação desde já:** adiciona `ruff` e `black`, cria a configuração no `pyproject.toml` e corre uma vez para veres que passa.

> **Concluído quando:** `docker compose up` arranca app + base de dados + Redis sem erros, e o *linting* corre limpo.

---

## FASE 2 — Extração de Dados (Extract) — *a fase mais demorada e frágil*
**Objetivo:** *scrapers* que recolhem preços de forma fiável e "educada". *(≈ 1-2 semanas)*

> ⚠️ **Reserva folga aqui.** Os *scrapers* partem quando os sites mudam de estrutura — é a parte que mais tempo consome e que mais manutenção exige. Constrói-os para falharem de forma controlada, não silenciosa.

12. **Escolhe a ferramenta por site:** `requests` + `BeautifulSoup` para sites estáticos simples; **Playwright** para sites que carregam preços via JavaScript. Inspeciona cada site no navegador (F12) para decidir.
13. **Cria uma classe base de scraper** (`scrapers/base.py`) com o comportamento comum: *headers*, *delays*, *retry*. Cada site herda dela e implementa só a extração específica.
14. **Implementa o primeiro scraper** para o site mais simples, com testes manuais. Extrai: nome do produto, preço, moeda, URL, e um identificador (SKU/EAN se existir).
15. **Adiciona boas práticas de "cortesia":** rotação de *user-agents*, *delays* aleatórios entre pedidos (ex.: 2-5 segundos), e respeitar o `robots.txt`. O objetivo é nunca sobrecarregar o servidor alvo.
16. **Adiciona tratamento de erros e *retry* com *backoff* exponencial** (biblioteca `tenacity`). Se um pedido falha, tenta de novo com espera crescente; ao fim de N tentativas, regista o erro e segue.
17. **Repete para os restantes sites alvo,** reaproveitando a classe base.
18. **Escreve testes unitários para cada scraper** usando **HTML de exemplo guardado localmente** em `tests/fixtures/` — nunca fazendo pedidos reais nos testes. Assim os testes são rápidos e não dependem da internet.
19. **Configura um *scheduler*.** Para o MVP, um `cron` simples que corre os *scrapers* de X em X horas chega. Deixa a nota de que em produção isto pode passar a Celery + Redis Beat.

> **Concluído quando:** cada scraper recolhe os produtos piloto de forma fiável, falha graciosamente, e tem testes que passam com *fixtures* locais.

---

## FASE 3 — Transformação de Dados (Transform)
**Objetivo:** transformar dados brutos e sujos em registos limpos e comparáveis entre lojas. *(≈ 1 semana)*

20. **Define o esquema de dados normalizado:** `nome_produto`, `preco`, `moeda`, `loja`, `url`, `sku_ean`, `timestamp`. Documenta-o.
21. **Implementa validação com Pydantic:** rejeita preços negativos, nulos, zero, ou absurdos (ex.: um telemóvel a 2 €). Regista o que for rejeitado em vez de o descartar em silêncio.
22. **Implementa o *matching* de produtos entre lojas** — a parte que costuma consumir mais tempo do que se espera:
    - **1ª escolha:** casar por SKU/EAN quando disponível (fiável).
    - **Recurso (*fallback*):** *fuzzy matching* dos nomes com a biblioteca `rapidfuzz`, depois de normalizar as *strings* (minúsculas, remover acentos, tirar palavras de ruído tipo "novo", "portes grátis"). Define um limiar de semelhança acima do qual consideras que é o mesmo produto.
23. **Implementa o cálculo de métricas:** diferença de preço % entre lojas, histórico por produto, e tendência (subiu/desceu/estável).
24. **Escreve testes unitários** para a validação e para o *matching* — incluindo casos difíceis (nomes parecidos que **não** são o mesmo produto).

> **Concluído quando:** os dados brutos entram e saem limpos, validados e corretamente associados entre lojas, com testes a cobrir os casos-limite.

---

## FASE 4 — Armazenamento (Load)
**Objetivo:** guardar o histórico de forma consultável e eficiente. *(≈ 3-5 dias)*

25. **Desenha o schema da base de dados.** Tabelas mínimas:
    - `lojas` (id, nome, url)
    - `produtos` (id, nome_canónico, sku_ean, categoria)
    - `precos_historicos` (id, produto_id, loja_id, preco, moeda, timestamp)
    - `alertas` (id, produto_id, tipo, threshold, timestamp)
26. **Cria as migrações com Alembic** (`alembic init`, depois `alembic revision --autogenerate`). Assim a base de dados é versionada como o código.
27. **Implementa a camada de acesso a dados com *repository pattern*** — funções tipo `guardar_preco()`, `obter_historico(produto_id)` — para separares a lógica de negócio do SQL.
28. **Otimiza os índices** para as consultas frequentes: índice em `(produto_id, timestamp)` e em `loja_id`. Se usares TimescaleDB, transforma `precos_historicos` numa *hypertable*.
29. **Testa inserções em volume:** gera alguns milhares de registos falsos e confirma que as consultas continuam rápidas.

> **Concluído quando:** os preços recolhidos são gravados e consultáveis por produto e por data, com migrações a funcionar e consultas rápidas.

---

## FASE 5 — Análise de Dados *(a tua componente de Ciência de Dados — não está no plano original)*
**Objetivo:** extrair *insight* dos dados, não só mostrá-los. Esta é a fase que te distingue de um projeto puramente de engenharia. *(≈ 1-1,5 semanas)*

> **Nota importante sobre a ordem:** a *previsão* de preços precisa de histórico que não vais ter nas primeiras semanas — por isso começar por aí dá um modelo fraco. Faz primeiro o que dá valor com pouco histórico (anomalias, posicionamento) e deixa a previsão para o fim.

30. **Deteção de anomalias / quedas de preço.** Com `pandas`, sinaliza descidas fora do normal usando métodos estatísticos simples: *z-score* sobre a média móvel, ou intervalo interquartil (IQR). É isto que alimenta os alertas úteis.
31. **Análise de posicionamento competitivo.** Para cada produto, calcula onde cada concorrente se situa face à mediana do mercado — quem é consistentemente *premium* e quem é *low-cost*. Boa fonte de gráficos para o relatório.
32. **Padrões de sazonalidade / *timing*.** Analisa quando as lojas mudam preços (dia da semana, promoções recorrentes). Mesmo uma análise descritiva simples é valiosa.
33. **(No fim, com histórico suficiente) Previsão de preços.** Aí sim, experimenta um modelo de série temporal — `Prophet` ou `statsmodels`. Apresenta-o honestamente como extensão, com as suas limitações.
34. **Valida a análise:** confirma que os *insights* fazem sentido contra o que sabes do nicho (aqui é que a escolha de uma categoria que conheces compensa).

> **Concluído quando:** tens pelo menos deteção de anomalias e análise de posicionamento a funcionar sobre dados reais, com resultados que consegues explicar.

---

## FASE 6 — Apresentação e Alertas
**Objetivo:** tornar o sistema utilizável e visível. *(≈ 1-1,5 semanas)*

35. **Constrói o dashboard em Streamlit** (rápido para MVP). Mostra a lista de produtos monitorizados, o preço atual por loja, e permite escolher um produto.
36. **Implementa gráficos de evolução de preços** por produto e por concorrente (com `plotly` ou `altair`), e um gráfico de posicionamento vindo da Fase 5.
37. **Implementa o sistema de alertas:** dispara uma notificação quando um *threshold* é ultrapassado (ex.: concorrente baixou preço >5%). Canal à escolha — email via SMTP (`smtplib`) ou *webhook* de Telegram/Slack (mais simples de montar).
38. **Cria uma API REST com FastAPI** com *endpoints* documentados (Swagger automático em `/docs`): por exemplo `GET /produtos`, `GET /produtos/{id}/historico`, `GET /alertas`. Usa modelos Pydantic para as respostas.

> **Concluído quando:** consegues abrir o dashboard, ver a evolução de preços de um produto, receber um alerta de teste, e consultar a API em `/docs`.

---

## FASE 7 — Qualidade e Robustez *(versão leve — não gold-platear)*
**Objetivo:** mostrar que sabes fazer isto bem, sem te perderes a polir. *(≈ 3-5 dias)*

39. **Configura CI com GitHub Actions:** um *workflow* que corre `pytest` e `ruff` a cada *push*. Um *badge* verde no README vale muito.
40. **Adiciona *logging* estruturado** (`structlog`) nos componentes principais — sobretudo nos *scrapers*, para saberes quando e porque falharam.
41. **Integra o Sentry** (plano gratuito) para captura de erros em produção.
42. **Revê a cobertura de testes** no código crítico (transformação, *matching*, acesso a dados). Não precisas de >70% em todo o lado — foca-te no que importa.

> **Concluído quando:** os testes correm automaticamente no GitHub a cada *push* e os componentes-chave têm *logging* e captura de erros.

---

## FASE 8 — Deployment
**Objetivo:** o link clicável. Esta fase existe para pôr o projeto no ar. *(≈ 3-5 dias)*

> ⚠️ O primeiro *deploy* traz sempre surpresas (variáveis em falta, versões diferentes, portas). Conta com isso e não deixes para a última hora.

43. **Escolhe a plataforma:** **Railway** ou **Render** são os mais rápidos para começar (têm PostgreSQL gerido); um **VPS com Docker** dá mais controlo mas mais trabalho.
44. **Configura as variáveis de ambiente de produção** na plataforma (as mesmas chaves do `.env`, com valores reais).
45. **Faz o *deploy*** do pipeline, da base de dados e do dashboard. Confirma que o dashboard abre num URL público.
46. **Configura *backups* automáticos** da base de dados (as plataformas geridas costumam ter esta opção).

> **Concluído quando:** existe um URL público onde o dashboard abre com dados reais.

---

## FASE 9 — Correr, Validar e Iterar
**Objetivo:** provar que o sistema aguenta o tempo. *(≈ 1-2 semanas de calendário — sobretudo tempo a correr, não de esforço)*

47. **Deixa o sistema a correr 1-2 semanas** a recolher dados sozinho, enquanto fazes outras coisas.
48. **Recolhe métricas de desempenho:** *uptime* dos *scrapers*, taxa de erro, latência das consultas. Guarda-as — são resultados para o relatório.
49. **Corrige os bugs** que aparecerem no período de teste (vão aparecer, sobretudo nos *scrapers*).
50. **Documenta limitações conhecidas** e melhorias futuras à medida que as identificas.

> **Concluído quando:** tens 1-2 semanas de dados reais recolhidos e um registo das métricas de desempenho.

---

## FASE 10 — Relatório Final
**Objetivo:** o entregável que te faz parecer engenheiro, não alguém que seguiu um tutorial. *(≈ 3-5 dias de escrita, mas distribuídos ao longo do projeto)*

> **O truque:** escreve ao longo do caminho, não só no fim. Guarda notas das decisões e problemas quando eles acontecem — reconstruí-los no fim é muito mais difícil e menos honesto.

Estrutura sugerida:

1. **Introdução** — contexto do problema, motivação, objetivos.
2. **Revisão de mercado** — soluções existentes (Prisync, Price2Spy, Competera…) e as lacunas que o teu projeto endereça.
3. **Arquitetura do sistema** — diagrama geral e justificação das escolhas tecnológicas.
4. **Metodologia de implementação** — descrição de cada fase (Extract / Transform / Load / Análise / Apresentação).
5. **Desafios técnicos e soluções** — *matching* de produtos, resiliência dos *scrapers*, dados sujos.
6. **Testes e validação** — estratégia de testes, resultados, cobertura.
7. **Resultados** — métricas obtidas: nº de produtos monitorizados, tempo de deteção de alterações, precisão do *matching*, *uptime*.
8. **Avaliação crítica** — limitações, riscos (ex.: mudanças na estrutura dos sites-alvo) e considerações éticas/legais (aqui entra o que documentaste na Fase 0).
9. **Conclusão e trabalho futuro** — extensões possíveis (ex.: previsão de preços com ML mais robusta).
10. **Anexos** — código relevante, diagrama da base de dados, capturas do dashboard.

---

## Estimativas de tempo (total)

O tempo depende muito de duas coisas — o teu nível atual e as horas por semana que dedicas:

| Cenário | Estimativa |
|---|---|
| **A tempo inteiro**, Python intermédio, ferramentas já familiares | **~5 a 7 semanas** |
| **Part-time (~10-15h/semana)**, nível intermédio | **~2,5 a 4 meses** de calendário |
| **A aprender várias ferramentas pelo caminho** | **~4 a 6 meses** — e neste caso corta o âmbito (menos sites, salta a API, dashboard mais simples) para garantires que chegas ao fim |

---

## Onde o tempo costuma disparar (atenção redobrada)

- **Fase 2 (scrapers):** a mais frágil e demorada. Sites mudam, e cada mudança parte um scraper. Reserva folga.
- **Fase 3 (matching de produtos):** quase sempre consome mais do que o previsto, sobretudo o *fuzzy matching*.
- **Fase 8 (primeiro deploy):** as surpresas de produção acumulam-se aqui. Não deixes para o fim absoluto.

---

## Riscos e notas finais

- **Manutenção contínua:** se algum dia isto virar um serviço pago, os *scrapers* exigem manutenção constante — precifica esse custo.
- **Legal/ético:** preferir dados públicos e APIs oficiais quando existam, ser conservador nos *delays*, e manter documentada a verificação da Fase 0.
- **Âmbito antes de perfeição:** se o tempo apertar, corta funcionalidades (menos sites, sem API, dashboard mais simples) em vez de deixar o projeto incompleto. **Completo e no ar > perfeito e na gaveta.**
