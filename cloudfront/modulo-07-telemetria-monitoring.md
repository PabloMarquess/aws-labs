# Módulo 07 — Telemetria, Monitoring e Logs

> **Nível:** 300-400 (Advanced/Expert)
> **Tempo Total Estimado:** 6-8 horas de labs
> **Objetivo do Módulo:** Implementar observabilidade completa no CloudFront — desde access logs e métricas até real-time monitoring, dashboards e runbooks de troubleshooting.

---

## Mapa de Observabilidade

```
                    ┌──────────────────────────────────────────────────┐
                    │              OBSERVABILIDADE CLOUDFRONT           │
                    └──────────────────────────────────────────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
        ┌─────┴─────┐           ┌────────┴────────┐        ┌───────┴───────┐
        │   LOGS     │           │    MÉTRICAS      │        │   REPORTS     │
        └─────┬─────┘           └────────┬────────┘        └───────┬───────┘
              │                          │                          │
      ┌───────┼───────┐          ┌───────┼───────┐          ┌──────┼──────┐
      │               │          │               │          │             │
  Standard       Real-Time   CloudWatch      Additional   Console     Athena
  Logs (S3)     Logs(Kinesis) Metrics        Metrics      Reports    Queries
      │               │          │               │          │             │
   Athena          Lambda     Dashboards     Per-edge   Cache Stats  Custom
   Queries        Consumer    + Alarms       Per-code   Popular Obj  Analysis
                      │                                 Top Referrer
                  OpenSearch                            Usage/Viewers
                  CloudWatch
```

---

## Desafio 39: Standard Logs (Access Logs) para S3

> **Level:** 300 | **Tempo:** 60 min | **Custo:** S3 storage (~$0.023/GB)

### Objetivo
Habilitar standard logging, entender cada campo dos logs, e analisar com Amazon Athena usando queries prontas para produção.

### Contexto Real
Standard logs são a base de toda análise de tráfego. Sem eles, você está "voando cego". Todo time de operações precisa de logs para debugging, análise de performance, e auditoria de segurança.

### Passo a Passo

#### 1. Criar Bucket para Logs

```bash
# Criar bucket para logs (mesma região, nome descritivo)
aws s3 mb s3://meu-cloudfront-logs-001 --region us-east-1

# CloudFront precisa de permissão para escrever logs
# A forma moderna é via bucket policy (não ACL)
aws s3api put-bucket-policy --bucket meu-cloudfront-logs-001 --policy '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontLogs",
            "Effect": "Allow",
            "Principal": {
                "Service": "delivery.logs.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::meu-cloudfront-logs-001/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control",
                    "AWS:SourceAccount": "123456789012"
                }
            }
        }
    ]
}'

# Lifecycle para deletar logs antigos (economia)
aws s3api put-bucket-lifecycle-configuration --bucket meu-cloudfront-logs-001 --lifecycle-configuration '{
    "Rules": [
        {
            "ID": "DeleteOldLogs",
            "Status": "Enabled",
            "Filter": {"Prefix": ""},
            "Transitions": [
                {"Days": 30, "StorageClass": "STANDARD_IA"},
                {"Days": 90, "StorageClass": "GLACIER"}
            ],
            "Expiration": {"Days": 365}
        }
    ]
}'
```

#### 2. Habilitar Logging na Distribution

```hcl
# Terraform
resource "aws_cloudfront_distribution" "main" {
  # ... origins, behaviors ...

  logging_config {
    include_cookies = false  # true para debug de cookies
    bucket          = "${aws_s3_bucket.logs.bucket_domain_name}"
    prefix          = "cloudfront-logs/"
  }
}
```

```bash
# CLI — atualizar distribution existente
# Obter config, adicionar Logging, e atualizar
```

#### 3. Formato dos Logs — Todos os Campos

Cada linha de log tem estes campos (tab-separated):

