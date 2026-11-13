# Case: Performance de Pagamentos — iFood Pago

## Contexto

Case para a posição de Analista de Negócios Jr no iFood Pago. O problema de negócio é investigar a queda de performance nas aprovações transacionais de algumas adquirentes em determinados negócios do iFood.

## Dados

Três bases fornecidas em um arquivo Excel, importadas para um Google Sheets compartilhado:

| Base | Registros | Granularidade | Descrição |
|---|---|---|---|
| `base_transacao` | 1.000 | 1 transação/linha | Dados transacionais (valor, status, código de erro, método, bandeira, data) |
| `base_adquirente` | 999 | 1 transaction_id/adquirente | Mapeia transação → adquirente (A, B, C, D) |
| `base_negocio_iFood` | 1.000 | 1 transaction_id/negócio | Mapeia transação → linha de negócio (Delivery, Market, Club) |

**Período:** 22 a 29/01/2024 (6 dias — sábado e domingo sem dados)

## Estrutura do projeto

```
payments-case/
├── notebooks/
│   └── case_analise.py          # Notebook Databricks (SQL + Python)
├── data/                         # Dados originais em CSV
│   ├── base_transacao.csv
│   ├── base_adquirente.csv
│   └── base_negocio_iFood.csv
├── presentation/                 # Apresentação final
└── README.md
```

## Estrutura do notebook

O notebook é organizado em fases, alternando células Python (ingestão e tratamento) e SQL (consultas analíticas). Cada fase tem uma markdown cell de introdução e uma de conclusões com os achados.

| Fase | Pergunta | Células | O que faz |
|---|---|---|---|
| 0 | Setup | 2 Python | Importa 3 bases do Google Sheets via CSV, trata nulos, cria temp views |
| 1 | Q1 | 2 SQL + 1 markdown | Cria view unificada com LEFT JOIN + colunas analíticas |
| 2 | Q2 | 4 SQL + 1 markdown | TPV diário, rotação de adquirentes, decomposição volume vs ticket |
| 3 | Q3 | 3 SQL + 1 markdown | Conversão geral, trajetória, segmentada |
| 4 | Q4 | 5 SQL + 1 markdown | TPV recusado por adquirente, recuperável vs não recuperável |
| 5 | Q5 | 6 SQL + 1 markdown | Matriz multidimensional, perfil de erro, normalização, D, aquecimento, CLUB |
| 6 | Q6 | 4 SQL + 1 markdown | Concentração do B, detalhe do D, recusas do A, impacto de recuperação |

## Como executar

### 1. Google Sheets

As 3 URLs de exportação CSV já estão configuradas no notebook (seção 0.2). O Sheets está compartilhado como "qualquer pessoa com o link pode ver" — não precisa publicar.

### 2. Databricks

Importar `notebooks/case_analise.py` como notebook e executar as células em ordem (Run All).

As células de ingestão (Fase 0) são Python. As consultas analíticas (Q1-Q6) são SQL puro — o Databricks renderiza as tabelas nativamente.

## Premissas

1. **LEFT JOIN** para preservar todas as transações (69 sem adquirente mapeado)
2. `adquirente` nulo → "Não identificado" (via COALESCE)
3. `card_issuer_code` string 'null' → nulo (ocorre em PIX)
4. `amount` tratado para float (vírgula decimal do Sheets → ponto)
5. Coluna `operação` usa backticks no SQL por conter acento
6. Sábado e domingo sem dados — limitação explícita da base
7. Códigos de erro seguem padrão ABECS/ISO 8583

## Códigos de retorno (ABECS)

| Código | Significado | Reversível | Responsável |
|---|---|---|---|
| 51 | Saldo/limite insuficiente | Sim | Cliente/Emissor |
| 5 | Do not honor (recusa genérica) | Ambíguo | Emissor |
| 91 | Emissor fora do ar | Sim | Infraestrutura |

