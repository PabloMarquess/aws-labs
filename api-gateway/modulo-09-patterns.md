# Módulo 09 — Padrões Avançados

> **Nível:** 400 (Expert)
> **Tempo Total Estimado:** 10-14 horas de labs
> **Custo Estimado:** ~$5-10
> **Objetivo do Módulo:** Implementar padrões arquiteturais avançados — API composition para microservices, transformação VTL avançada, API Gateway como BFF, multi-region failover, OpenAPI-first development e API documentation.

---

## Mapa do Módulo

```mermaid
graph TB
    MOD[Módulo 09<br/>Padrões Avançados]

    MOD --> D52[D.52 API Composition<br/>Microservices]
    MOD --> D53[D.53 VTL Avançado<br/>Transformation]
    MOD --> D54[D.54 BFF Pattern<br/>Backend for Frontend]
    MOD --> D55[D.55 Multi-Region<br/>Failover]
    MOD --> D56[D.56 OpenAPI-First<br/>Development]
    MOD --> D57[D.57 API Documentation<br/>Developer Portal]

    style MOD fill:#e94560,color:#fff
    style D52 fill:#0f3460,color:#fff
    style D54 fill:#533483,color:#fff
    style D55 fill:#16213e,color:#fff
```

---

## Desafio 52: API Composition — Microservices Backend

> **Level:** 400 | **Tempo:** 90 min | **Custo:** ~$2

### Objetivo

Usar API Gateway como **API composition layer** — uma API unificada na frente de múltiplos microservices.

### Arquitetura

```mermaid
graph TB
    CLIENT[Mobile App / Web] --> APIGW[API Gateway<br/>api.empresa.com]

    APIGW -->|"/users/*"| USERS[Users Service<br/>Lambda]
    APIGW -->|"/orders/*"| ORDERS[Orders Service<br/>ECS + ALB via VPC Link]
    APIGW -->|"/payments/*"| PAYMENTS[Payments Service<br/>Step Functions]
    APIGW -->|"/notifications/*"| NOTIF[Notifications<br/>SQS Direct]
    APIGW -->|"/search/*"| SEARCH[Search Service<br/>OpenSearch via Lambda]

    USERS --> DDB_U[(DynamoDB)]
    ORDERS --> RDS[(RDS)]
    PAYMENTS --> DDB_P[(DynamoDB)]

    style APIGW fill:#e94560,color:#fff
    style USERS fill:#0f3460,color:#fff
    style ORDERS fill:#533483,color:#fff
    style PAYMENTS fill:#16213e,color:#fff
```

### Benefícios

```
API Gateway como API Composition:
├── 1 URL para o client (api.empresa.com)
├── Backend heterogêneo (Lambda, ECS, Step Functions, SQS)
├── Auth centralizada (1 Cognito Authorizer para todos)
├── Throttling por serviço (usage plans, method-level)
├── Monitoramento centralizado (CloudWatch, X-Ray)
├── CORS centralizado (1 configuração)
└── Versionamento centralizado (stages)

Sem API Gateway:
├── Client precisa conhecer N URLs
├── N configurações de auth
├── N configurações de CORS
├── N monitoring setups
└── Complexidade exponencial
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| API Composition | 1 API Gateway na frente de N microservices |
| Single entry point | Client conhece apenas 1 URL |
| Mixed backends | Lambda + ECS + Step Functions + SQS no mesmo API |
| Centralized concerns | Auth, CORS, throttling, monitoring em um lugar |

---

## Desafio 55: Multi-Region API com Route 53 Failover

> **Level:** 400 | **Tempo:** 120 min | **Custo:** ~$5

### Objetivo

Implementar **multi-region failover** para APIs — alta disponibilidade com Route 53 health checks.

### Arquitetura

```mermaid
graph TB
    CLIENT[Clientes] --> R53[Route 53<br/>api.empresa.com<br/>Failover routing]

    R53 -->|Primary| API_US[API Gateway<br/>us-east-1<br/>Regional]
    R53 -->|Secondary| API_EU[API Gateway<br/>eu-west-1<br/>Regional]

    API_US --> L_US[Lambda us-east-1]
    API_EU --> L_EU[Lambda eu-west-1]

    L_US --> DDB[DynamoDB<br/>Global Tables]
    L_EU --> DDB

    HC[Route 53<br/>Health Check] -.->|Monitor| API_US
    HC -.->|Monitor| API_EU

    style R53 fill:#e94560,color:#fff
    style API_US fill:#0f3460,color:#fff
    style API_EU fill:#533483,color:#fff
    style DDB fill:#16213e,color:#fff
```

```hcl
# Health Check para API primária
resource "aws_route53_health_check" "api_us" {
  fqdn              = aws_api_gateway_domain_name.api_us.regional_domain_name
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 30
}

# DNS Failover
resource "aws_route53_record" "api_primary" {
  zone_id = var.hosted_zone_id
  name    = "api.empresa.com"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier = "primary"
  health_check_id = aws_route53_health_check.api_us.id

  alias {
    name                   = aws_api_gateway_domain_name.api_us.regional_domain_name
    zone_id                = aws_api_gateway_domain_name.api_us.regional_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_secondary" {
  zone_id = var.hosted_zone_id
  name    = "api.empresa.com"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "secondary"

  alias {
    name                   = aws_api_gateway_domain_name.api_eu.regional_domain_name
    zone_id                = aws_api_gateway_domain_name.api_eu.regional_zone_id
    evaluate_target_health = true
  }
}
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| Multi-region | Mesmo API deployado em 2+ regiões |
| Route 53 Failover | Health check → switch automático para region backup |
| DynamoDB Global Tables | Dados replicados entre regiões automaticamente |
| Custom domain | Mesmo domínio para ambas as regiões (Route 53 roteia) |
| RTO | Recovery Time Objective: ~60 segundos (health check interval) |

> **💡 Expert Tip:** Multi-region API é caro (2x infra) e complexo (data consistency). Use apenas quando o SLA exige 99.99%+ ou regulação exige multi-region. Para a maioria dos casos, uma API regional com Lambda (que já tem HA multi-AZ) é suficiente. DynamoDB Global Tables resolve a consistência de dados mas tem eventual consistency — cuidado com operações que exigem strong consistency.

---

## Resumo do Módulo 09

```
┌──────────────────────────────────────────────────────────────┐
│               MÓDULO 09 — CONQUISTAS                          │
│                                                               │
│  ✅ Desafio 52: API Composition (Microservices)              │
│  ✅ Desafio 53: VTL Avançado                                 │
│  ✅ Desafio 54: BFF Pattern                                  │
│  ✅ Desafio 55: Multi-Region Failover                        │
│  ✅ Desafio 56: OpenAPI-First Development                    │
│  ✅ Desafio 57: API Documentation                            │
│                                                               │
│  Próximo: Módulo 10 — Cenários Expert                        │
└──────────────────────────────────────────────────────────────┘
```

**Próximo:** [Módulo 10 — Cenários Expert →](modulo-10-cenarios-expert.md)
