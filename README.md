# 🏗️ Fintech ETL Pipeline + Data Warehouse

![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Airflow](https://img.shields.io/badge/Apache_Airflow-017CEE?style=for-the-badge&logo=apacheairflow&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-003B57?style=for-the-badge&logo=sqlite&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-150458?style=for-the-badge&logo=pandas&logoColor=white)
![Status](https://img.shields.io/badge/Status-Concluído-34d399?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-60a5fa?style=for-the-badge)

> Pipeline ETL completo para processamento de transações financeiras, com modelagem dimensional (Star Schema), orquestração via Apache Airflow e framework de qualidade de dados integrado.

---

## 📌 Visão Geral

Instituições financeiras processam milhões de transações diariamente. Este projeto implementa um pipeline ETL de ponta a ponta — da ingestão de múltiplas fontes até a carga em um Data Warehouse dimensional — com foco em **rastreabilidade**, **qualidade de dados** e **escalabilidade**.

### Resultados do Pipeline

| Métrica | Valor |
|:--------|------:|
| **Transações processadas/dia** | **150.000** |
| **Tempo de carga (fact table)** | **~43s** |
| **Checks de qualidade** | **7 automatizados** |
| **Uptime simulado (90 dias)** | **97%+** |
| **Tabelas no DW** | **4 (Star Schema)** |

---

## 🎯 Problema de Negócio

**Contexto:** Uma Fintech precisa consolidar transações de múltiplas fontes (APIs, CSVs, sistemas legados) em um repositório analítico centralizado para suportar decisões estratégicas.

**Desafios endereçados:**
- Dados inconsistentes entre fontes (moeda em formatos diferentes, booleans numpy não serializáveis, valores nulos)
- Necessidade de rastreabilidade total (checksum por extração)
- Garantia de qualidade antes da carga no DW
- Orquestração confiável com alertas em caso de falha

---

## 🗂️ Estrutura do Projeto

```
fintech-etl-pipeline/
│
├── 📓 notebooks/
│   └── fintech_etl_pipeline.ipynb    # Notebook principal completo
│
├── 📁 dags/
│   └── fintech_etl_dag.py            # DAG do Apache Airflow
│
├── 📁 src/
│   ├── extract.py                    # Classe DataExtractor
│   ├── transform.py                  # Classe TransactionTransformer
│   ├── load.py                       # Classe DataLoader
│   └── quality.py                    # Classe DataQualityValidator
│
├── 📁 data/
│   ├── raw/                          # Dados brutos (não versionados)
│   │   ├── transactions.csv
│   │   ├── customers.json
│   │   └── merchants.csv
│   ├── processed/                    # Dados transformados
│   └── warehouse/                    # Banco SQLite do DW
│       └── fintech_dw.db
│
├── 📁 logs/                          # Logs de execução
├── requirements.txt
├── .gitignore
└── README.md
```

---

## 🏛️ Arquitetura

```
╔═══════════════════════════════════════════════════════════════════════════╗
║              FINTECH ETL PIPELINE — ARQUITETURA                          ║
╠═══════════════════════════════════════════════════════════════════════════╣
║                                                                           ║
║  FONTES (EXTRACT)           STAGING (TRANSFORM)       DW (LOAD)          ║
║  ──────────────────         ──────────────────        ─────────────────  ║
║  📄 transactions.csv  ──►  🔧 clean_transactions ──► fact_transactions   ║
║  📄 customers.json    ──►  🔧 clean_customers    ──► dim_customers        ║
║  📄 merchants.csv     ──►  🔧 clean_merchants    ──► dim_merchants        ║
║                            🔧 build_date_dim     ──► dim_date             ║
║                            🔧 data_quality_checks                         ║
║                                                                           ║
║  ORQUESTRAÇÃO (AIRFLOW DAG — daily @ 02:00 BRT)                          ║
║  ─────────────────────────────────────────────────────────────────────── ║
║  extract_sources ──► validate_raw ──► transform_all ──► load_dw          ║
║       │                                                    │             ║
║       └──► alert_on_failure              quality_report ◄──┘             ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

### Modelo Dimensional — Star Schema

```
                    ┌─────────────────┐
                    │    dim_date     │
                    │─────────────────│
                    │ date_key (PK)   │
                    │ full_date       │
                    │ year / quarter  │
                    │ month / week    │
                    │ is_weekend      │
                    └────────┬────────┘
                             │
 ┌──────────────────┐        │        ┌──────────────────┐
 │  dim_customers   │        │        │  dim_merchants   │
 │──────────────────│        │        │──────────────────│
 │ customer_id (PK) ├────────┤        │ merchant_id (PK) │
 │ name / email     │        │        │ name / category  │
 │ age / age_group  │   ┌────┴─────┐  │ city / state     │
 │ city / state     │   │  fact_   │  │ mdr_rate         │
 │ income_bracket   ├───┤transact- ├──┤ is_active        │
 │ is_active        │   │ions      │  └──────────────────┘
 └──────────────────┘   │──────────│
                        │ amount   │
                        │ status   │
                        │ payment_ │
                        │ type     │
                        │ device   │
                        └──────────┘
```

---

## 🔬 Metodologia

### Pipeline ETL Detalhado

**EXTRACT**
- Leitura de CSV e JSON com logging estruturado
- Geração de checksum MD5 por fonte para rastreabilidade
- Registro de metadados (linhas, colunas, timestamp)

**TRANSFORM**
- Classe `TransactionTransformer` com método chaining
- Padronização de moeda (BRL), tratamento de nulos, validação de amounts
- Feature engineering: `time_of_day`, `is_high_value`, `is_installment`, `installment_amount`
- Construção das dimensões: `dim_customers`, `dim_merchants`, `dim_date` (366 registros)

**LOAD**
- Schema com DDL explícito, foreign keys e índices analíticos
- Carga em chunks de 10.000 registros para eficiência de memória
- Audit log com tempo de carga por tabela

**QUALITY**
- 7 checks automatizados: row count, null rate, integridade referencial, approval rate
- Relatório de PASS/FAIL com thresholds configuráveis

### Boas Práticas Implementadas

| Prática | Implementação |
|:--------|:-------------|
| **Rastreabilidade** | Checksum MD5 em cada extração |
| **Separação de responsabilidades** | Classes independentes por etapa ETL |
| **Idempotência** | Carga com `replace` para reprocessamento seguro |
| **Integridade referencial** | Foreign keys no schema dimensional |
| **Observabilidade** | Logging estruturado com timestamp e nível |
| **Alertas proativos** | EmailOperator no Airflow com `TriggerRule.ONE_FAILED` |
| **Resiliência** | Retries configurados na DAG (2x com delay de 5min) |

---

## 📊 Resultados dos Logs de Execução

```
2026-03-11 14:04:15 | INFO | EXTRACT | CSV | transactions.csv
2026-03-11 14:04:15 | INFO |   → 150,000 linhas | checksum: c3d8340bb1
2026-03-11 14:04:15 | INFO | EXTRACT | JSON | customers.json
2026-03-11 14:04:15 | INFO |   → 5,000 linhas | checksum: 08e52e8fd5
2026-03-11 14:04:15 | INFO | EXTRACT | CSV | merchants.csv
2026-03-11 14:04:15 | INFO |   → 300 linhas | checksum: 51e33e4c4b
2026-03-11 14:10:13 | INFO | LOAD | dim_date         | 366 linhas → 0.12s
2026-03-11 14:10:13 | INFO | LOAD | dim_customers    | 5,000 linhas → 1.34s
2026-03-11 14:10:14 | INFO | LOAD | dim_merchants    | 300 linhas → 0.07s
2026-03-11 14:10:14 | INFO | LOAD | fact_transactions | 150,000 linhas → 42.71s
```

---

## ⚙️ Como Executar

### 1. Clone o repositório
```bash
git clone https://github.com/GabrielAlessi/fintech-etl-pipeline.git
cd fintech-etl-pipeline
```

### 2. Crie o ambiente virtual
```bash
python -m venv venv
source venv/bin/activate        # Linux/Mac
# venv\Scripts\activate         # Windows
```

### 3. Instale as dependências
```bash
pip install -r requirements.txt
```

### 4. Execute o notebook
```bash
jupyter notebook notebooks/fintech_etl_pipeline.ipynb
```

> **💡 Kaggle:** O notebook está disponível publicamente no Kaggle com todos os outputs já renderizados — sem necessidade de executar localmente.

### 5. Executar com Airflow (opcional)
```bash
# Inicializar o banco do Airflow
airflow db init

# Copiar a DAG
cp dags/fintech_etl_dag.py ~/airflow/dags/

# Subir o scheduler e webserver
airflow scheduler &
airflow webserver --port 8080
```

---

## 📦 Dependências

```
pandas>=1.5.0
numpy>=1.23.0
faker>=18.0.0
sqlalchemy>=2.0.0
matplotlib>=3.6.0
seaborn>=0.12.0
jupyter>=1.0.0
ipykernel>=6.0.0
apache-airflow>=2.6.0   # apenas para orquestração real
```

---

## 🚀 Próximos Passos

- [ ] **Migrar DW para PostgreSQL** em ambiente de produção (AWS RDS)
- [ ] **Implementar CDC** com Debezium para ingestão near real-time
- [ ] **Integrar com dbt** para transformações versionadas e testadas
- [ ] **Adicionar camada Gold** com agregações pré-computadas para o BI
- [ ] **Monitoramento com Grafana** — dashboard de SLA e latência
- [ ] **Data Catalog** com Apache Atlas para governança e lineage

---

## 👨‍💻 Autor

**Gabriel Alessi Naumann**  
Cientista de Dados Sênior

[![LinkedIn](https://img.shields.io/badge/LinkedIn-gabriel--alessi--naumann-0077B5?style=flat&logo=linkedin)](https://www.linkedin.com/in/gabriel-alessi-naumann/)
[![GitHub](https://img.shields.io/badge/GitHub-GabrielAlessi-181717?style=flat&logo=github)](https://github.com/GabrielAlessi)
[![Kaggle](https://img.shields.io/badge/Kaggle-gabrielalessinaumann-20BEFF?style=flat&logo=kaggle)](https://www.kaggle.com/gabrielalessinaumann)

---

## 📄 Licença

Este projeto está sob a licença MIT. Veja o arquivo [LICENSE](LICENSE) para mais detalhes.

---

*⭐ Se este projeto foi útil para você, considere deixar uma estrela no repositório!*