| # | Campo | Exemplo | Descrição |
|---|-------|---------|-----------|
| 1 | `date` | `2025-01-15` | Data UTC |
| 2 | `time` | `14:23:05` | Hora UTC |
| 3 | `x-edge-location` | `GRU50-C1` | Edge location que processou |
| 4 | `sc-bytes` | `23456` | Bytes enviados ao viewer |
| 5 | `c-ip` | `203.0.113.50` | IP do viewer |
| 6 | `cs-method` | `GET` | HTTP method |
| 7 | `cs(Host)` | `d1234.cloudfront.net` | Host header |
| 8 | `cs-uri-stem` | `/images/logo.png` | URI path |
| 9 | `sc-status` | `200` | HTTP status code |
| 10 | `cs(Referer)` | `https://meusite.com/` | Referer header |
| 11 | `cs(User-Agent)` | `Mozilla/5.0...` | User-Agent |
| 12 | `cs-uri-query` | `size=large` | Query string |
| 13 | `cs(Cookie)` | `-` | Cookie header (se habilitado) |
| 14 | `x-edge-result-type` | `Hit` | Resultado do cache |
| 15 | `x-edge-request-id` | `abc123==` | ID único da request |
| 16 | `x-host-header` | `cdn.meusite.com` | Host header original |
| 17 | `cs-protocol` | `https` | Protocolo |
| 18 | `cs-bytes` | `512` | Bytes recebidos do viewer |
| 19 | `time-taken` | `0.002` | Tempo total (segundos) |
| 20 | `x-forwarded-for` | `203.0.113.50` | IP real (se proxy) |
| 21 | `ssl-protocol` | `TLSv1.3` | Versão TLS |
| 22 | `ssl-cipher` | `TLS_AES_128_GCM_SHA256` | Cipher suite |
| 23 | `x-edge-response-result-type` | `Hit` | Resultado final |
| 24 | `cs-protocol-version` | `HTTP/2.0` | Versão HTTP |
| 25 | `fle-status` | `-` | Status do FLE |
| 26 | `fle-encrypted-fields` | `-` | Campos criptografados |
| 27 | `c-port` | `54321` | Porta do viewer |
| 28 | `time-to-first-byte` | `0.001` | TTFB (segundos) |
| 29 | `x-edge-detailed-result-type` | `Hit` | Resultado detalhado |
| 30 | `sc-content-type` | `image/png` | Content-Type da resposta |
| 31 | `sc-content-len` | `23456` | Content-Length |
| 32 | `sc-range-start` | `-` | Range start (para byte-range) |
| 33 | `sc-range-end` | `-` | Range end |

#### Valores de x-edge-result-type

| Valor | Significado | Implicação |
|-------|-------------|------------|
| **Hit** | Servido do cache do edge | Ideal - menor latência |
| **Miss** | Não estava no cache, buscou na origin | Normal para primeira request |
| **RefreshHit** | Estava no cache mas expirado, revalidou com origin (304) | Bom - menor transferência |
| **LimitExceeded** | Cache cheio, não cacheou | Raro - objeto muito popular |
| **CapacityExceeded** | Edge location sem capacidade | Muito raro |
| **Error** | Erro na origin ou no CloudFront | Investigar! |
| **Redirect** | CloudFront gerou redirect (HTTP→HTTPS) | Normal se ViewerProtocolPolicy = redirect |

#### 4. Analisar com Amazon Athena

```sql
-- Criar database
CREATE DATABASE IF NOT EXISTS cloudfront_logs;

-- Criar tabela com partition projection (automático, sem MSCK REPAIR)
CREATE EXTERNAL TABLE cloudfront_logs.access_logs (
    `date` DATE,
    `time` STRING,
    x_edge_location STRING,
    sc_bytes BIGINT,
    c_ip STRING,
    cs_method STRING,
    cs_host STRING,
    cs_uri_stem STRING,
    sc_status INT,
    cs_referer STRING,
    cs_user_agent STRING,
    cs_uri_query STRING,
    cs_cookie STRING,
    x_edge_result_type STRING,
    x_edge_request_id STRING,
    x_host_header STRING,
    cs_protocol STRING,
    cs_bytes BIGINT,
    time_taken DOUBLE,
    x_forwarded_for STRING,
    ssl_protocol STRING,
    ssl_cipher STRING,
    x_edge_response_result_type STRING,
    cs_protocol_version STRING,
    fle_status STRING,
    fle_encrypted_fields STRING,
    c_port INT,
    time_to_first_byte DOUBLE,
    x_edge_detailed_result_type STRING,
    sc_content_type STRING,
    sc_content_len BIGINT,
    sc_range_start BIGINT,
    sc_range_end BIGINT
)
PARTITIONED BY (
    year STRING,
    month STRING,
    day STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
LOCATION 's3://meu-cloudfront-logs-001/cloudfront-logs/'
TBLPROPERTIES (
    'skip.header.line.count'='2',
    'projection.enabled'='true',
    'projection.year.type'='integer',
    'projection.year.range'='2024,2030',
    'projection.month.type'='integer',
    'projection.month.range'='1,12',
    'projection.month.digits'='2',
    'projection.day.type'='integer',
    'projection.day.range'='1,31',
    'projection.day.digits'='2',
    'storage.location.template'='s3://meu-cloudfront-logs-001/cloudfront-logs/${year}/${month}/${day}/'
);
```

#### 5. Queries Essenciais para Produção

