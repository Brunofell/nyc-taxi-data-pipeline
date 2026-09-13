# NYC Taxi Data Pipeline — Roteiro do Projeto (versão AWS Free Tier)

Projeto de portfólio em Data Engineering usando o dataset **NYC Yellow Taxi Trip Data** (Kaggle), com arquitetura em camadas (raw → staging → curated) rodando em serviços reais da AWS, dentro dos limites do Free Tier.

**Sugestão de nome para o repositório:** `nyc-taxi-data-pipeline`

⚠️ **Atenção com custos:** o Free Tier da AWS cobre 12 meses pra a maioria dos serviços aqui (S3, RDS). Sempre monitore o **AWS Billing Dashboard** e configure um **Billing Alarm**. No fim do projeto, tem uma etapa de teardown (desligar tudo) — não pule ela.

---

## Arquitetura

```
Kaggle (fonte)
      │
      ▼
  S3 - raw/           (dados brutos, intocados)
      │
      ▼
  Transformação local (Python/Pandas ou PySpark, rodando no seu PC/Docker)
      │
      ▼
  S3 - staging/        (dados limpos)
      │
      ▼
  S3 - curated/         (Parquet particionado, modelo pronto)
      │
      ▼
  RDS PostgreSQL (Free Tier)  →  dbt (schema estrela: fato + dimensões)
      │
      ▼
  Dashboard (Streamlit/Metabase local, conectado no RDS)

  Orquestração: Airflow (Docker local, mas operando sobre os recursos da AWS)
```

---

## Etapas

### 1. Definição de escopo e arquitetura
Desenhar o fluxo completo (fonte → raw → staging → curated → consumo) e escrever no README do repositório, já deixando claro que a infraestrutura é AWS.
**Ferramentas:** Markdown, draw.io (opcional)

### 2. Configuração da conta AWS e IAM
Criar um usuário IAM próprio pro projeto (nunca use o root da conta), com uma política de permissões restrita só ao necessário (S3, RDS, e Athena/Glue se for usar). Configurar a AWS CLI localmente com esse usuário. Revisar os limites do Free Tier antes de criar qualquer recurso.
**Ferramentas:** AWS IAM, AWS CLI

### 3. Ingestão (extract)
Script Python que baixa o CSV do Kaggle (via API) e sobe pro bucket S3, na pasta/prefixo `raw/`, sem alterar nada nos dados.
**Ferramentas:** Python, biblioteca `kaggle`, `boto3`

### 4. Data lake (camada raw) no S3
Criar o bucket S3 com prefixos organizados (`raw/`, `staging/`, `curated/`) simulando zonas de um data lake dentro do mesmo bucket. Ativar versionamento básico do bucket como boa prática.
**Ferramentas:** AWS S3

### 5. Transformação e limpeza (staging)
Baixar os dados do S3 raw pra processar localmente: remover corridas com coordenadas inválidas, tarifas negativas, durações absurdas, nulos em campos-chave. Subir o resultado limpo pro S3 em `staging/`.
**Ferramentas:** Python (Pandas ou PySpark local), `boto3`

### 6. Armazenamento otimizado (curated)
Salvar os dados limpos como Parquet particionado (por ano/mês) no prefixo `curated/` do S3 — formato colunar essencial pra performance e pra reduzir custo se for usar Athena depois.
**Ferramentas:** PySpark/Pandas + PyArrow, S3

### 7. Modelagem dimensional (data warehouse)
Subir uma instância **RDS PostgreSQL** (free tier: db.t3.micro, 750h/mês, 20GB). Carregar os dados do S3 curated pra lá e modelar um esquema estrela: tabela fato (corridas) e dimensões (data, zona, tipo de pagamento, vendor). Usar dbt pra essa camada de transformação SQL.
**Ferramentas:** AWS RDS (PostgreSQL), dbt, `psycopg2`/`boto3`

### 8. Otimização de queries
Comparar performance de queries no RDS antes/depois de criar índices, usando `EXPLAIN ANALYZE`. Se quiser ir além, use o **Athena** pra rodar a mesma consulta direto em cima do Parquet no S3 (sem carregar no RDS) e compare custo/velocidade entre as duas abordagens — ótimo ponto pra falar em entrevista.
**Ferramentas:** SQL (`EXPLAIN ANALYZE`), Amazon Athena (opcional, cobra por dado escaneado — fique de olho)

### 9. Orquestração
Rodar o pipeline inteiro (extract → raw → staging → curated → load no RDS) via uma DAG no Airflow, local em Docker, mas com as tasks conversando com os recursos reais da AWS via `boto3`.
**Ferramentas:** Apache Airflow (Docker)

