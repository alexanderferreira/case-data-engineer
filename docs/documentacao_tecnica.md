# Documentação Técnica — Case Data Engineer

## Visão Geral

O objetivo desta solução foi transformar nove fontes brutas, com formatos e padrões completamente diferentes entre si, em uma base analítica confiável e pronta para uso pelo time de BI. Para isso, optei por uma arquitetura em camadas (Medallion Architecture), que separa claramente a ingestão, o tratamento e a entrega dos dados.

A escolha por essa abordagem não foi por modismo — foi porque ela torna o processo rastreável. Se um número no dashboard parecer estranho, é possível voltar camada por camada e encontrar exatamente onde aquele dado foi gerado ou transformado.

---

## Arquitetura

```
sources/  (arquivos brutos)
    │
    ▼
┌──────────────────────────────────────────────────┐
│  BRONZE  — Ingestão fiel, sem transformações     │
│  bronze.erp_pedidos_cabecalho  (403 registros)   │
│  bronze.erp_pedidos_itens      (995 registros)   │
│  bronze.crm_clientes           (183 registros)   │
│  bronze.comercial_canais       (  8 registros)   │
│  bronze.cadastro_produtos      ( 72 registros)   │
│  bronze.logistica_entregas     (322 registros)   │
│  bronze.atendimento_ocorrencias(270 registros)   │
│  bronze.vendedores             ( 42 registros)   │
│  bronze.legado_regioes         (  8 registros)   │
└──────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────┐
│  SILVER  — Tratamento, normalização, qualidade   │
│  silver.dim_regioes        silver.dim_canais     │
│  silver.dim_produtos       silver.dim_clientes   │
│  silver.dim_vendedores                           │
│  silver.fct_pedidos_cabecalho                    │
│  silver.fct_pedidos_itens                        │
│  silver.fct_entregas       silver.fct_ocorrencias│
└──────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────┐
│  GOLD  — Modelo analítico para BI                │
│  gold.dim_clientes         gold.dim_produtos     │
│  gold.dim_vendedores       gold.dim_canais       │
│  gold.dim_regioes                                │
│  gold.fct_pedidos          gold.fct_itens_pedido │
│  gold.fct_entregas         gold.fct_ocorrencias  │
│  gold.mart_desempenho_mensal                     │
└──────────────────────────────────────────────────┘
```

---

## Exploração das Fontes

Antes de escrever qualquer transformação, dediquei tempo para entender cada arquivo — estrutura, conteúdo, qualidade e como cada um se relaciona com os demais. Esse passo evitou surpresas no meio do desenvolvimento.

### 1. erp_pedidos_cabecalho_2025.csv
- **Separador:** ponto-e-vírgula (`;`)
- **Registros:** 403
- **Colunas:** order_id, customer_code, seller_id, order_date, promised_date, status_order, gross_amount, discount_amount, net_amount, payment_details, last_update
- **Relacionamentos:** customer_code → crm_clientes, seller_id → vendedores

### 2. erp_pedidos_itens_2025.csv
- **Separador:** vírgula
- **Registros:** 995
- **Colunas:** order_id, item_seq, product_code, quantity, unit_price, total_item, item_status
- **Relacionamentos:** order_id → pedidos_cabecalho, product_code → cadastro_produtos

### 3. crm_clientes_export.xlsx
- **Registros:** 183
- **Colunas:** customer_id, nome_cliente, segmento, porte, cidade, estado, data_cadastro, email, status_cliente, updated_at

### 4. comercial_canais.xlsx
- **Registros:** 8 (incluindo duplicatas)
- **Colunas:** id_canal, nome_canal, tipo_canal, ativo, observacao

### 5. cadastro_produtos_api_dump.json
- **Formato:** Array JSON com objetos aninhados (product, pricing, attributes)
- **Registros:** 72 (71 únicos após deduplicação)
- **Observação:** estrutura aninhada foi achatada na Silver

### 6. logistica_entregas.json
- **Formato:** Array JSON com objetos aninhados (carrier, timestamps, destination)
- **Registros:** 322
- **Relacionamentos:** order_ref → pedidos_cabecalho