```sql
-- 1. Top 20 URLs mais acessadas (últimas 24h)
SELECT cs_uri_stem, COUNT(*) as requests,
       SUM(sc_bytes) as total_bytes,
       ROUND(SUM(sc_bytes) / 1024.0 / 1024.0, 2) as total_mb
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
GROUP BY cs_uri_stem
ORDER BY requests DESC
LIMIT 20;

-- 2. Cache Hit Ratio por hora
SELECT date, SUBSTR(time, 1, 2) as hour,
       COUNT(*) as total_requests,
       SUM(CASE WHEN x_edge_result_type = 'Hit' THEN 1 ELSE 0 END) as hits,
       SUM(CASE WHEN x_edge_result_type = 'Miss' THEN 1 ELSE 0 END) as misses,
       ROUND(SUM(CASE WHEN x_edge_result_type = 'Hit' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as hit_ratio
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
GROUP BY date, SUBSTR(time, 1, 2)
ORDER BY date, hour;

-- 3. Erros 5xx agrupados por URI
SELECT cs_uri_stem, sc_status, COUNT(*) as error_count,
       MIN(time) as first_seen, MAX(time) as last_seen
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
  AND sc_status >= 500
GROUP BY cs_uri_stem, sc_status
ORDER BY error_count DESC
LIMIT 20;

-- 4. Top 10 IPs com mais requests (potenciais abusadores)
SELECT c_ip, COUNT(*) as requests,
       COUNT(DISTINCT cs_uri_stem) as unique_urls,
       SUM(sc_bytes) as total_bytes,
       ROUND(AVG(time_taken), 4) as avg_latency
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
GROUP BY c_ip
ORDER BY requests DESC
LIMIT 10;

-- 5. Latência média por edge location
SELECT x_edge_location,
       COUNT(*) as requests,
       ROUND(AVG(time_taken) * 1000, 2) as avg_latency_ms,
       ROUND(APPROX_PERCENTILE(time_taken, 0.50) * 1000, 2) as p50_ms,
       ROUND(APPROX_PERCENTILE(time_taken, 0.90) * 1000, 2) as p90_ms,
       ROUND(APPROX_PERCENTILE(time_taken, 0.99) * 1000, 2) as p99_ms
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
GROUP BY x_edge_location
ORDER BY requests DESC;

-- 6. Bandwidth por hora
SELECT SUBSTR(time, 1, 2) as hour,
       ROUND(SUM(sc_bytes) / 1024.0 / 1024.0 / 1024.0, 2) as gb_transferred,
       COUNT(*) as requests
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
GROUP BY SUBSTR(time, 1, 2)
ORDER BY hour;

-- 7. Top User-Agents (separar bots de humanos)
SELECT cs_user_agent, COUNT(*) as requests,
       CASE
           WHEN LOWER(cs_user_agent) LIKE '%bot%' OR LOWER(cs_user_agent) LIKE '%crawl%'
                OR LOWER(cs_user_agent) LIKE '%spider%' THEN 'Bot'
           ELSE 'Human'
       END as type
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
GROUP BY cs_user_agent
ORDER BY requests DESC
LIMIT 20;

-- 8. URLs com mais Cache MISS (candidatas a otimização)
SELECT cs_uri_stem,
       COUNT(*) as total,
       SUM(CASE WHEN x_edge_result_type = 'Miss' THEN 1 ELSE 0 END) as misses,
       ROUND(SUM(CASE WHEN x_edge_result_type = 'Miss' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as miss_rate
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
GROUP BY cs_uri_stem
HAVING COUNT(*) > 100
ORDER BY miss_rate DESC
LIMIT 20;

-- 9. Distribuição de status codes
SELECT sc_status, COUNT(*) as requests,
       ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) as percentage
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
GROUP BY sc_status
ORDER BY requests DESC;

-- 10. Protocolos e versões TLS em uso
SELECT ssl_protocol, cs_protocol_version, COUNT(*) as requests,
       ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) as percentage
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
  AND ssl_protocol != '-'
GROUP BY ssl_protocol, cs_protocol_version
ORDER BY requests DESC;

-- 11. Erros 404 (URLs que não existem — bots, links quebrados)
SELECT cs_uri_stem, COUNT(*) as requests,
       ARRAY_AGG(DISTINCT cs_referer) as referers
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
  AND sc_status = 404
GROUP BY cs_uri_stem
ORDER BY requests DESC
LIMIT 20;

-- 12. Tráfego por Content-Type
SELECT sc_content_type, COUNT(*) as requests,
       ROUND(SUM(sc_bytes) / 1024.0 / 1024.0, 2) as total_mb
FROM cloudfront_logs.access_logs
WHERE year = '2025' AND month = '01' AND day = '15'
GROUP BY sc_content_type
ORDER BY total_mb DESC;
```

### Terraform Completo