**Sources:**
- [Códigos ABECS — Braspag](https://suporte.braspag.com.br/hc/pt-br/articles/360047632291)
- [ISO 8583 Response Codes — DevindTools](https://devindtools.com/iso8583/response-codes)

## Principais achados

### Q2 — TPV ao longo do período
- TPV total: R$ 45.725,02 | Aprovado: R$ 39.186,76 | Recusado: R$ 6.538,26 (14,3%)
- Volume triplica do dia 24→25 (88→262), 92% por efeito volume (entrada Adquirente B)
- Adquirentes operam em rotação (sequência de dias), não em paralelo
- % TPV recusado sobe de 9,3% (dia 24) para 17,6% (dia 25)

### Q3 — Conversão
- Conversão geral: 85,7% (857/1.000)
- Queda de 93,2% (dia 23) para 82,4% (dia 25) coincide com entrada do Adquirente B

### Q4 — TPV recusado
- TPV recusado: R$ 6.538,26
- Recuperável (cód 51+91): R$ 4.863,05 (74,4%) → retry/alternativa
- Não recuperável (cód 5): R$ 1.675,21 (25,6%) → investigar emissor
- Se recuperado: conversão 85,7% → ~95,7%
- Adquirente D recusa transações de valor alto (ticket recusado R$ 70 vs R$ 42 aprovado)

### Q5 — Performance por adquirente

| Adquirente | Conv | TPV Recusado | %Rec | Recup% | Cod dominante | Ticket recusado |
|---|---|---|---|---|---|---|
| A | 87,2% | R$ 2.272 | 11,3% | 100% | 51 (saldo) | R$ 44,56 |
| B | 82,9% | R$ 2.106 | 15,6% | 29,7% | 5 (do not honor) | R$ 43,87 |
| C | 86,5% | R$ 1.136 | 14,2% | 84,6% | 91 (emissor fora) | R$ 49,38 |
| D | 82,7% | R$ 982 | 25,9% | 100% | 91 (emissor fora) | R$ 70,12 |

**Achados adicionais:**
- Cada adquirente tem perfil de erro diferente (saldo, do not honor, infraestrutura)
- Normalização A vs B: B é genuinamente pior em CREDIT e PIX (não é efeito de mix)
- Adquirente D recusa valor alto: 94,1% conv em R$ 0-20 vs 66,7% em R$ 100-200
- Padrão de aquecimento: todas melhoram do 1º para o 2º dia (cold start)
- IFOOD_BR_CLUB não tem adquirente mapeado (governança)

### Q6 — Priorização

| Prio | Ação | Impacto | Esforço | Quem |
|---|---|---|---|---|
| P1 | Retry automático cod 91 (C e D) | R$ 1.955 | Baixo | Interno (tech) |
| P2 | Oferta método alternativo cod 51 (A) | R$ 2.272 | Médio | Interno (produto) |
| P3 | Roteamento alternativo cod 5 (B) | ~R$ 1.480 | Médio | Interno (tech) |
| P4 | Diagnóstico estruturado com D | R$ 982 | Médio | Externo (parceiro) |
| P5 | Investigação emissores 1 e 341 (via B) | ~R$ 560 | Alto | Externo (emissor) |
| P6 | Mapear adquirente IFOOD_BR_CLUB | R$ 389 | Baixo | Interno (dados) |

**Justificativa da ordem:** P1 e P2 primeiro (maior TPV recuperável + controle interno). P3 interno mas com incerteza. P4 e P5 precisam de parceiro externo. P6 baixo impacto mas resolve governança.

## Scorecard de acompanhamento

**Indicadores diários:** conversão por adquirente, TPV recusado por adquirente/código, % recuperável
**Indicadores semanais:** distribuição de códigos, ticket recusa vs aprovação
**Alertas:** conv -5pp/dia, TPV recusado >15%, código de erro novo

## Limitações

- 6 dias de dados (sem sábado e domingo)
- 69 transações sem adquirente mapeado
- Adquirente D com apenas 1 dia de operação
- Sem horário de transação (apenas data)
- Amostra de 1.000 transações — não representa o volume total do iFood Pago
- Padrão de aquecimento baseado em apenas 2 dias por adquirente (exceto A)
