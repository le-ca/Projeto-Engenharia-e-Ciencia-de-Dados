# Análise Legal e Ética — Verificação Pré-Scraping

Data da verificação: 2026-07-30
Âmbito: recolha de preços públicos de placas gráficas (dados factuais, sem
dados pessoais) para projeto académico / de portefólio.

## Princípios adotados
- Apenas páginas públicas, sem login, sem contornar proteções anti-bot.
- Descoberta de produtos via sitemap (não via páginas de pesquisa/filtro).
- Preços extraídos do HTML da página de produto (não de APIs internas).
- Delays conservadores (2-5s); mais lento nas lojas com anti-bot.
- User-agent identificável e honesto.

## Quadro-resumo

| Loja          | robots.txt (produto/categoria) | Termos: uso automatizado        | Portefólio | Comercial |
|---------------|-------------------------------|---------------------------------|------------|-----------|
| Chip7         | Permitido (Allow: /)          | Sem cláusula                    | ✅         | ✅        |
| Globaldata    | Permitido                     | Só RGPD (dados pessoais)        | ✅         | ✅        |
| PCDIGA        | Permitido                     | Proíbe spider/scrape comercial  | ✅ (pessoal)| ❌ s/ autorização |
| PcComponentes | Permitido (APIs bloqueadas)   | Proíbe reprodução + treino ML   | ⚠️         | ❌        |

## Por loja

### Chip7 (chip7.pt) — VERDE
- robots.txt: o mais permissivo, termina com Allow: /. Sem restrições de
  parâmetros nem crawl-delay. → candidata ao 1.º scraper.
- Termos: lidos na íntegra, sem cláusula de scraping/IP/IA. Só proíbe uso
  ilícito/fraudulento.
- Nota: alimenta comparador Kelkoo → preços distribuídos publicamente.
- DECISÃO: incluída, sem reservas.

### Globaldata (globaldata.pt, op. Caseking Iberia) — VERDE + REFERÊNCIA
- robots.txt: permite produto/categoria. Bloqueia pesquisa facetada
  (prefn/prefv, srule, pmin/pmax) e /search. Plataforma Salesforce (Demandware).
- Termos: menções a "base de dados"/"automático" são todas RGPD (tratamento
  dos MEUS dados pessoais), não restringem scraping. Sem cláusula de IP a
  proibir reprodução de conteúdos.
- Notas: cobertura 100% dos 12 modelos; stock em tempo real; reCAPTCHA só no login.
- DECISÃO: incluída como LOJA DE REFERÊNCIA (a mais limpa e completa).

### PCDIGA (pcdiga.com) — CONDICIONAL
- robots.txt: permite produto/categoria. Bloqueia parâmetros de
  ordenação/filtro/pesquisa e /recondicionados. Sitemap de produtos disponível.
- Anti-bot ativo (bloqueou acesso automático) → provável necessidade de
  Playwright + ritmo lento.
- Termos: secção 5 proíbe "monitorizar (spider, scrape)... guardar ou
  reproduzir o conteúdo... para qualquer atividade comercial, sem autorização
  escrita"; MAS permite copiar/armazenar para uso EXCLUSIVAMENTE pessoal.
  Secção 3(f) restringe meios de obtenção diferentes dos habituais (ambíguo).
- DECISÃO: incluída no MVP ao abrigo do uso pessoal/académico. Uma versão
  comercial exigiria autorização escrita da PCDIGA — ou substituí-la.

### PcComponentes (pccomponentes.pt) — VERMELHO
- robots.txt: permite produto/categoria, mas bloqueia /api/articles/search e
  /api/articles/*/buybox (não usar API de preços — extrair do HTML). Regra
  Disallow para Python-urllib → pedido de contenção. Anti-bot forte.
- Termos (aviso legal): proíbe reproduzir/copiar/explorar conteúdos sem
  autorização E proíbe usar conteúdos para treinar IA/ML (comercial ou não).
- DECISÃO: EXCLUÍDA do MVP. Reavaliar apenas com autorização; nunca treinar
  o modelo de previsão com dados desta loja.

## Conclusão
- Conjunto do MVP (não comercial): Globaldata (referência) + PCDIGA + Chip7.
- Base "limpa" para eventual versão comercial: Globaldata + Chip7.
- Alternativas a scraping a explorar antes de comercializar: feeds oficiais,
  programas de afiliado, ou autorização escrita (PCDIGA, PcComponentes).
- Regras técnicas transversais: descoberta via sitemap; preços do HTML;
  nunca contornar anti-bot; delays 2-5s; UA honesto.
- Nota: análise feita pelo próprio autor; não constitui aconselhamento jurídico.