```hcl
# Bucket para logs
resource "aws_s3_bucket" "cf_logs" {
  bucket = "${var.project_name}-cloudfront-logs"
}

resource "aws_s3_bucket_policy" "cf_logs" {
  bucket = aws_s3_bucket.cf_logs.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowCloudFrontLogs"
      Effect    = "Allow"
      Principal = { Service = "delivery.logs.amazonaws.com" }
      Action    = "s3:PutObject"
      Resource  = "${aws_s3_bucket.cf_logs.arn}/*"
      Condition = {
        StringEquals = {
          "s3:x-amz-acl"    = "bucket-owner-full-control"
          "AWS:SourceAccount" = data.aws_caller_identity.current.account_id
        }
      }
    }]
  })
}

resource "aws_s3_bucket_lifecycle_configuration" "cf_logs" {
  bucket = aws_s3_bucket.cf_logs.id
  rule {
    id     = "log-lifecycle"
    status = "Enabled"
    transition { days = 30; storage_class = "STANDARD_IA" }
    transition { days = 90; storage_class = "GLACIER" }
    expiration { days = 365 }
  }
}

# Athena
resource "aws_athena_database" "cf" {
  name   = "cloudfront_logs"
  bucket = aws_s3_bucket.athena_results.id
}

resource "aws_s3_bucket" "athena_results" {
  bucket = "${var.project_name}-athena-results"
}

resource "aws_athena_named_query" "cache_hit_ratio" {
  name      = "cache-hit-ratio"
  database  = aws_athena_database.cf.name
  query     = <<-EOT
    SELECT date, COUNT(*) as total,
           SUM(CASE WHEN x_edge_result_type = 'Hit' THEN 1 ELSE 0 END) as hits,
           ROUND(SUM(CASE WHEN x_edge_result_type = 'Hit' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as ratio
    FROM cloudfront_logs.access_logs
    WHERE year = CAST(YEAR(CURRENT_DATE) AS VARCHAR)
      AND month = LPAD(CAST(MONTH(CURRENT_DATE) AS VARCHAR), 2, '0')
      AND day = LPAD(CAST(DAY(CURRENT_DATE) AS VARCHAR), 2, '0')
    GROUP BY date ORDER BY date
  EOT
}
```

### O Que Aprendemos

| Conceito | Descrição |
|----------|-----------|
| **Standard Logs** | Logs de acesso escritos em S3 (~5 min de delay) |
| **33 campos** | Cada request gera uma linha com 33 campos tab-separated |
| **Partition Projection** | Athena descobre partições automaticamente (sem MSCK REPAIR) |
| **x-edge-result-type** | Hit, Miss, RefreshHit, Error — a métrica mais importante |
| **time-taken** | Tempo total da request (proxy para latência do viewer) |
| **time-to-first-byte** | TTFB no edge (proxy para latência percebida) |

### Dica Expert
> **Standard Logs têm delay de 5-60 minutos.** Para debugging em tempo real, use Real-Time Logs (Desafio 40). Para análises batch (diárias/semanais), Standard Logs + Athena são perfeitos e muito mais baratos. Use partition projection sempre — ele elimina a necessidade de `MSCK REPAIR TABLE` que é lento e falha em buckets grandes.

---

## Desafio 40: Real-Time Logs com Kinesis Data Streams

> **Level:** 400 | **Tempo:** 60 min | **Custo:** ~$5-15/mês (Kinesis + processamento)

### Objetivo
Configurar real-time logging para análise instantânea de tráfego e detecção de problemas em tempo real.

### Standard Logs vs Real-Time Logs

| Aspecto | Standard Logs | Real-Time Logs |
|---------|--------------|----------------|
| **Delay** | 5-60 minutos | 1-2 segundos |
| **Destino** | S3 | Kinesis Data Streams |
| **Formato** | TSV (tab-separated) | JSON |
| **Sampling** | 100% (tudo) | 1-100% (configurável) |
| **Campos** | 33 fixos | 40+ selecionáveis |
| **Custo** | Apenas S3 storage | Kinesis + processamento |
| **Por behavior** | Não (distribution inteira) | Sim |
| **Melhor para** | Análise batch, compliance | Debugging, alertas real-time |

### Terraform Completo

```hcl
# Kinesis Data Stream
resource "aws_kinesis_stream" "cf_realtime" {
  name             = "cloudfront-realtime-logs"
  shard_count      = 2  # Ajustar baseado no volume
  retention_period = 24

  stream_mode_details {
    stream_mode = "PROVISIONED"  # ou ON_DEMAND
  }
}

# IAM Role para CloudFront escrever no Kinesis
resource "aws_iam_role" "cf_realtime_logs" {
  name = "cloudfront-realtime-logs-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "cf_realtime_logs" {
  name = "kinesis-write"
  role = aws_iam_role.cf_realtime_logs.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["kinesis:DescribeStreamSummary", "kinesis:DescribeStream", "kinesis:PutRecord", "kinesis:PutRecords"]
      Resource = aws_kinesis_stream.cf_realtime.arn
    }]
  })
}

# Real-Time Log Config
resource "aws_cloudfront_realtime_log_config" "main" {
  name          = "realtime-all-fields"
  sampling_rate = 100  # 100% para debugging; reduzir em produção

  fields = [
    "timestamp",
    "c-ip",
    "sc-status",
    "cs-uri-stem",
    "cs-method",
    "cs-bytes",
    "sc-bytes",
    "time-taken",
    "time-to-first-byte",
    "x-edge-location",
    "x-edge-result-type",
    "x-edge-detailed-result-type",
    "cs-protocol",
    "cs-protocol-version",
    "ssl-protocol",
    "cs(User-Agent)",
    "cs(Referer)",
    "cs-uri-query",
    "cs(Host)",
    "x-forwarded-for",
    "sc-content-type",
    "c-country",
    "cs-header-names"
  ]

  endpoint {
    stream_type = "Kinesis"
    kinesis_stream_config {
      role_arn   = aws_iam_role.cf_realtime_logs.arn
      stream_arn = aws_kinesis_stream.cf_realtime.arn
    }
  }
}

# Associar ao behavior
resource "aws_cloudfront_distribution" "main" {
  # ...
  default_cache_behavior {
    # ...
    realtime_log_config_arn = aws_cloudfront_realtime_log_config.main.arn
  }
}
```