### 10. Qualidade de dados
Testes automáticos de integridade (ex: nenhuma corrida sem zona válida, distância não pode ser negativa), rodando como parte do dbt ou com Great Expectations antes da carga final.
**Ferramentas:** dbt tests ou Great Expectations

### 11. Visualização e documentação final
Dashboard simples (rodando local, conectado no RDS) com métricas (corridas por hora do dia, tarifa média por zona, gorjeta média por tipo de pagamento). Finalizar o README com arquitetura, decisões técnicas e custos observados.
**Ferramentas:** Streamlit ou Metabase (Docker local) conectado ao RDS

### 12. Teardown (desligar recursos)
Ao concluir (ou pausar) o projeto, deletar/parar a instância RDS, esvaziar e remover o bucket S3 (ou só os dados, se quiser manter o bucket), e revisar o Billing Dashboard pra garantir que nada ficou cobrando. Documentar esse passo no README também — mostra maturidade de custo, algo que times de dados valorizam.
**Ferramentas:** AWS Console / AWS CLI

---

## Checklist rápido

- [ ] Etapa 1 — Escopo e arquitetura
- [ ] Etapa 2 — Conta AWS e IAM
- [ ] Etapa 3 — Ingestão
- [ ] Etapa 4 — Data lake (raw) no S3
- [ ] Etapa 5 — Transformação/limpeza (staging)
- [ ] Etapa 6 — Armazenamento otimizado (curated)
- [ ] Etapa 7 — Modelagem dimensional (RDS + dbt)
- [ ] Etapa 8 — Otimização de queries
- [ ] Etapa 9 — Orquestração (Airflow)
- [ ] Etapa 10 — Qualidade de dados
- [ ] Etapa 11 — Visualização e documentação final
- [ ] Etapa 12 — Teardown (desligar recursos AWS)

---

## Estrutura de pastas do repositório

```
nyc-taxi-data-pipeline/
│
├── docker-compose.yml         # sobe Airflow e o dashboard local (S3/RDS ficam na AWS, não aqui)
├── .env.example                # modelo das variáveis de ambiente (NUNCA commitar .env real com chaves)
├── .gitignore                  # ignora .env, credenciais e dados locais
├── requirements.txt
├── README.md                   # documentação, diagrama de arquitetura e notas de custo
│
├── infra/                       # ETAPA 2 — Conta AWS e IAM
│   └── iam_policy.json          # política IAM de permissão mínima usada no projeto
│
├── ingestion/                    # ETAPA 3 — Extract
│   └── extract_to_s3.py          # baixa do Kaggle e envia pro S3 (raw/)
│
├── transform/                     # ETAPA 5 e 6 — Staging e Curated
│   ├── clean_staging.py           # lê raw do S3, limpa, escreve em staging/
│   └── build_curated.py           # gera Parquet particionado em curated/
│
├── warehouse/                      # ETAPA 7 — Modelagem dimensional
│   ├── ddl/                        # scripts SQL de criação das tabelas fato/dimensão
│   └── load_to_rds.py              # carrega curated (S3) para o RDS
│
├── dbt_project/                     # ETAPA 7 e 10 — Transformação SQL + testes
│   ├── models/
│   │   ├── staging/
│   │   └── marts/
│   └── tests/
│
├── analysis/                         # ETAPA 8 — Otimização de queries
│   └── query_benchmarks.sql          # queries de teste + EXPLAIN ANALYZE / comparação com Athena
│
├── airflow/                           # ETAPA 9 — Orquestração
│   ├── dags/
│   │   └── taxi_pipeline_dag.py
│   └── Dockerfile
│
├── dashboards/                         # ETAPA 11 — Visualização
│   └── app.py                          # Streamlit conectado ao RDS
│
└── docs/
    └── architecture.png                # diagrama do fluxo completo
```

### Função de cada pasta

| Pasta | Função |
|---|---|
| `infra/` | Guarda a definição de permissões IAM — mostra que você pensa em segurança desde o início, não só em fazer o pipeline "funcionar". |
| `ingestion/` | Só extrai da fonte e sobe pro S3 raw. Não trata nada aqui. |
| `transform/` | Onde a limpeza acontece — separa staging (limpo) de curated (pronto pro modelo dimensional). |
| `warehouse/` | Define a estrutura do banco analítico (RDS) e faz a carga final vinda do S3. |
| `dbt_project/` | Centraliza a lógica de transformação em SQL versionado + os testes de qualidade. |
| `analysis/` | Evidência de que você pensou em performance/custo de query — ponto forte pra citar em entrevista. |
| `airflow/` | Amarra tudo numa sequência automatizada, mesmo os recursos estando na nuvem. |
| `dashboards/` | Camada de consumo — onde o valor de negócio aparece. |
| `docs/` | Registro visual da arquitetura pra quem for ler o repositório. |