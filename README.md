# NYC Taxi Data Pipeline — Roteiro do Projeto

Projeto de portfólio em Data Engineering usando o dataset **NYC Yellow Taxi Trip Data**, com arquitetura em camadas (raw → staging → curated) e stack containerizada.

**Sugestão de nome para o repositório:** `nyc-taxi-data-pipeline`

---

## Arquitetura

```
Fonte (NYC TLC / Kaggle)
      │
      ▼
  Raw Layer (MinIO / S3)
      │
      ▼
  Staging (PySpark - limpeza)
      │
      ▼
  Curated (Parquet particionado + modelo estrela)
      │
      ▼
  Data Warehouse (Postgres + dbt)
      │
      ▼
  Dashboard (Metabase / Streamlit)

  Orquestração: Airflow (ponta a ponta)
```

---

## Etapas

### 1. Definição de escopo e arquitetura
Desenhar o fluxo completo (fonte → raw → staging → curated → consumo) e escrever no README do repositório.
**Ferramentas:** Markdown, draw.io (opcional)

### 2. Ingestão (extract)
Script Python que baixa os arquivos Parquet mensais direto do site oficial da NYC TLC (ou via Kaggle API), parametrizado por mês/ano — pensando em ingestão incremental, não tudo de uma vez.
**Ferramentas:** Python, `requests` ou biblioteca `kaggle`

### 3. Data lake local (camada raw)
Subir um MinIO (S3 compatível) via Docker e enviar os arquivos brutos, particionados por pasta ano/mês (ex: `raw/2024/01/`).
**Ferramentas:** Docker, MinIO, boto3

### 4. Transformação e limpeza (staging)
Tratar os dados: remover corridas com coordenadas inválidas, tarifas negativas, durações absurdas, nulos em campos-chave. Como o volume é grande (milhões de linhas/mês), usar processamento distribuído.
**Ferramentas:** PySpark (local mode)

### 5. Armazenamento otimizado (curated)
Salvar os dados limpos em Parquet particionado por ano/mês/dia — essencial pra performance de leitura em datasets grandes.
**Ferramentas:** PySpark, Parquet

### 6. Modelagem dimensional
Criar um esquema estrela: tabela fato (corridas) e dimensões (data, zona de embarque/desembarque, tipo de pagamento, vendor). Carregar no data warehouse.
**Ferramentas:** Postgres (Docker), dbt

### 7. Otimização de queries
Aplicar particionamento e índices nas tabelas do warehouse, e comparar performance de queries antes/depois.
**Ferramentas:** SQL, `EXPLAIN ANALYZE` (Postgres)

### 8. Orquestração
Colocar o pipeline inteiro (extract → raw → staging → curated) rodando via Airflow, com uma DAG que processa um mês por vez, simulando ingestão incremental real.
**Ferramentas:** Apache Airflow (Docker)

### 9. Qualidade de dados
Testes automáticos de integridade (ex: nenhuma corrida sem zona válida, distância não pode ser negativa).
**Ferramentas:** dbt tests ou Great Expectations

### 10. Visualização e documentação final
Dashboard simples com métricas (corridas por hora do dia, tarifa média por zona, gorjeta média por tipo de pagamento) + README final com arquitetura, decisões técnicas e como rodar tudo com um único comando.
**Ferramentas:** Metabase ou Streamlit, `docker-compose up`

---

## Checklist rápido

- [ ] Etapa 1 — Escopo e arquitetura
- [ ] Etapa 2 — Ingestão
- [ ] Etapa 3 — Data lake (raw)
- [ ] Etapa 4 — Transformação/limpeza
- [ ] Etapa 5 — Armazenamento otimizado
- [ ] Etapa 6 — Modelagem dimensional
- [ ] Etapa 7 — Otimização de queries
- [ ] Etapa 8 — Orquestração
- [ ] Etapa 9 — Qualidade de dados
- [ ] Etapa 10 — Visualização e documentação final