### Lambda Consumer para Processar Logs

```python
# lambda_consumer.py
import json
import base64
import boto3
from datetime import datetime

cloudwatch = boto3.client('cloudwatch')

def handler(event, context):
    metrics = {
        'total_requests': 0,
        'cache_hits': 0,
        'cache_misses': 0,
        'errors_4xx': 0,
        'errors_5xx': 0,
        'total_bytes': 0,
        'total_latency': 0
    }

    for record in event['Records']:
        payload = base64.b64decode(record['kinesis']['data']).decode('utf-8')

        for line in payload.strip().split('\n'):
            try:
                fields = line.split('\t')
                if len(fields) < 10:
                    continue

                metrics['total_requests'] += 1

                status = int(fields[3]) if fields[3] != '-' else 0
                result_type = fields[10] if len(fields) > 10 else ''
                sc_bytes = int(fields[6]) if fields[6] != '-' else 0
                time_taken = float(fields[7]) if fields[7] != '-' else 0

                if result_type == 'Hit':
                    metrics['cache_hits'] += 1
                elif result_type == 'Miss':
                    metrics['cache_misses'] += 1

                if 400 <= status < 500:
                    metrics['errors_4xx'] += 1
                elif status >= 500:
                    metrics['errors_5xx'] += 1

                metrics['total_bytes'] += sc_bytes
                metrics['total_latency'] += time_taken

            except (ValueError, IndexError):
                continue

    # Enviar métricas customizadas para CloudWatch
    if metrics['total_requests'] > 0:
        cloudwatch.put_metric_data(
            Namespace='CloudFront/RealTime',
            MetricData=[
                {
                    'MetricName': 'RequestCount',
                    'Value': metrics['total_requests'],
                    'Unit': 'Count',
                    'Timestamp': datetime.utcnow()
                },
                {
                    'MetricName': 'CacheHitRate',
                    'Value': (metrics['cache_hits'] / metrics['total_requests']) * 100,
                    'Unit': 'Percent'
                },
                {
                    'MetricName': '5xxErrorRate',
                    'Value': (metrics['errors_5xx'] / metrics['total_requests']) * 100,
                    'Unit': 'Percent'
                },
                {
                    'MetricName': 'AverageLatencyMs',
                    'Value': (metrics['total_latency'] / metrics['total_requests']) * 1000,
                    'Unit': 'Milliseconds'
                }
            ]
        )

    return {'statusCode': 200, 'processed': metrics['total_requests']}
```

### Dica Expert
> **Comece com sampling rate de 1-5% em produção** e aumente conforme necessário. 100% de sampling em um site com 10k req/s = 864M records/dia = custo alto de Kinesis. Para debugging pontual, aumente temporariamente para 100%. Use Kinesis Data Firehose entre Kinesis e S3/OpenSearch para delivery automático.

---

## Desafio 41: CloudWatch Metrics e Dashboards

> **Level:** 300 | **Tempo:** 45 min | **Custo:** Free tier (dashboards: $3/mês cada além dos 3 grátis)

### Todas as Métricas CloudFront

#### Métricas Padrão (Grátis)

| Métrica | Unidade | Descrição |
|---------|---------|-----------|
| `Requests` | Count | Total de requests |
| `BytesDownloaded` | Bytes | Bytes transferidos para viewers |
| `BytesUploaded` | Bytes | Bytes recebidos dos viewers |
| `TotalErrorRate` | Percent | % de requests com erro (4xx+5xx) |
| `4xxErrorRate` | Percent | % de erros 4xx |
| `5xxErrorRate` | Percent | % de erros 5xx |
| `CacheHitRate` | Percent | % de requests servidas do cache |
| `OriginLatency` | Milliseconds | Latência entre CF e origin |

#### Métricas Adicionais (Custo: $0.30/métrica/mês)

| Métrica | Unidade | Descrição |
|---------|---------|-----------|
| `401ErrorRate` | Percent | % de 401 Unauthorized |
| `403ErrorRate` | Percent | % de 403 Forbidden |
| `404ErrorRate` | Percent | % de 404 Not Found |
| `502ErrorRate` | Percent | % de 502 Bad Gateway |
| `503ErrorRate` | Percent | % de 503 Service Unavailable |
| `504ErrorRate` | Percent | % de 504 Gateway Timeout |

#### Métricas de CloudFront Functions

| Métrica | Unidade | Descrição |
|---------|---------|-----------|
| `FunctionInvocations` | Count | Total de invocações |
| `FunctionValidationErrors` | Count | Erros de validação |
| `FunctionExecutionErrors` | Count | Erros de execução |
| `FunctionComputeUtilization` | Percent | % do tempo de CPU usado (alvo: <80%) |
| `FunctionThrottles` | Count | Requests throttled |

