<div align="center">

# 🛠️ Tech Stack — Competências & Referência Técnica

> Documentação pessoal de tecnologias utilizadas em projetos reais.  
> Aqui estão padrões, decisões de arquitetura, armadilhas conhecidas e boas práticas acumuladas ao longo de mais de 4 anos de trabalho.

[![Elasticsearch](https://img.shields.io/badge/Elasticsearch-005571?style=for-the-badge&logo=elasticsearch&logoColor=white)](#elasticsearch)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](#docker)
[![Apache NiFi](https://img.shields.io/badge/Apache%20NiFi-728E9B?style=for-the-badge&logo=apache&logoColor=white)](#apache-nifi)
[![SQL Server](https://img.shields.io/badge/SQL%20Server-CC2927?style=for-the-badge&logo=microsoftsqlserver&logoColor=white)](#sql-server)
[![Oracle](https://img.shields.io/badge/Oracle-F80000?style=for-the-badge&logo=oracle&logoColor=white)](#oracle-database)
[![TOTVS](https://img.shields.io/badge/TOTVS%20Protheus-EA1D2C?style=for-the-badge&logoColor=white)](#totvs-protheus)
[![OpenEdge](https://img.shields.io/badge/OpenEdge%20ABL-5B9BD5?style=for-the-badge&logoColor=white)](#openedge-progress)

</div>

---

## 📋 Índice

| Tecnologia | Nível | Foco principal |
|---|---|---|
| [Elasticsearch](#elasticsearch) | Avançado | Busca, agregações, ILM, clusters |
| [Docker](#docker) | Avançado | Containers, Compose, CI/CD |
| [Apache NiFi](#apache-nifi) | Intermediário/Avançado | ETL, pipelines de dados, integração |
| [SQL Server](#sql-server) | Avançado | Tuning, procedures, HA |
| [Oracle Database](#oracle-database) | Intermediário/Avançado | PL/SQL, RMAN, multitenant |
| [TOTVS Protheus](#totvs-protheus) | Avançado | ADVPL/TLPP, MVC, REST |
| [OpenEdge Progress](#openedge-progress) | Intermediário | ABL, PASOE, ProDataSet |

---

## Elasticsearch

### Visão Geral

O Elasticsearch é o coração de pipelines de busca e observabilidade. Trabalho com ele principalmente para indexação de grandes volumes de dados transacionais, logs de aplicação (stack ELK) e buscas full-text em produtos.

**Versão de referência:** 8.x  
**Stack comum:** Elasticsearch + Logstash + Kibana + Filebeat/Metricbeat

---

### Arquitetura de Cluster

```
┌────────────────────────────────────────────────┐
│                    CLUSTER                     │
│                                                │
│  ┌──────────────┐    ┌──────────────┐          │
│  │  Master Node │    │  Master Node │  (quorum)│
│  └──────────────┘    └──────────────┘          │
│                                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ Data Node│  │ Data Node│  │ Data Node│     │
│  │  Shard 0 │  │  Shard 1 │  │  Shard 2 │     │
│  │ (Primary)│  │ (Replica)│  │ (Primary)│     │
│  └──────────┘  └──────────┘  └──────────┘     │
│                                                │
│  ┌──────────────────┐                          │
│  │  Coordinating /  │  ← recebe requests       │
│  │  Ingest Node     │    e distribui           │
│  └──────────────────┘                          │
└────────────────────────────────────────────────┘
```

**Decisões que aprendi na prática:**
- Nunca usar menos de 3 master-eligible nodes em produção (split-brain)
- Separar data nodes de ingest nodes em clusters com volume alto de ingestão
- Dedicated master nodes evitam que o master fique sobrecarregado com queries

---

### Mapeamento (Mapping)

Mapping incorreto é a causa #1 de reindexação de dados em produção. Defina sempre **antes** de indexar.

```json
PUT /pedidos
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "index.refresh_interval": "5s"
  },
  "mappings": {
    "properties": {
      "id_pedido":       { "type": "keyword" },
      "cliente":         { "type": "keyword" },
      "descricao":       { "type": "text", "analyzer": "portuguese" },
      "valor_total":     { "type": "scaled_float", "scaling_factor": 100 },
      "status":          { "type": "keyword" },
      "data_criacao":    { "type": "date", "format": "yyyy-MM-dd'T'HH:mm:ss||epoch_millis" },
      "itens": {
        "type": "nested",
        "properties": {
          "produto":   { "type": "keyword" },
          "quantidade": { "type": "integer" }
        }
      }
    }
  }
}
```

> ⚠️ **Armadilha comum:** usar `text` em campos que serão usados em filtros/agregações. Use `keyword` para status, IDs e categorias. Use `text` apenas quando precisar de busca full-text.

---

### Queries Essenciais

```json
// Bool query — o canivete suíço do Elasticsearch
GET /pedidos/_search
{
  "query": {
    "bool": {
      "must":   [{ "match": { "descricao": "notebook" } }],
      "filter": [
        { "term":  { "status": "APROVADO" } },
        { "range": { "data_criacao": { "gte": "2024-01-01", "lte": "now" } } }
      ],
      "must_not": [{ "term": { "status": "CANCELADO" } }]
    }
  },
  "sort": [{ "data_criacao": "desc" }],
  "size": 20,
  "from": 0
}
```

```json
// Agregações — relatórios sem banco relacional
GET /pedidos/_search
{
  "size": 0,
  "aggs": {
    "por_status": {
      "terms": { "field": "status", "size": 10 },
      "aggs": {
        "total_valor": { "sum":  { "field": "valor_total" } },
        "ticket_medio": { "avg": { "field": "valor_total" } }
      }
    },
    "por_mes": {
      "date_histogram": {
        "field": "data_criacao",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      }
    }
  }
}
```

---

### Index Lifecycle Management (ILM)

Essencial para gestão de logs e dados com ciclo de vida definido.

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover":    { "max_size": "50gb", "max_age": "7d" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink":      { "number_of_shards": 1 },
          "forcemerge":  { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": { "freeze": {} }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

---

### Ingest Pipeline

```json
PUT _ingest/pipeline/enriquece-pedido
{
  "description": "Normaliza e enriquece pedidos antes de indexar",
  "processors": [
    { "uppercase": { "field": "status" } },
    { "date": {
        "field": "data_criacao_raw",
        "target_field": "data_criacao",
        "formats": ["dd/MM/yyyy HH:mm:ss"]
    }},
    { "remove": { "field": "data_criacao_raw" } },
    { "set": {
        "field": "indexado_em",
        "value": "{{_ingest.timestamp}}"
    }},
    { "script": {
        "lang": "painless",
        "source": "ctx.valor_com_taxa = ctx.valor_total * 1.1"
    }}
  ]
}
```

---

### Diagnóstico e Operação

```bash
# Saúde do cluster — RED é emergência, YELLOW é atenção
GET /_cluster/health?pretty

# Shards não alocados — investigar o porquê
GET /_cluster/allocation/explain

# Índices com mais espaço
GET /_cat/indices?v&s=store.size:desc&h=index,store.size,docs.count

# Threads em uso (útil para diagnosticar lentidão)
GET /_nodes/stats/thread_pool

# Hot threads — o que está consumindo CPU agora
GET /_nodes/hot_threads

# Forçar merge (somente fora de horário de pico)
POST /meu-index/_forcemerge?max_num_segments=1

# Reindex com transformação
POST /_reindex
{
  "source": { "index": "pedidos-v1" },
  "dest":   { "index": "pedidos-v2", "pipeline": "enriquece-pedido" }
}
```

---

### Referências
- [Elastic Docs 8.x](https://www.elastic.co/guide/en/elasticsearch/reference/current/)
- [Elastic Blog — Best Practices](https://www.elastic.co/blog/category/engineering)

---

## Docker

### Visão Geral

Docker é base do meu workflow de desenvolvimento e da infraestrutura de produção. Uso Compose para ambientes locais e pipelines de CI/CD com imagens customizadas para deploys consistentes.

---

### Dockerfile de Produção

Boas práticas que aplico em todo Dockerfile de aplicação:

```dockerfile
# ── Build stage ──────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

# Copiar manifests antes do código (cache de camadas)
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# ── Runtime stage ─────────────────────────────────────────
FROM node:20-alpine AS runtime

# Nunca rodar como root em produção
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copiar apenas o necessário do estágio anterior
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules

USER appuser

EXPOSE 3000

# Healthcheck — essencial para orquestração
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

**Por que multi-stage?**
- Imagem final não carrega compiladores, SDKs de build, arquivos de teste
- Superfície de ataque reduzida
- Imagens menores = deploys mais rápidos

---

### Docker Compose — Ambiente Completo

```yaml
# compose.yaml
name: minha-plataforma

services:

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      target: runtime
    image: minha-api:local
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DB_HOST: sqlserver
    env_file: .env
    depends_on:
      sqlserver:
        condition: service_healthy
      elasticsearch:
        condition: service_healthy
    networks:
      - backend
    deploy:
      resources:
        limits:
          memory: 512m

  elasticsearch:
    image: elasticsearch:8.13.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    healthcheck:
      test: curl -sf http://localhost:9200/_cluster/health || exit 1
      interval: 20s
      timeout: 5s
      retries: 5
    networks:
      - backend

  kibana:
    image: kibana:8.13.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks:
      - backend

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: Y
      SA_PASSWORD: "${SA_PASSWORD}"
      MSSQL_PID: Developer
    ports:
      - "1433:1433"
    volumes:
      - mssql_data:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "${SA_PASSWORD}" -Q "SELECT 1"
      interval: 15s
      timeout: 10s
      retries: 10
    networks:
      - backend

volumes:
  es_data:
  mssql_data:

networks:
  backend:
    driver: bridge
```

---

### Comandos do Dia a Dia

```bash
# Build ignorando cache (útil quando dependências mudam)
docker build --no-cache -t minha-app:latest .

# Inspecionar camadas e tamanho
docker history minha-app:latest

# Copiar arquivo de dentro do container
docker cp meu-container:/app/logs/app.log ./app.log

# Executar comando pontual sem subir serviço
docker compose run --rm api npm run migration

# Rebuild somente de um serviço específico
docker compose up -d --build api

# Ver uso de recursos em tempo real
docker stats

# Limpar tudo que não está em uso (cuidado em produção)
docker system prune --volumes -f

# Ver logs com timestamp e seguir
docker compose logs -f --timestamps api

# Inspecionar network — ver IPs dos containers
docker network inspect minha-plataforma_backend
```

---

### Boas Práticas — O Que Aprendi

| Prática | Motivo |
|---|---|
| Multi-stage build | Imagens menores, sem ferramentas de build |
| Usuário não-root | Segurança, princípio do menor privilégio |
| `.dockerignore` sempre | Evita copiar `node_modules`, `.git`, segredos |
| `COPY package*.json` antes do código | Aproveita cache de camadas |
| Healthcheck em todo serviço | Orchestrators precisam saber se o app está saudável |
| `depends_on: condition: service_healthy` | Garante ordem real de inicialização |
| Secrets via env_file ou secrets do Compose | Nunca hardcodar senhas no Dockerfile |
| Versão fixa de imagem (`8.13.0` não `latest`) | Builds reproduzíveis |

---

### Referências
- [Docker Docs](https://docs.docker.com/)
- [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)
- [Dive — analisar camadas de imagem](https://github.com/wagoodman/dive)

---

## Apache NiFi

### Visão Geral

Uso o NiFi como plataforma central de integração e ETL, conectando bancos relacionais (SQL Server, Oracle), APIs REST, filas de mensageria e o Elasticsearch. O ponto forte é a rastreabilidade: cada FlowFile tem proveniência completa.

**Versão de referência:** NiFi 1.24+ / 2.x  
**Stack comum:** NiFi + NiFi Registry + ZooKeeper (cluster)

---

### Arquitetura de um Fluxo ETL Típico

```
┌─────────────────────────────────────────────────────────────┐
│  PROCESS GROUP: Ingestão de Pedidos                         │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ QueryDatabase│───▶│ ConvertRecord│───▶│  SplitJson   │  │
│  │  Table (SQL) │    │ (SQL → JSON) │    │  (1 doc/FF)  │  │
│  └──────────────┘    └──────────────┘    └──────┬───────┘  │
│                                                 │           │
│                        ┌────────────────────────┘           │
│                        ▼                                    │
│                 ┌──────────────┐    ┌──────────────┐       │
│                 │ JoltTransform│───▶│  PutElastic  │       │
│                 │  (mapeamento)│    │  SearchRecord│       │
│                 └──────────────┘    └──────────────┘       │
│                                                             │
│  ── failure ──▶ [ PutFile / LogMessage / Slack Alert ]     │
└─────────────────────────────────────────────────────────────┘
```

---

### Processors Mais Utilizados

#### Leitura de Banco de Dados

```
Processor: QueryDatabaseTable
Configuração crítica:
  - Database Connection Pooling Service: DBCPConnectionPool (Controller Service)
  - Table Name: schema.tabela
  - Maximum-value Columns: dt_atualizacao   ← controla o watermark
  - Initial Load Strategy: Start at Beginning
  - Fetch Size: 10000

⚠️  Sempre usar Maximum-value Columns para não reprocessar tudo
⚠️  Separar DBCP por banco — não compartilhar pool entre fluxos críticos
```

#### Transformação de Registros

```
Processor: ConvertRecord
  Reader:  JsonTreeReader   ou  AvroReader
  Writer:  JsonRecordSetWriter

Processor: JoltTransformRecord
  Jolt Specification:
  [
    {
      "operation": "shift",
      "spec": {
        "CD_PEDIDO":    "id_pedido",
        "NM_CLIENTE":   "cliente.nome",
        "VL_TOTAL":     "valor_total",
        "DT_EMISSAO":   "data_criacao"
      }
    },
    {
      "operation": "default",
      "spec": {
        "origem": "erp-protheus"
      }
    }
  ]
```

#### Envio para Elasticsearch

```
Processor: PutElasticsearchRecord
  Client Service: ElasticsearchClientService
  Index: pedidos-${now():format('yyyy-MM')}       ← index dinâmico por mês
  Type: _doc
  ID_Record_Path: /id_pedido                       ← ID determinístico (evita duplicatas)
```

---

### Expression Language — Referência Rápida

```
// Data e hora
${now():format('yyyy-MM-dd')}
${now():toNumber():minus(86400000):format('yyyy-MM-dd')}   // ontem

// Manipulação de string
${filename:substringBefore('.')}
${campo:toUpper()}
${campo:replaceAll('[^a-zA-Z0-9]', '_')}

// Condicional
${status:equals('ATIVO'):ifElse('true', 'false')}

// Atributo do FlowFile
${uuid}
${fileSize}
${entryDate:toNumber():format('yyyy-MM-dd HH:mm:ss')}

// Uso em nome dinâmico de arquivo
${now():format('yyyyMMdd')}_${origem}_${uuid}.json
```

---

### Parameter Context — Ambiente vs Produção

Nunca hardcode endpoints, senhas ou configurações de ambiente no fluxo.

```
Parameter Context: PROD
  ├── es.host         = https://elasticsearch-prod:9200
  ├── es.user         = nifi_writer
  ├── db.url          = jdbc:sqlserver://sql-prod:1433;database=ERP
  └── db.password     = (Sensitive) ••••••••

Parameter Context: DEV
  ├── es.host         = http://localhost:9200
  ├── es.user         = elastic
  ├── db.url          = jdbc:sqlserver://localhost:1433;database=ERP_DEV
  └── db.password     = (Sensitive) ••••••••

Uso no processor: #{es.host}
```

---

### Tratamento de Erros — O Que Não Pode Faltar

```
Todo processor deve ter as seguintes connections tratadas:

  success  ──▶  próximo processor
  failure  ──▶  PutFile (/nifi/errors/${now():format('yyyyMMdd')}/)
             ──▶ LogMessage (log nível ERROR com atributos do FlowFile)
             ──▶ [opcional] InvokeHTTP para alerta Slack/Teams

Outros estados importantes:
  retry    ──▶  loop de volta ao processor com backoff
  original ──▶  preservar FlowFile original (SplitJson, etc.)

⚠️  FlowFile na fila "failure" sem destino = backpressure acumulando = sistema trava
```

---

### NiFi Registry — Versionamento de Fluxos

```bash
# Workflow obrigatório antes de qualquer mudança em produção:
1. Process Group → botão direito → "Start Version Control"
2. Selecionar Registry bucket
3. Dar versão semântica: 1.0.0 → 1.1.0 (feature) → 2.0.0 (breaking)
4. Testar em DEV com o Process Group versionado
5. Promover versão para PROD via "Change Version"

# NiFi Registry também funciona como backup —
# configurar com Git backend para rastreabilidade total
```

---

### Referências
- [Apache NiFi Docs](https://nifi.apache.org/documentation.html)
- [NiFi Expression Language Guide](https://nifi.apache.org/docs/nifi-docs/html/expression-language-guide.html)
- [NiFi Best Practices — Cloudera](https://docs.cloudera.com/cfm/2.1.6/nifi-best-practices/)

---

## SQL Server

### Visão Geral

SQL Server é o banco relacional mais presente nos ambientes onde trabalho, frequentemente como fonte de dados para pipelines NiFi e como backend de sistemas ERP. O trabalho vai além de queries: tuning de índices, gestão de planos de execução e alta disponibilidade.

**Versões:** SQL Server 2016, 2019, 2022  
**Ferramentas:** SSMS, Azure Data Studio, sqlcmd

---

### Modelagem e DDL com Boas Práticas

```sql
-- Sempre usar schemas para organizar objetos
CREATE SCHEMA financeiro;
GO

-- Tabela com todas as boas práticas de DDL
CREATE TABLE financeiro.pedidos (
    id_pedido     INT             NOT NULL IDENTITY(1,1),
    cd_cliente    VARCHAR(20)     NOT NULL,
    ds_pedido     NVARCHAR(500)   NULL,         -- N para Unicode
    vl_total      DECIMAL(18, 2)  NOT NULL DEFAULT 0,
    st_pedido     CHAR(1)         NOT NULL DEFAULT 'A'
                  CHECK (st_pedido IN ('A', 'C', 'F')),
    dt_criacao    DATETIME2(0)    NOT NULL DEFAULT SYSDATETIME(),
    dt_alteracao  DATETIME2(0)    NULL,
    CONSTRAINT PK_pedidos PRIMARY KEY CLUSTERED (id_pedido),
    CONSTRAINT FK_pedidos_cliente
        FOREIGN KEY (cd_cliente) REFERENCES cadastro.clientes(cd_cliente)
);
GO

-- Index para os filtros mais frequentes
CREATE NONCLUSTERED INDEX IX_pedidos_cliente_status
ON financeiro.pedidos (cd_cliente, st_pedido)
INCLUDE (vl_total, dt_criacao);     -- INCLUDE evita key lookup
GO
```

---

### T-SQL Avançado

```sql
-- CTE recursiva para hierarquia
WITH hierarquia AS (
    -- Âncora: nós raiz
    SELECT id, nome, id_pai, 0 AS nivel, CAST(nome AS VARCHAR(MAX)) AS caminho
    FROM organizacao.departamentos
    WHERE id_pai IS NULL

    UNION ALL

    -- Recursão
    SELECT d.id, d.nome, d.id_pai, h.nivel + 1, h.caminho + ' > ' + d.nome
    FROM organizacao.departamentos d
    INNER JOIN hierarquia h ON d.id_pai = h.id
)
SELECT * FROM hierarquia ORDER BY caminho;
GO

-- Window functions — análise sem perder linhas
SELECT
    cd_cliente,
    dt_criacao,
    vl_total,
    SUM(vl_total)   OVER (PARTITION BY cd_cliente ORDER BY dt_criacao
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS vl_acumulado,
    LAG(vl_total, 1, 0) OVER (PARTITION BY cd_cliente ORDER BY dt_criacao) AS vl_anterior,
    ROW_NUMBER()    OVER (PARTITION BY cd_cliente ORDER BY dt_criacao DESC) AS rn
FROM financeiro.pedidos;
GO

-- MERGE — upsert performático
MERGE INTO financeiro.pedidos AS destino
USING staging.pedidos_novos AS origem
    ON destino.cd_pedido_externo = origem.cd_pedido_externo
WHEN MATCHED AND destino.dt_alteracao < origem.dt_alteracao THEN
    UPDATE SET vl_total = origem.vl_total, st_pedido = origem.st_pedido,
               dt_alteracao = origem.dt_alteracao
WHEN NOT MATCHED BY TARGET THEN
    INSERT (cd_cliente, cd_pedido_externo, vl_total, st_pedido)
    VALUES (origem.cd_cliente, origem.cd_pedido_externo, origem.vl_total, origem.st_pedido)
WHEN NOT MATCHED BY SOURCE AND destino.st_pedido = 'A' THEN
    UPDATE SET st_pedido = 'C';     -- cancela o que sumiu da origem
GO
```

---

### Stored Procedures — Padrão de Qualidade

```sql
CREATE OR ALTER PROCEDURE financeiro.usp_ProcessarPedido
    @id_pedido   INT,
    @novo_status CHAR(1),
    @usuario     VARCHAR(100),
    @debug       BIT = 0
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;      -- rollback automático em erro dentro de transação

    -- Validação de entrada
    IF @novo_status NOT IN ('A', 'C', 'F')
    BEGIN
        RAISERROR('Status inválido: %s', 16, 1, @novo_status);
        RETURN;
    END

    BEGIN TRY
        BEGIN TRANSACTION;

            UPDATE financeiro.pedidos
               SET st_pedido    = @novo_status,
                   dt_alteracao = SYSDATETIME()
             WHERE id_pedido = @id_pedido;

            IF @@ROWCOUNT = 0
                RAISERROR('Pedido %d não encontrado.', 16, 1, @id_pedido);

            INSERT INTO auditoria.historico_pedidos
                (id_pedido, st_anterior, st_novo, usuario, dt_evento)
            SELECT @id_pedido, st_pedido, @novo_status, @usuario, SYSDATETIME()
            FROM   financeiro.pedidos WHERE id_pedido = @id_pedido;

        COMMIT TRANSACTION;

    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;

        INSERT INTO logs.erros (procedure_name, error_message, error_line, dt_erro)
        VALUES (OBJECT_NAME(@@PROCID), ERROR_MESSAGE(), ERROR_LINE(), SYSDATETIME());

        THROW;  -- re-raise para o caller
    END CATCH
END;
GO
```

---

### Diagnóstico de Performance

```sql
-- Queries mais lentas no cache de planos
SELECT TOP 20
    qs.total_elapsed_time / qs.execution_count / 1000.0 AS avg_ms,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count         AS avg_logical_reads,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
              ((CASE qs.statement_end_offset
                    WHEN -1 THEN DATALENGTH(qt.text)
                    ELSE qs.statement_end_offset END
               - qs.statement_start_offset)/2)+1)       AS query_text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle)         qt
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle)      qp
ORDER BY avg_ms DESC;

-- Índices faltando (sugeridos pelo otimizador)
SELECT TOP 10
    migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) AS improvement_measure,
    mid.statement                                       AS tabela,
    mid.equality_columns, mid.inequality_columns, mid.included_columns,
    'CREATE INDEX IX_ ON ' + mid.statement
    + ' (' + ISNULL(mid.equality_columns, '')
    + CASE WHEN mid.inequality_columns IS NOT NULL THEN ',' + mid.inequality_columns ELSE '' END
    + ')' + CASE WHEN mid.included_columns IS NOT NULL THEN ' INCLUDE (' + mid.included_columns + ')' ELSE '' END AS create_index_sql
FROM sys.dm_db_missing_index_groups    mig
JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
JOIN sys.dm_db_missing_index_details   mid  ON mig.index_handle   = mid.index_handle
ORDER BY improvement_measure DESC;

-- Sessões bloqueadas
SELECT
    r.session_id, r.blocking_session_id,
    r.wait_type, r.wait_time / 1000.0 AS wait_segundos,
    SUBSTRING(t.text, (r.statement_start_offset/2)+1, 200) AS query
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.blocking_session_id <> 0;
```

---

### Always On Availability Groups

```sql
-- Status do grupo de disponibilidade
SELECT
    ag.name             AS grupo,
    ar.replica_server_name,
    ar.availability_mode_desc,
    ar.failover_mode_desc,
    ars.role_desc,
    ars.synchronization_health_desc,
    ars.connected_state_desc
FROM sys.availability_groups          ag
JOIN sys.availability_replicas        ar  ON ar.group_id  = ag.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ars.replica_id = ar.replica_id;

-- Latência de sincronização
SELECT
    database_name,
    synchronization_state_desc,
    log_send_queue_size,
    redo_queue_size,
    last_commit_time
FROM sys.dm_hadr_database_replica_states
JOIN sys.availability_replicas ON replica_id = replica_id;
```

---

### Referências
- [Microsoft Learn — SQL Server](https://learn.microsoft.com/pt-br/sql/sql-server/)
- [Brent Ozar — First Responder Kit](https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit)
- [Ola Hallengren — Maintenance Solution](https://ola.hallengren.com/)

---

## Oracle Database

### Visão Geral

Oracle aparece principalmente em sistemas legados de ERP e integrações com TOTVS Protheus. O trabalho envolve PL/SQL, gestão de schemas complexos com centenas de tabelas, e integração via Database Links e Data Pump.

**Versões:** Oracle 12c, 19c (LTS)  
**Ferramentas:** SQL*Plus, SQL Developer, TOAD

---

### PL/SQL — Padrões de Desenvolvimento

```sql
-- Package: a forma correta de organizar PL/SQL
CREATE OR REPLACE PACKAGE financeiro_pkg AS

    -- Tipos públicos
    TYPE t_pedido IS RECORD (
        id_pedido  pedidos.id_pedido%TYPE,
        cd_cliente pedidos.cd_cliente%TYPE,
        vl_total   pedidos.vl_total%TYPE
    );
    TYPE t_pedidos IS TABLE OF t_pedido;

    -- Cabeçalhos públicos
    PROCEDURE processar_pedido(p_id IN NUMBER, p_status IN VARCHAR2);
    FUNCTION  buscar_pedido(p_id IN NUMBER) RETURN t_pedido;
    FUNCTION  listar_pendentes RETURN t_pedidos PIPELINED;

END financeiro_pkg;
/

CREATE OR REPLACE PACKAGE BODY financeiro_pkg AS

    -- Constante privada
    c_modulo CONSTANT VARCHAR2(50) := 'FINANCEIRO_PKG';

    PROCEDURE processar_pedido(p_id IN NUMBER, p_status IN VARCHAR2) IS
        v_pedido pedidos%ROWTYPE;
    BEGIN
        SELECT * INTO v_pedido FROM pedidos WHERE id_pedido = p_id FOR UPDATE NOWAIT;

        IF v_pedido.st_pedido = 'F' THEN
            RAISE_APPLICATION_ERROR(-20001, 'Pedido já finalizado: ' || p_id);
        END IF;

        UPDATE pedidos SET st_pedido = p_status, dt_alteracao = SYSDATE
         WHERE id_pedido = p_id;

        INSERT INTO auditoria_pedidos VALUES (p_id, v_pedido.st_pedido, p_status, SYSDATE, SYS_CONTEXT('USERENV','SESSION_USER'));

        COMMIT;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RAISE_APPLICATION_ERROR(-20002, 'Pedido não encontrado: ' || p_id);
        WHEN OTHERS THEN
            ROLLBACK;
            logs_pkg.registrar(c_modulo, SQLERRM, DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
            RAISE;
    END processar_pedido;

    -- Pipelined function — retorna linhas uma a uma sem esperar tudo na memória
    FUNCTION listar_pendentes RETURN t_pedidos PIPELINED IS
        v_row t_pedido;
    BEGIN
        FOR r IN (SELECT id_pedido, cd_cliente, vl_total FROM pedidos WHERE st_pedido = 'P') LOOP
            v_row.id_pedido  := r.id_pedido;
            v_row.cd_cliente := r.cd_cliente;
            v_row.vl_total   := r.vl_total;
            PIPE ROW(v_row);
        END LOOP;
    END listar_pendentes;

END financeiro_pkg;
/
```

---

### Bulk Operations — Performance com Volume

```sql
-- FORALL + BULK COLLECT: a diferença é brutal em volume
DECLARE
    TYPE t_ids   IS TABLE OF pedidos.id_pedido%TYPE;
    TYPE t_status IS TABLE OF pedidos.st_pedido%TYPE;

    v_ids    t_ids;
    v_status t_status;
    v_errors PLS_INTEGER;
BEGIN
    -- Busca em lote (evita N roundtrips ao banco)
    SELECT id_pedido, st_pedido
    BULK COLLECT INTO v_ids, v_status
    FROM pedidos
    WHERE st_pedido = 'P' AND dt_criacao < SYSDATE - 30;

    -- Update em lote com SAVE EXCEPTIONS (não para no primeiro erro)
    FORALL i IN 1..v_ids.COUNT SAVE EXCEPTIONS
        UPDATE pedidos SET st_pedido = 'C' WHERE id_pedido = v_ids(i);

    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        v_errors := SQL%BULK_EXCEPTIONS.COUNT;
        FOR i IN 1..v_errors LOOP
            DBMS_OUTPUT.PUT_LINE('Erro no índice ' || SQL%BULK_EXCEPTIONS(i).ERROR_INDEX
                || ': ' || SQLERRM(-SQL%BULK_EXCEPTIONS(i).ERROR_CODE));
        END LOOP;
        COMMIT; -- comita os sucessos
END;
/
```

---

### Diagnóstico e Tuning

```sql
-- ASH — o que o banco está fazendo agora
SELECT ash.sql_id, ash.event, COUNT(*) AS samples, sq.sql_text
FROM v$active_session_history ash
LEFT JOIN v$sql sq ON sq.sql_id = ash.sql_id
WHERE ash.sample_time > SYSDATE - 1/24     -- última hora
GROUP BY ash.sql_id, ash.event, sq.sql_text
ORDER BY samples DESC
FETCH FIRST 20 ROWS ONLY;

-- Explain plan
EXPLAIN PLAN FOR
SELECT * FROM pedidos WHERE cd_cliente = 'CLI001' AND st_pedido = 'A';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(FORMAT=>'ALL'));

-- Objetos inválidos e recompilação em massa
BEGIN
    FOR obj IN (SELECT owner, object_name, object_type FROM dba_objects WHERE status = 'INVALID') LOOP
        BEGIN
            EXECUTE IMMEDIATE 'ALTER ' || obj.object_type || ' ' || obj.owner || '.' || obj.object_name || ' COMPILE';
        EXCEPTION WHEN OTHERS THEN NULL;
        END;
    END LOOP;
END;
/

-- Espaço por tablespace
SELECT t.tablespace_name,
       ROUND(t.total_mb, 1) AS total_mb,
       ROUND(f.free_mb,  1) AS free_mb,
       ROUND((1 - f.free_mb/t.total_mb)*100, 1) AS pct_used
FROM (SELECT tablespace_name, SUM(bytes)/1024/1024 AS total_mb FROM dba_data_files GROUP BY tablespace_name) t
JOIN (SELECT tablespace_name, SUM(bytes)/1024/1024 AS free_mb  FROM dba_free_space    GROUP BY tablespace_name) f
  ON t.tablespace_name = f.tablespace_name
ORDER BY pct_used DESC;
```

---

### Data Pump — Export e Import

```bash
# Export de schema completo
expdp system/senha@SID \
  SCHEMAS=FINANCEIRO \
  DIRECTORY=DATA_PUMP_DIR \
  DUMPFILE=financeiro_%U.dmp \    # %U = múltiplos arquivos paralelos
  LOGFILE=expdp_financeiro.log \
  PARALLEL=4 \
  COMPRESSION=ALL

# Import em schema diferente (migração/clone)
impdp system/senha@SID_DESTINO \
  DIRECTORY=DATA_PUMP_DIR \
  DUMPFILE=financeiro_%U.dmp \
  LOGFILE=impdp_financeiro.log \
  REMAP_SCHEMA=FINANCEIRO:FINANCEIRO_HML \
  REMAP_TABLESPACE=FINANCEIRO_TBS:USERS \
  TABLE_EXISTS_ACTION=REPLACE \
  PARALLEL=4

# Export de tabela específica com filtro
expdp system/senha@SID \
  TABLES=FINANCEIRO.PEDIDOS \
  QUERY=FINANCEIRO.PEDIDOS:'"WHERE dt_criacao >= SYSDATE - 30"' \
  DIRECTORY=DATA_PUMP_DIR \
  DUMPFILE=pedidos_30dias.dmp
```

---

### Referências
- [Oracle Docs 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/)
- [Ask TOM](https://asktom.oracle.com/)
- [Oracle Base](https://oracle-base.com/)

---

## TOTVS Protheus

### Visão Geral

Trabalho com customização e desenvolvimento de rotinas no Protheus há mais de 4 anos. O foco está em pontos de entrada, MVC, REST e integração com sistemas externos. A arquitetura ERP exige conhecimento profundo do modelo de dados e das convenções da plataforma.

**Módulos trabalhados:** SIGAFIN, SIGACOM, SIGAFAT, SIGAEST, SIGAMDI  
**Linguagem:** ADVPL / TLPP  
**Ferramentas:** IDE TDS (VSCode), SmartClient

---

### ADVPL / TLPP — Fundamentos Sólidos

```advpl
// Estrutura de uma função ADVPL profissional
User Function ProcPedido(cNrPed, cFilial)

    Local cModulo  := "SIGAFAT"
    Local cFunc    := "ProcPedido"
    Local oErro    := Nil
    Local lRet     := .T.
    Local cErro    := ""

    Private lMsErroAuto := .F.
    Private aAutoErro   := {}

    // Validação de entrada
    If Empty(cNrPed)
        MsgStop("Número do pedido não informado.", "Erro")
        Return .F.
    EndIf

    // Proteção com BEGIN SEQUENCE
    BEGIN SEQUENCE

        // Abertura da área de trabalho
        DbSelectArea("SC5")
        SC5->(DbSetOrder(1))    // Índice 1: C5_FILIAL + C5_NUM

        If !SC5->(MsSeek(FWxFilial("SC5") + cNrPed))
            MsgStop("Pedido " + cNrPed + " não encontrado.", "Erro")
            Break
        EndIf

        If SC5->C5_LIBEROK == "2"
            MsgStop("Pedido já processado.", "Informação")
            Break
        EndIf

        // Processamento com proteção de gravação
        RecLock("SC5", .F.)     // .F. = Lock sem criar novo registro
            SC5->C5_LIBEROK := "2"
            SC5->C5_USERLKG := RetCodUsr()
        MsUnLock()

        lRet := .T.

    END SEQUENCE

    If lMsErroAuto
        lRet := .F.
        cErro := MsErroAuto()
        MsgStop(cErro, "Erro no processamento")
    EndIf

Return lRet
```

---

### MVC — Desenvolvimento de Telas

```advpl
// Model
Static Function ModelDef()

    Local oModel := MPFormModel():New("ZMVPEDIDO",, {|oModel, nOpc| Valid(oModel, nOpc)})

    oModel:SetDescription("Manutenção de Pedidos Customizados")

    // Entidade principal (cabeçalho)
    oModel:AddFields("CABEC", /*cOwner*/, "FLDCABEC")
    oModel:SetPrimaryKey({"ZP_FILIAL", "ZP_CODIGO"})

    // Grid de itens
    oModel:AddGrid("ITENS", "CABEC", "FLDITENS")
    oModel:SetRelation("ITENS", {{"ZI_FILIAL","ZP_FILIAL"},{"ZI_PEDIDO","ZP_CODIGO"}}, ZI->(IndexKey(1)))

    oModel:GetModel("ITENS"):SetOptional(.T.)    // Grid não obrigatório

Return oModel

// View
Static Function ViewDef()

    Local oView
    Local oModel := FWLoadModel("ZMVPEDIDO")

    oView := FWFormView():New()
    oView:SetDescription("Pedidos Customizados")
    oView:AddField("VIEW_CABEC", oModel:GetModel("CABEC"), "PANELCABEC")
    oView:AddGrid("VIEW_ITENS",  oModel:GetModel("ITENS"),  "PANELITENS")

    oView:CreateHorizontalBox("PANELCABEC", 40)
    oView:CreateHorizontalBox("PANELITENS", 60)

Return oView
```

---

### REST com ADVPL (TLPP)

```tlpp
// Endpoint: POST /api/v1/pedidos
#Include "tlpp-core.th"
#Include "tlpp-rest.th"

@Get("/api/v1/pedidos/:id")
Function GetPedido() As Object

    Local cId    := oRest:getPathParam("id")
    Local oResp  := JsonObject():New()
    Local oErro  := JsonObject():New()

    DbSelectArea("SC5")
    SC5->(DbSetOrder(1))

    If SC5->(MsSeek(FWxFilial("SC5") + cId))
        oResp["status"]    := "ok"
        oResp["nr_pedido"] := cId
        oResp["cliente"]   := AllTrim(SC5->C5_CLIENT)
        oResp["valor"]     := SC5->C5_VALBRUT
        oResp["situacao"]  := AllTrim(SC5->C5_LIBEROK)

        oRest:setResponse(oResp:ToJson())
        oRest:setStatus(200)
    Else
        oErro["status"]    := "erro"
        oErro["mensagem"]  := "Pedido " + cId + " não encontrado"
        oRest:setResponse(oErro:ToJson())
        oRest:setStatus(404)
    EndIf

Return Nil
```

---

### Pontos de Entrada — Customização Sem Modificar Fonte

```advpl
// Ponto de entrada na gravação do pedido de venda (SC5)
User Function SC5OK()

    Local lRet  := .T.
    Local nOpc  := SC5->(nOpc)    // 3 = inclusão, 4 = alteração

    // Validação custom apenas na inclusão
    If nOpc == 3
        If SC5->C5_VALBRUT > 50000 .And. Empty(SC5->C5_AUTORI)
            MsgStop("Pedidos acima de R$50.000 exigem autorização.", "Pendência")
            lRet := .F.
        EndIf
    EndIf

Return lRet

// Ponto de entrada no relatório
User Function MT680PRI()
    // Adiciona coluna customizada no relatório de pedidos
    @ oRel:nLinha, 150 Say "Coluna Extra" Font oRel:oFont
Return
```

---

### Boas Práticas ADVPL

| Prática | Motivo |
|---|---|
| Sempre usar `BEGIN SEQUENCE / END SEQUENCE` | Captura erros sem deixar locks abertos |
| `Private lMsErroAuto := .F.` no início de toda rotina automática | Captura erros do MsExecAuto |
| `AllTrim()` em todo campo de string | Campos ADVPL têm espaços à direita por padrão |
| Nunca usar `USE` diretamente — usar `DbSelectArea` | `USE` fecha outras áreas de trabalho abertas |
| Testar ponto de entrada no SmartClient antes do TDS | Ambiente de homologação antes de subir RPO |
| Versionar fontes no Git com TDS + extensão de versionamento | Rastreabilidade de alterações |

---

### Referências
- [TOTVS Developers](https://developers.totvs.com/)
- [TDN ADVPL](https://tdn.totvs.com/display/public/framework/ADVPL)
- [Fórum TOTVS Community](https://community.totvs.com/)

---

## OpenEdge Progress

### Visão Geral

OpenEdge (ABL) está presente em sistemas ERP legados e em integrações com plataformas que ainda usam bancos Progress. O trabalho envolve manutenção e desenvolvimento de procedures ABL, criação de serviços REST via PASOE e integração com sistemas externos.

**Versão:** OpenEdge 12.x  
**Ferramentas:** Progress Developer Studio (Eclipse), PDSOE, proenv

---

### ABL — Fundamentos e Padrões

```abl
/*------------------------------------------------------------------------
  Procedure: processar_pedido.p
  Descrição: Processa pedido e atualiza status
------------------------------------------------------------------------*/

DEFINE INPUT  PARAMETER p-nr-pedido AS INTEGER   NO-UNDO.
DEFINE INPUT  PARAMETER p-status    AS CHARACTER NO-UNDO.
DEFINE OUTPUT PARAMETER p-sucesso   AS LOGICAL   NO-UNDO INITIAL FALSE.
DEFINE OUTPUT PARAMETER p-mensagem  AS CHARACTER NO-UNDO.

DEFINE VARIABLE v-usuario AS CHARACTER NO-UNDO.

/* Validação */
IF p-nr-pedido <= 0 THEN DO:
    p-mensagem = "Número de pedido inválido.".
    RETURN.
END.

IF NOT CAN-DO("A,C,F", p-status) THEN DO:
    p-mensagem = SUBSTITUTE("Status inválido: &1", p-status).
    RETURN.
END.

/* Processamento em transação */
DO TRANSACTION ON ERROR UNDO, LEAVE:

    FIND pedidos WHERE pedidos.nr-pedido = p-nr-pedido
         EXCLUSIVE-LOCK NO-ERROR.

    IF NOT AVAILABLE pedidos THEN DO:
        p-mensagem = SUBSTITUTE("Pedido &1 não encontrado.", p-nr-pedido).
        UNDO, LEAVE.
    END.

    IF pedidos.st-pedido = "F" THEN DO:
        p-mensagem = "Pedido já finalizado.".
        UNDO, LEAVE.
    END.

    ASSIGN
        pedidos.st-anterior  = pedidos.st-pedido
        pedidos.st-pedido    = p-status
        pedidos.dt-alteracao = TODAY
        pedidos.hr-alteracao = TIME.

    CREATE auditoria-pedidos.
    ASSIGN
        auditoria-pedidos.nr-pedido  = p-nr-pedido
        auditoria-pedidos.st-novo    = p-status
        auditoria-pedidos.dt-evento  = TODAY
        auditoria-pedidos.nm-usuario = USERID("USERFILE").

    p-sucesso  = TRUE.
    p-mensagem = "Pedido processado com sucesso.".

END. /* DO TRANSACTION */
```

---

### ProDataSet — Transferência de Dados Estruturada

```abl
/* Definição de ProDataSet com relacionamento */
DEFINE TEMP-TABLE tt-pedido NO-UNDO
    FIELD nr-pedido  AS INTEGER
    FIELD cd-cliente AS CHARACTER
    FIELD vl-total   AS DECIMAL
    FIELD st-pedido  AS CHARACTER.

DEFINE TEMP-TABLE tt-item NO-UNDO
    FIELD nr-pedido  AS INTEGER
    FIELD nr-item    AS INTEGER
    FIELD cd-produto AS CHARACTER
    FIELD qt-item    AS DECIMAL
    FIELD vl-item    AS DECIMAL.

DEFINE DATASET ds-pedido
    FOR tt-pedido, tt-item
    DATA-RELATION rel-itens FOR tt-pedido, tt-item
        RELATION-FIELDS (nr-pedido, nr-pedido).

/* Leitura com cursor eficiente */
DEFINE QUERY q-pedidos FOR pedidos SCROLLING.

OPEN QUERY q-pedidos
    FOR EACH pedidos NO-LOCK
    WHERE pedidos.st-pedido = "P"
      AND pedidos.dt-criacao >= TODAY - 30
    BY pedidos.dt-criacao DESCENDING.

GET FIRST q-pedidos.
DO WHILE AVAILABLE pedidos:

    CREATE tt-pedido.
    ASSIGN
        tt-pedido.nr-pedido  = pedidos.nr-pedido
        tt-pedido.cd-cliente = pedidos.cd-cliente
        tt-pedido.vl-total   = pedidos.vl-total
        tt-pedido.st-pedido  = pedidos.st-pedido.

    FOR EACH itens-pedido NO-LOCK WHERE itens-pedido.nr-pedido = pedidos.nr-pedido:
        CREATE tt-item.
        ASSIGN
            tt-item.nr-pedido  = pedidos.nr-pedido
            tt-item.nr-item    = itens-pedido.nr-item
            tt-item.cd-produto = itens-pedido.cd-produto
            tt-item.qt-item    = itens-pedido.qt-item.
    END.

    GET NEXT q-pedidos.
END.
CLOSE QUERY q-pedidos.
```

---

### PASOE — Serviço REST

```abl
/*------------------------------------------------------------------------
  Class: PedidosResource.cls
  URI:   /web/psa/api/v1/pedidos
------------------------------------------------------------------------*/

CLASS api.PedidosResource INHERITS OpenEdge.Web.WebHandler:

    METHOD OVERRIDE PROTECTED INTEGER HandleGet(
        INPUT p-request  AS OpenEdge.Web.IWebRequest):

        DEFINE VARIABLE oResponse AS OpenEdge.Web.SendExceptionError NO-UNDO.
        DEFINE VARIABLE oWriter   AS OpenEdge.Web.DataObject.Writer.BodyWriter NO-UNDO.
        DEFINE VARIABLE oJson     AS Progress.Json.ObjectModel.JsonObject NO-UNDO.
        DEFINE VARIABLE oArray    AS Progress.Json.ObjectModel.JsonArray  NO-UNDO.
        DEFINE VARIABLE oItem     AS Progress.Json.ObjectModel.JsonObject NO-UNDO.

        oJson  = NEW Progress.Json.ObjectModel.JsonObject().
        oArray = NEW Progress.Json.ObjectModel.JsonArray().

        FOR EACH pedidos NO-LOCK WHERE pedidos.st-pedido = "P":
            oItem = NEW Progress.Json.ObjectModel.JsonObject().
            oItem:Add("nr_pedido",  pedidos.nr-pedido).
            oItem:Add("cd_cliente", pedidos.cd-cliente).
            oItem:Add("vl_total",   pedidos.vl-total).
            oArray:Add(oItem).
        END.

        oJson:Add("status",  "ok").
        oJson:Add("total",   oArray:Length).
        oJson:Add("pedidos", oArray).

        THIS-OBJECT:StatusCode  = 200.
        THIS-OBJECT:ContentType = "application/json".
        THIS-OBJECT:Entity      = oJson.

        RETURN 0.

    END METHOD.

END CLASS.
```

---

### Operações Administrativas

```bash
# Ambiente proenv (sempre trabalhar dentro do proenv)
proenv

# Conectar ao banco
pro -db /caminho/banco/sports2000 -H localhost -S 3000

# Backup online
probkup online /caminho/banco/meubanco /backup/meubanco.bak -com

# Restore
prorest /caminho/banco/meubanco /backup/meubanco.bak

# Compilar procedure
procomp minha-procedure.p -r minha-procedure.r

# Gerar RPO (arquivo compilado)
prolib rpo/meuprojeto.pl -create
prolib rpo/meuprojeto.pl -add minha-procedure.r

# Verificar integridade do banco
procheck /caminho/banco/meubanco
```

---

### Referências
- [Progress OpenEdge Docs 12.x](https://docs.progress.com/bundle/openedge-abl-reference-122/page/Contents.html)
- [Progress Community](https://community.progress.com/)
- [Progress KnowledgeBase](https://knowledgebase.progress.com/)

---
