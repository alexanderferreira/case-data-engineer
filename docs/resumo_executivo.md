# Resumo Executivo Técnico
## Case Data Engineer — Plataforma Analítica de Operações Comerciais

---

## 1. O que foi construído

Parti de nove fontes brutas com formatos completamente diferentes — CSVs com separadores distintos, XLSXs, JSONs aninhados, NDJSON e até um arquivo pipe-delimited de sistema legado — e transformei tudo isso em uma base analítica estruturada, pronta para o Analista de BI trabalhar sem precisar retratar dados ou fazer joins complexos.

A solução foi organizada em três camadas:

| Camada | O que faz | O que entrega |
|---|---|---|
| Bronze | Ingere as fontes como estão, sem tocar nos dados | 9 tabelas Delta com os dados brutos preservados |
| Silver | Trata inconsistências, normaliza tipos e resolve duplicatas | 9 tabelas limpas (5 dimensões + 4 fatos) |
| Gold | Monta o modelo analítico orientado ao consumo | 10 tabelas finais (5 dim + 4 fct + 1 mart) |

No total, cerca de 2.300 registros brutos foram processados, cobrindo pedidos, itens, clientes, produtos, vendedores, canais, regiões, entregas e ocorrências de atendimento.

---

## 2. Principais decisões técnicas

**Por que Medallion Architecture?**
A separação em camadas não foi uma escolha estética. É o que permite rastrear qualquer dado do dashboard até o arquivo bruto original. Se um número parecer errado, você sabe exatamente onde investigar. A Bronze preserva a fonte; a Silver documenta o que foi corrigido; a Gold entrega o que o analista precisa.

**Desnormalização no Gold**
Em vez de obrigar o analista a cruzar tabelas para cada consulta, as tabelas fato já carregam os atributos de contexto mais usados — canal, região e segmento do cliente ficam dentro do próprio registro de pedido. As dimensões continuam disponíveis para quem precisar de filtros mais detalhados.

**Canal herdado do vendedor**
O ERP não registra o canal de venda diretamente no pedido. A solução foi herdar o canal do vendedor responsável. É uma aproximação válida para a maioria dos casos, mas está documentada como premissa — e a correção ideal seria adicionar esse campo na origem.

**total_item recalculado**
Cerca de 8% dos itens tinham o valor de `total_item` divergindo de `quantidade × preço_unitário`. Como os campos primários são auditáveis e o campo calculado não é, optei por recalcular. Os registros originalmente divergentes foram sinalizados para rastreabilidade.

---

## 3. Principais desafios encontrados nos dados

Os dados tinham problemas em praticamente todas as fontes. Alguns eram esperados para sistemas legados; outros surgiram só na exploração.

| Problema | Escala | Como tratei |
|---|---|---|
| Datas em 3 formatos diferentes | Todos os arquivos com datas | Parse multi-formato com `try_to_timestamp` via `expr` |
| Valores numéricos com vírgula decimal | Campos de valor em pedidos e itens | `replace` + `try_cast` antes do cast para Double |
| Status inconsistentes (case, underscores) | 6 colunas em 5 fontes | Normalização para valores canônicos via `UPPER/LOWER + TRIM` |
| `status_order` nulo em 16% dos pedidos | pedidos_cabecalho | Mantidos com flag — descartar distorceria volume e receita |
| `item_status` nulo em 26% dos itens | pedidos_itens | Interpretado como "Ativo" — ausência de cancelamento é o estado padrão |
| 77 divergências em `total_item` | pedidos_itens | Recalculado como `qty × price`, original sinalizado |
| Estado dos clientes com 18 variações | crm_clientes | Mapa explícito para sigla UF padrão IBGE |
| Duplicatas em canais, vendedores e produtos | 3 fontes | Deduplicação com critério documentado para cada caso |
| `delivery_status` nulo em 19% das entregas | logistica | Normalizado para "Desconhecido" com flag |

---

## 4. Visão geral do modelo final

```
                    ┌─────────────────┐
                    │  dim_clientes   │
                    │  (183 clientes) │
                    └────────┬────────┘
                             │ id_cliente
┌──────────────┐    ┌────────▼────────────────────────────────┐
│ dim_vendedores│──→│              fct_pedidos                │
│ (canal e     │    │  (403 pedidos — receita, status,        │
│  região      │    │   canal, região, segmento do cliente)   │
│  resolvidos) │    └──┬──────────────────────────┬──────────┘
└──────────────┘       │ id_pedido                │ id_pedido
                ┌──────▼──────────┐     ┌─────────▼──────────┐
                │ fct_itens_pedido│     │   fct_entregas     │
                │ (995 itens com  │     │   (322 entregas,   │
                │  produto e      │     │    flag_atraso,    │
                │  categoria)     │     │    dias_atraso)    │
                └──────┬──────────┘     └────────────────────┘
                       │ id_produto
                ┌──────▼──────────┐
                │  dim_produtos   │
                │  (71 produtos)  │
                └─────────────────┘

  + fct_ocorrencias (270 tickets) ligada a fct_pedidos e dim_clientes
  + dim_canais e dim_regioes disponíveis para filtros standalone
  + mart_desempenho_mensal com KPIs pré-agregados por mês/canal/região/segmento
```

Com esse modelo, as principais perguntas do negócio podem ser respondidas diretamente:

- Receita líquida por canal, região, segmento e período — via `fct_pedidos` ou `mart_desempenho_mensal`
- Ticket médio e taxa de cancelamento — via `fct_pedidos`
- Produtos e categorias que mais geram receita — via `fct_itens_pedido`
- Taxa de atraso nas entregas e quantos dias em média — via `fct_entregas`
- Volume e tipo de ocorrências de atendimento — via `fct_ocorrencias`

---

## 5. Próximos passos

A solução entregue é funcional e cobre os requisitos do case, mas há pontos claros onde ela pode evoluir.

O mais urgente na minha avaliação seria **reportar os problemas de qualidade ao time de origem** — datas em formatos mistos, status nulos e divergências em valores são sintomas de algo que deveria ser corrigido no ERP ou no CRM, não apenas tratado no pipeline.

Na sequência, os passos mais impactantes seriam:

1. **Adicionar `canal_id` diretamente no pedido** — elimina a maior limitação do modelo atual sem nenhuma mudança no pipeline.
2. **Orquestração com Databricks Workflows** — substituir a execução manual por um fluxo automatizado com dependências e alertas.
3. **Testes de qualidade automatizados** — contagens mínimas, limites de nulos e integridade referencial verificados a cada execução.
4. **SCD Tipo 2 nas dimensões** — para capturar mudanças históricas em clientes e produtos ao longo do tempo.
5. **Streaming incremental** — pedidos e ocorrências têm alta frequência de atualização; Delta Live Tables reduziria significativamente a latência.
