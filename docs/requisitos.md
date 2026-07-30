# Requisitos — Sistema de Monitorização de Preços

## O que o sistema faz
Monitorização de preços de placas gráficas dedicadas, comparando uma loja de
referência com concorrentes portugueses, sobre um conjunto piloto de modelos.

## Âmbito do piloto
12 modelos. Onze presentes nas 4 lojas analisadas (RTX 5060, 5060 Ti 8GB,
5060 Ti 16GB, 5070, 5070 Ti, 5080, 5090; RX 9060 XT 8GB, 9060 XT 16GB, 9070,
9070 XT) + Intel Arc B580 (cobertura parcial em 2 lojas, mantido pela
diversidade de marca). RX 9070 GRE e geração anterior excluídos por cobertura
insuficiente.

Matching ao nível chipset (variante mais barata por loja), com correspondência
exata por EAN como técnica secundária.

## Lojas (MVP, uso não comercial)
- Globaldata — loja de referência
- PCDIGA
- Chip7

PcComponentes excluída do MVP — ver docs/analise_legal.md.

## Métricas que produz
- Diferença de preço (%) entre lojas
- Histórico de preços por produto
- Tendência (subiu / desceu / estável)

## Alertas que gera
- Quando um concorrente baixa o preço mais de X% (limiar a definir)