### Dashboard CloudWatch Completo

```hcl
resource "aws_cloudwatch_dashboard" "cloudfront" {
  dashboard_name = "CloudFront-Production"
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title   = "Requests por Minuto"
          metrics = [["AWS/CloudFront", "Requests", "DistributionId", var.distribution_id, "Region", "Global", { stat = "Sum", period = 60 }]]
          view    = "timeSeries"
          region  = "us-east-1"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 6
        height = 6
        properties = {
          title   = "Cache Hit Rate"
          metrics = [["AWS/CloudFront", "CacheHitRate", "DistributionId", var.distribution_id, "Region", "Global", { stat = "Average", period = 300 }]]
          view    = "gauge"
          yAxis   = { left = { min = 0, max = 100 } }
        }
      },
      {
        type   = "metric"
        x      = 18
        y      = 0
        width  = 6
        height = 6
        properties = {
          title = "Error Rates"
          metrics = [
            ["AWS/CloudFront", "5xxErrorRate", "DistributionId", var.distribution_id, "Region", "Global", { stat = "Average", period = 300, color = "#d62728" }],
            ["AWS/CloudFront", "4xxErrorRate", "DistributionId", var.distribution_id, "Region", "Global", { stat = "Average", period = 300, color = "#ff7f0e" }]
          ]
          view = "timeSeries"
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 12
        height = 6
        properties = {
          title = "Origin Latency (p50, p90, p99)"
          metrics = [
            ["AWS/CloudFront", "OriginLatency", "DistributionId", var.distribution_id, "Region", "Global", { stat = "p50", period = 300, label = "p50" }],
            ["...", { stat = "p90", period = 300, label = "p90" }],
            ["...", { stat = "p99", period = 300, label = "p99", color = "#d62728" }]
          ]
          view = "timeSeries"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 6
        width  = 12
        height = 6
        properties = {
          title = "Bandwidth (Bytes Downloaded)"
          metrics = [["AWS/CloudFront", "BytesDownloaded", "DistributionId", var.distribution_id, "Region", "Global", { stat = "Sum", period = 300 }]]
          view = "timeSeries"
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 12
        width  = 12
        height = 6
        properties = {
          title = "Function Compute Utilization"
          metrics = [["AWS/CloudFront", "FunctionComputeUtilization", "DistributionId", var.distribution_id, "FunctionName", "viewer-request-handler", "Region", "Global", { stat = "Average", period = 60 }]]
          view      = "timeSeries"
          yAxis     = { left = { min = 0, max = 100 } }
          annotations = { horizontal = [{ value = 80, label = "Danger Zone", color = "#d62728" }] }
        }
      }
    ]
  })
}
```

---

## Desafio 42: CloudWatch Alarms

> **Level:** 300 | **Tempo:** 30 min | **Custo:** $0.10/alarm/mês

### 7 Alarmes Essenciais