### 7. atendimento_ocorrencias.ndjson
- **Formato:** NDJSON — um objeto JSON por linha, sem necessidade de `multiLine`
- **Registros:** 270
- **Colunas:** ticket_id, order_id, event_type, created_at, severity, status

### 8. vendedores.csv
- **Separador:** ponto-e-vírgula
- **Registros:** 42 (40 únicos após deduplicação)
- **Colunas:** seller_id, seller_name, canal_id, regional_code, hire_date, status

### 9. legado_regioes_pipe.txt
- **Separador:** pipe (`|`)
- **Registros:** 8 (5 válidos após tratamento)
- **Colunas:** regional_code, regional_name, state, manager_name, active_flag

---

## Problemas de Qualidade Identificados e Tratamentos

Essa foi a parte que mais exigiu atenção. Os dados tinham problemas em praticamente todas as fontes — alguns esperados para dados legados, outros mais sutis. Documentei cada um com a decisão tomada e o motivo.

### Datas em múltiplos formatos

**Impacto:** Sem normalização, ordenação e cálculos temporais ficam comprometidos.  
**Ocorrência:** `order_date` e `promised_date` chegavam em três formatos diferentes na mesma coluna: `yyyy-MM-dd`, `yyyy/MM/dd` e `dd/MM/yyyy`.  
**Tratamento:** Criei a função `parse_date_multiformat()` usando `COALESCE` com `try_to_timestamp` via `expr` para cada formato — o Photon engine exige essa abordagem para não lançar exceção em valores que não encaixam no padrão. Registros que falham em todos os formatos ficam como `null` com flag rastreável.

### Valores numéricos com vírgula como separador decimal

**Impacto:** Cast direto para `DOUBLE` falha com erro de parse.  
**Ocorrência:** Campos como `gross_amount`, `unit_price`, `total_item` chegavam com vírgula no lugar do ponto (`1274,78` ao invés de `1274.78`).  
**Tratamento:** `try_cast(replace(col, ',', '.') AS DOUBLE)` em todos os campos numéricos afetados.

### Status inconsistentes (maiúsculas, underscores, variações)

**Impacto:** Agrupamentos incorretos e filtros que perdem registros.  
**Ocorrência:** `status_order` com `"em separacao"`, `"EM_SEPARACAO"`, `"faturado"`, `"Faturado"` — todas representando o mesmo estado.  
**Tratamento:** `UPPER(REGEXP_REPLACE(col, '_', ' '))` para pedidos; `CASE WHEN LOWER(TRIM(col)) IN (...)` para os demais campos.  
**Afetou:** `item_status`, `status_cliente`, `porte`, `ativo` nos canais, `delivery_status`, `event_type`, `severity`.

### Nulos em status_order (64 registros — ~16%)

**Decisão:** Mantidos com flag `flag_status_nulo = true`.  
**Motivo:** Descartar 16% dos pedidos distorceria as análises de volume e receita. Status nulo indica dado ausente na origem, não um estado de negócio como cancelamento.

### item_status nulo (258 registros — ~26%)

**Decisão:** Nulo interpretado como "Ativo".  
**Motivo:** O campo só é preenchido quando há cancelamento explícito. Ausência de cancelamento é, portanto, o comportamento normal do sistema de origem.

### 77 divergências em total_item vs quantity × unit_price

**Impacto:** Usar `total_item` original resultaria em receita incorreta para ~8% dos itens.  
**Tratamento:** `total_item` recalculado como `ROUND(quantity * unit_price, 2)`. Os registros originalmente divergentes receberam `flag_total_divergente = true` para rastreabilidade.

### 1 gross_amount nulo em pedidos

**Tratamento:** Imputado com a soma dos itens do pedido, que era a única fonte de verdade disponível.

### Estado dos clientes com 18 variações

**Exemplos:** `"santa catarina"`, `"Santa Catarina"`, `"Sta Catarina"`, `"S. Catarina"`, `"SC"` — todos representando o mesmo estado.  
**Tratamento:** Mapa explícito de normalização para sigla de 2 letras seguindo o padrão IBGE.

### Canais duplicados e inconsistentes

