# Case Data Engineer

Pipeline de dados em Medallion Architecture (Bronze/Silver/Gold) com PySpark e Delta Tables no Databricks.

## Estrutura

notebooks/01_bronze_ingestion.ipynb — ingestão das 9 fontes

notebooks/02_silver_transform.ipynb — tratamento e normalização

notebooks/03_gold_analytical.ipynb  — modelo analítico + validações

docs/documentacao_tecnica.md

docs/resumo_executivo.md

sources/ — arquivos fonte (não versionados)


## Execução

1. Fazer upload dos arquivos fonte em /Volumes/sandbox/case_de/sources/
2. Rodar os notebooks em ordem: 01 → 02 → 03

## Ambiente

Databricks com Unity Catalog, catálogo sandbox, PySpark e Delta Tables.