```hcl
# SNS Topic para notificações
resource "aws_sns_topic" "cloudfront_alerts" {
  name = "cloudfront-alerts"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.cloudfront_alerts.arn
  protocol  = "email"
  endpoint  = "sre-team@meusite.com"
}

# Alarm 1: 5xx Error Rate > 1% por 5 minutos (CRITICAL)
resource "aws_cloudwatch_metric_alarm" "error_5xx" {
  alarm_name          = "CloudFront-5xxErrorRate-Critical"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "5xxErrorRate"
  namespace           = "AWS/CloudFront"
  period              = 300
  statistic           = "Average"
  threshold           = 1
  alarm_description   = "5xx error rate > 1% por 5 minutos"
  alarm_actions       = [aws_sns_topic.cloudfront_alerts.arn]
  ok_actions          = [aws_sns_topic.cloudfront_alerts.arn]
  dimensions = {
    DistributionId = var.distribution_id
    Region         = "Global"
  }
}

# Alarm 2: 4xx Error Rate > 10% por 10 min (WARNING)
resource "aws_cloudwatch_metric_alarm" "error_4xx" {
  alarm_name          = "CloudFront-4xxErrorRate-Warning"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "4xxErrorRate"
  namespace           = "AWS/CloudFront"
  period              = 300
  statistic           = "Average"
  threshold           = 10
  alarm_description   = "4xx error rate > 10% por 10 minutos"
  alarm_actions       = [aws_sns_topic.cloudfront_alerts.arn]
  dimensions = {
    DistributionId = var.distribution_id
    Region         = "Global"
  }
}

# Alarm 3: Cache Hit Rate < 80% por 1 hora (WARNING)
resource "aws_cloudwatch_metric_alarm" "cache_hit_low" {
  alarm_name          = "CloudFront-CacheHitRate-Low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 12
  metric_name         = "CacheHitRate"
  namespace           = "AWS/CloudFront"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "Cache hit rate < 80% por 1 hora"
  alarm_actions       = [aws_sns_topic.cloudfront_alerts.arn]
  dimensions = {
    DistributionId = var.distribution_id
    Region         = "Global"
  }
}

# Alarm 4: Origin Latency p99 > 2s por 5 min (CRITICAL)
resource "aws_cloudwatch_metric_alarm" "origin_latency" {
  alarm_name          = "CloudFront-OriginLatency-Critical"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "OriginLatency"
  namespace           = "AWS/CloudFront"
  period              = 300
  extended_statistic  = "p99"
  threshold           = 2000  # 2 seconds in ms
  alarm_description   = "Origin latency p99 > 2s por 5 minutos"
  alarm_actions       = [aws_sns_topic.cloudfront_alerts.arn]
  dimensions = {
    DistributionId = var.distribution_id
    Region         = "Global"
  }
}

# Alarm 5: Total Error Rate > 5% (CRITICAL)
resource "aws_cloudwatch_metric_alarm" "total_error" {
  alarm_name          = "CloudFront-TotalErrorRate-Critical"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "TotalErrorRate"
  namespace           = "AWS/CloudFront"
  period              = 300
  statistic           = "Average"
  threshold           = 5
  alarm_actions       = [aws_sns_topic.cloudfront_alerts.arn]
  dimensions = {
    DistributionId = var.distribution_id
    Region         = "Global"
  }
}

# Alarm 6: Function Throttles > 0 (WARNING)
resource "aws_cloudwatch_metric_alarm" "function_throttles" {
  alarm_name          = "CloudFront-FunctionThrottles-Warning"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FunctionThrottles"
  namespace           = "AWS/CloudFront"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_actions       = [aws_sns_topic.cloudfront_alerts.arn]
  dimensions = {
    DistributionId = var.distribution_id
    Region         = "Global"
  }
}

# Alarm 7: Anomaly Detection — Requests (detecta spikes/drops)
resource "aws_cloudwatch_metric_alarm" "requests_anomaly" {
  alarm_name          = "CloudFront-Requests-Anomaly"
  comparison_operator = "LessThanLowerOrGreaterThanUpperThreshold"
  evaluation_periods  = 2
  threshold_metric_id = "ad1"
  alarm_actions       = [aws_sns_topic.cloudfront_alerts.arn]
  alarm_description   = "Anomalia detectada no volume de requests"

  metric_query {
    id          = "m1"
    return_data = true
    metric {
      metric_name = "Requests"
      namespace   = "AWS/CloudFront"
      period      = 300
      stat        = "Sum"
      dimensions = {
        DistributionId = var.distribution_id
        Region         = "Global"
      }
    }
  }

  metric_query {
    id          = "ad1"
    expression  = "ANOMALY_DETECTION_BAND(m1, 2)"
    label       = "Requests (Expected)"
    return_data = true
  }
}
```

---

## Desafio 43: Reports & Analytics do Console

> **Level:** 300 | **Tempo:** 20 min | **Custo:** Grátis

### Os 5 Reports do Console CloudFront

| Report | O Que Mostra | Insight Principal |
|--------|-------------|-------------------|
| **Cache Statistics** | Hit/Miss ratio, bytes por resultado, requests por resultado | Performance do cache |
| **Popular Objects** | Top 50 URLs, requests, hits, misses, bytes | Quais objetos otimizar |
| **Top Referrers** | De onde vem o tráfego (HTTP Referer) | Marketing, hotlinking |
| **Usage** | Requests e data transfer por protocolo e região | Dimensionamento e custo |
| **Viewers** | Devices, browsers, OS, locations | Audiência e compatibilidade |

### Como Extrair Insights

```bash
# Cache Statistics via CLI (métricas CloudWatch)
aws cloudwatch get-metric-statistics \
    --namespace AWS/CloudFront \
    --metric-name CacheHitRate \
    --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
    --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 86400 --statistics Average --output table
```

---

## Desafio 44: Observabilidade End-to-End

> **Level:** 400 | **Tempo:** 60 min

### Arquitetura Completa de Observabilidade

```
CloudFront Distribution
    │
    ├── Standard Logs ──→ S3 ──→ Athena (análise batch diária/semanal)
    │                              │
    │                        Queries salvas + Reports
    │
    ├── Real-Time Logs ──→ Kinesis ──→ Lambda Consumer ──→ CloudWatch Custom Metrics
    │                         │
    │                    Firehose ──→ OpenSearch (análise real-time, dashboards)
    │                         │
    │                    Firehose ──→ S3 (backup dos real-time logs)
    │
    ├── CloudWatch Metrics ──→ Dashboards (visualização)
    │                     ──→ Alarms (alertas proativos)
    │                     ──→ SNS ──→ Email + Slack + PagerDuty
    │
    └── X-Ray Traces ──→ Lambda@Edge execution traces
```

### Runbooks de Troubleshooting

#### Runbook 1: Cache Hit Rate Caiu de 95% para 60%