- **CH05** aparece duas vezes com nomes divergentes. Mantida a primeira ocorrência; a segunda descartada.
- **CH06** sem nome: substituído por `"NOME_AUSENTE"` para não gerar nulo silencioso.
- **ch07** com id em minúsculas: normalizado para `"CH07"`.
- Campo `ativo` com variações `sim/Sim/SIM/nao`: convertido para boolean.

### Vendedores duplicados

- **V004** aparecia duas vezes: uma com `canal_id = CH02` (válido) e outra com `CH99` (inexistente). Mantido o registro com canal válido.
- **V008** aparecia como `"Vendedor 8 duplicado"`: descartado.
- `regional_code = 'sul'` normalizado para `'S'`.
- `canal_id = 'CH99'` sinalizado com `flag_canal_invalido = true` — não existe na tabela de canais.

### Produtos duplicados

- 1 `product_id` duplicado: mantido o registro com `updated_at` mais recente.
- 21 registros com `status = null`: normalizados para `"Desconhecido"`.

### Regiões legado

- Entrada `regional_code = 'sul'` (minúsculas) com nome redundante `'Região Sul'`: deduplicada mantendo o nome mais curto (`'Sul'`).
- Entrada `XX` sem nome e `active_flag = 0`: descartada por ser inválida e inativa.

### Logística — delivery_status

- Variações `"delivered"` e `"Delivered"` → `"Entregue"`; `"in_transit"` → `"Em Trânsito"`.
- 60 nulos normalizados para `"Desconhecido"`.
- 72 registros sem `carrier.name`: mantidos como `null` com `flag_carrier_ausente = true`.

### Ocorrências — nulos generalizados

- `event_type`: 37 nulos (~14%). Normalizados para `"Desconhecido"`.
- `severity`: 59 nulos (~22%). Normalizados para `"Desconhecida"`.
- `status`: 75 nulos (~28%). Normalizados para `"Desconhecido"`.

---

## Modelo de Dados — Gold Layer

A camada Gold foi pensada para quem vai consumir os dados, não para quem vai construí-los. Isso significa colunas com nomes claros, atributos de contexto já disponíveis dentro das tabelas fato e sem necessidade de joins para os casos de uso mais comuns.

### Tabelas Dimensão

| Tabela | Granularidade | PK | Registros |
|---|---|---|---|
| `gold.dim_clientes` | 1 por cliente | `id_cliente` | ~183 |
| `gold.dim_produtos` | 1 por produto | `id_produto` | ~71 |
| `gold.dim_vendedores` | 1 por vendedor | `id_vendedor` | ~40 |
| `gold.dim_canais` | 1 por canal | `id_canal` | ~7 |
| `gold.dim_regioes` | 1 por região | `id_regional` | 5 |

### Tabelas Fato

| Tabela | Granularidade | FK Principais |
|---|---|---|
| `gold.fct_pedidos` | 1 por pedido | `id_cliente`, `id_vendedor`, `canal_id`, `regional_code` |
| `gold.fct_itens_pedido` | 1 por item de pedido | `id_pedido`, `id_produto` |
| `gold.fct_entregas` | 1 por entrega | `id_pedido` |
| `gold.fct_ocorrencias` | 1 por ticket | `id_pedido`, `id_cliente` |

### Mart Analítico

| Tabela | Granularidade | Uso |
|---|---|---|
| `gold.mart_desempenho_mensal` | 1 por combinação mês/canal/região/segmento | KPIs pré-agregados para dashboards |

### Relacionamentos

```
dim_canais ────────────────┐
dim_regioes ───────────────┤
                           ▼
dim_clientes ──┐      dim_vendedores
               │           │
               ▼           ▼
            fct_pedidos ◄──┘
               │
               ├──→ fct_itens_pedido ──→ dim_produtos
               ├──→ fct_entregas
               └──→ fct_ocorrencias ──→ dim_clientes
```

**Observação importante:** O pedido não registra canal diretamente. O `canal_id` presente em `fct_pedidos` é herdado do vendedor responsável pelo pedido. Essa é uma limitação da fonte que está documentada nas premissas e pode ser corrigida com uma mudança no sistema de origem.

---

## Premissas Adotadas

Toda decisão não óbvia foi documentada aqui. Nenhuma foi tomada de forma arbitrária.