```
1. VERIFICAR: CloudWatch → CacheHitRate por behavior
   - Todos os behaviors caíram ou apenas um?
   - Quando começou a queda?

2. SE todos caíram:
   - Alguém fez invalidation de /* recentemente?
     → aws cloudfront list-invalidations --distribution-id $DIST_ID
   - Houve mudança na Cache Policy?
     → Verificar git log do Terraform
   - Houve deploy que mudou Cache-Control headers na origin?

3. SE apenas um behavior:
   - Verificar se query strings ou headers novos estão na cache key
   - Verificar se há bot/scraper gerando URLs únicas
     → Athena: Top IPs + URLs com mais MISS
   - Verificar se houve mudança no padrão de tráfego

4. AÇÃO: Se causado por deploy, rollback. Se por bot, bloquear no WAF.
   Após correção, invalidar /* para limpar cache poluído.
```

#### Runbook 2: Erros 502 Intermitentes

```
1. 502 Bad Gateway = CloudFront não conseguiu parsear a resposta da origin

2. VERIFICAR: Origin está healthy?
   → curl -I <origin-direct-url>
   → aws elbv2 describe-target-health (se ALB)

3. VERIFICAR: Origin está respondendo muito lento?
   → CloudWatch → OriginLatency p99
   → Se > read timeout (default 30s), ajustar origin_read_timeout

4. VERIFICAR: Origin está retornando response inválida?
   → Logs do origin (ALB access logs, Nginx error logs)
   → Verificar se origin está retornando headers HTTP inválidos

5. VERIFICAR: SSL/TLS mismatch?
   → Se origin_protocol_policy = "https-only", verificar certificado da origin
   → openssl s_client -connect origin:443

6. AÇÃO: Corrigir health da origin. Se timeout, aumentar read timeout.
   Se persistir, habilitar Origin Failover com Origin Group.
```

#### Runbook 3: Alta Latência para Usuários no Brasil

```
1. VERIFICAR: É latência no edge ou na origin?
   → CloudWatch → OriginLatency (se alta, é a origin)
   → Se OriginLatency normal, é routing/network

2. SE origin latency alta:
   → Origin está em qual região? (se us-east-1, considerar sa-east-1)
   → Origin Shield habilitado? (adicionar em sa-east-1)
   → Verificar health da origin na região

3. SE latência no edge:
   → Verificar PriceClass (PriceClass_100 exclui América do Sul!)
   → Mudar para PriceClass_200 ou PriceClass_All
   → Verificar se edge locations brasileiras estão servindo
     → Athena: x_edge_location WHERE c_ip LIKE '200.%' (IPs brasileiros)

4. AÇÃO: Adicionar Origin Shield em sa-east-1, mudar PriceClass se necessário.
```

#### Runbook 4: Spike de Erros 403

```
1. 403 Forbidden pode ser:
   a) WAF bloqueando requests legítimos
   b) Signed URLs/Cookies expirados
   c) Geo Restriction bloqueando
   d) Bucket Policy S3 rejeitando
   e) OAC misconfigured

2. VERIFICAR WAF:
   → aws wafv2 get-sampled-requests (ver requests bloqueadas)
   → Qual regra está bloqueando?
   → Se managed rule: adicionar exceção ou mudar para COUNT

3. VERIFICAR Signed URLs:
   → Os tokens estão expirados?
   → A key pair está correta?

4. VERIFICAR Geo:
   → Algum país foi bloqueado recentemente?

5. VERIFICAR S3:
   → Bucket policy aceita o ARN da distribution?
   → OAC está configurado corretamente?

6. AÇÃO: Depende da causa. Se WAF, ajustar regra. Se OAC, corrigir policy.
```

### O Que Aprendemos

| Conceito | Descrição |
|----------|-----------|
| **Standard Logs** | Análise batch em S3 + Athena (5-60 min delay) |
| **Real-Time Logs** | Análise instantânea via Kinesis (1-2s delay) |
| **CloudWatch Metrics** | 8+ métricas padrão + additional metrics |
| **CloudWatch Alarms** | Alertas proativos com SNS |
| **Anomaly Detection** | ML-based detection de padrões anormais |
| **Runbooks** | Procedimentos documentados para troubleshooting |

---

## Resumo do Módulo 07

```
Observability Stack
├── LOGS
│   ├── Standard Logs → S3 → Athena (batch analysis)
│   └── Real-Time Logs → Kinesis → Lambda/OpenSearch (real-time)
├── METRICS
│   ├── CloudWatch Metrics (8+ standard, 6+ additional)
│   └── Custom Metrics (via Lambda consumer)
├── DASHBOARDS
│   └── CloudWatch Dashboards (6+ widgets essenciais)
├── ALARMS
│   ├── 7 alarmes essenciais (5xx, 4xx, cache, latency, etc)
│   └── Anomaly Detection (ML-based)
└── RUNBOOKS
    ├── Cache hit ratio caiu
    ├── Erros 502 intermitentes
    ├── Alta latência regional
    └── Spike de 403
```

**Próximo:** [Módulo 08 — Multi-tenant, SaaS e Features Avançadas →](modulo-08-multitenant-saas.md)