1. **item_status nulo = Ativo:** O campo só aparece preenchido quando há cancelamento. Ausência de valor significa item em fluxo normal.
2. **gross_amount nulo:** Imputado pela soma dos itens — única fonte de verdade disponível para esse caso.
3. **Canal do pedido via vendedor:** Não há campo de canal no pedido. O canal herdado do vendedor é a melhor aproximação disponível nos dados atuais.
4. **Deduplicação de vendedores:** Quando havia conflito, preferiu-se o registro com canal válido e data de admissão mais antiga.
5. **Deduplicação de canais:** Para o CH05 duplicado, manteve-se o registro sem observação de duplicidade.
6. **data_cadastro com datas futuras:** Mantidas com flag. Provavelmente erros de digitação na origem, mas não há como confirmar sem consultar a fonte.
7. **status_order nulo ≠ cancelado:** Pedidos sem status são mantidos e sinalizados. Tratá-los como cancelados distorceria as taxas.
8. **total_item recalculado:** Quando o valor original diverge de `qty × price`, o valor calculado é preferido — é derivável de campos primários e auditável.

---

## Limitações

Toda solução tem restrições. Prefiro deixá-las explícitas do que deixar o analista descobrir no momento errado.

1. **Período limitado:** Os dados cobrem principalmente 2025. Análises de tendência de longo prazo não são possíveis com essa base.
2. **Canal indireto:** O canal é derivado do vendedor, não do pedido. Se um vendedor atuar em múltiplos canais, essa lógica perde precisão.
3. **Transportadora ausente em ~22% das entregas:** Análises segmentadas por transportadora são parciais.
4. **~28% dos tickets sem status definido:** Métricas de resolução de atendimento devem ser lidas com cautela.
5. **Sem histórico de mudanças (SCD):** Se um cliente mudar de segmento ou um produto mudar de categoria, o modelo atual não registra essa evolução. Em produção, isso precisaria de SCD Tipo 2.

---

## Sugestões de Evolução

O que foi entregue é uma base sólida, mas há caminhos claros para amadurecer a solução.

1. **Qualidade de dados automatizada:** Adicionar testes após cada execução — contagens mínimas, percentual de nulos aceitável, integridade referencial. Ferramentas como Great Expectations ou um framework próprio resolveriam isso.
2. **SCD Tipo 2 nas dimensões:** Para capturar mudanças históricas em clientes e produtos ao longo do tempo.
3. **Orquestração:** Substituir a execução manual dos notebooks por Databricks Workflows ou Airflow, com dependências entre etapas e alertas de falha.
4. **Streaming incremental:** Pedidos e ocorrências têm atualização frequente. Migrar para Delta Live Tables reduziria a latência significativamente.
5. **Incluir canal no pedido na origem:** A principal limitação do modelo atual poderia ser eliminada com uma mudança simples no ERP — adicionar o `canal_id` diretamente no registro do pedido.
6. **Mart de atendimento:** Um `mart_atendimento_operacional` com SLAs de resolução de tickets seria útil para a equipe de operações.

---

## Instruções de Execução

### Pré-requisitos
- Databricks com Unity Catalog habilitado
- Catálogo `sandbox` criado
- Schema `case_de` criado dentro do catálogo `sandbox`
- Volume `sources` criado em `sandbox.case_de`
- Arquivos fonte carregados no volume `/Volumes/sandbox/case_de/sources/`

### Ordem de execução

```
1. notebooks/01_bronze_ingestion.ipynb   — ingestão das 9 fontes
2. notebooks/02_silver_transform.ipynb   — tratamento e normalização
3. notebooks/03_gold_analytical.ipynb    — modelo analítico + validações
```

Cada notebook começa com `spark.sql("USE CATALOG sandbox")`, então todas as tabelas são criadas em `sandbox.bronze.*`, `sandbox.silver.*` e `sandbox.gold.*`.

---

## Estrutura do Repositório

```
case-data-engineer/
├── README.md
├── docs/
│   ├── documentacao_tecnica.md      ← este arquivo
│   └── resumo_executivo.md
├── notebooks/
│   ├── 01_bronze_ingestion.ipynb
│   ├── 02_silver_transform.ipynb
│   └── 03_gold_analytical.ipynb
└── sources/                         ← arquivos fonte originais (não versionados)
    └── .gitkeep
```
