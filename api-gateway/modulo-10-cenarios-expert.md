# Módulo 10 — Cenários Expert (Grand Finale)

> **Nível:** 400 (Expert)
> **Tempo Total Estimado:** 12-16 horas de labs
> **Custo Estimado:** ~$10-20
> **Objetivo do Módulo:** Aplicar tudo dos módulos 01-09 em cenários reais — plataforma SaaS multi-tenant, migração monolith para microservices, API marketplace com monetização, event-driven architecture e decisão GraphQL vs REST.

---

## Mapa do Módulo Final

```mermaid
graph TB
    MOD[Módulo 10<br/>Grand Finale]

    MOD --> D58[D.58 CAPSTONE<br/>SaaS Multi-Tenant]
    MOD --> D59[D.59 Migração<br/>Monolith → Microservices]
    MOD --> D60[D.60 API Marketplace<br/>Monetização]
    MOD --> D61[D.61 Event-Driven<br/>API GW → EventBridge]
    MOD --> D62[D.62 GraphQL vs REST<br/>Decision Framework]
    MOD --> D63[D.63 Private API<br/>Microservices Internos]
    MOD --> D64[D.64 API Versioning<br/>Backward Compatibility]
    MOD --> D65[D.65 Certificação<br/>Próximos Passos]

    style MOD fill:#e94560,color:#fff
    style D58 fill:#0f3460,color:#fff
    style D59 fill:#533483,color:#fff
    style D60 fill:#533483,color:#fff
    style D62 fill:#16213e,color:#fff
```

---

## Desafio 58: CAPSTONE — Plataforma SaaS Multi-Tenant

> **Level:** 400 | **Tempo:** 180 min | **Custo:** ~$10

### Objetivo

Projetar a arquitetura completa de API para uma **plataforma SaaS multi-tenant** — isolamento por tenant, usage plans por plano de preço, auth centralizada e monitoring por tenant.

### Arquitetura

```mermaid
graph TB
    subgraph Tenants
        T1[Tenant A<br/>Plan: Free<br/>API Key: abc]
        T2[Tenant B<br/>Plan: Pro<br/>API Key: def]
        T3[Tenant C<br/>Plan: Enterprise<br/>API Key: ghi]
    end

    T1 --> APIGW[API Gateway<br/>api.saas-platform.com]
    T2 --> APIGW
    T3 --> APIGW

    APIGW --> AUTH[Lambda Authorizer<br/>Validate API Key +<br/>Resolve Tenant]

    AUTH --> FREE_PLAN[Usage Plan: Free<br/>100 req/dia]
    AUTH --> PRO_PLAN[Usage Plan: Pro<br/>10K req/dia]
    AUTH --> ENT_PLAN[Usage Plan: Enterprise<br/>1M req/dia]

    APIGW --> LAMBDA[Lambda<br/>Tenant-aware]
    LAMBDA --> DDB[(DynamoDB<br/>PK: tenantId#resourceId)]

    style APIGW fill:#0f3460,color:#fff
    style AUTH fill:#e94560,color:#fff
    style DDB fill:#533483,color:#fff
```

### Tenant Isolation Pattern

```python
# Lambda: tenant-aware CRUD
def handler(event, context):
    # Extrair tenant do authorizer context
    tenant_id = event['requestContext']['authorizer']['tenantId']
    plan = event['requestContext']['authorizer']['plan']

    method = event['httpMethod']
    path_params = event.get('pathParameters', {}) or {}

    if method == 'GET' and 'id' not in path_params:
        # Listar: query por tenant (NÃO scan!)
        result = table.query(
            KeyConditionExpression=Key('pk').eq(f'TENANT#{tenant_id}'),
            ScanIndexForward=False,
            Limit=100
        )
        return response(200, {'items': result['Items']})

    elif method == 'POST':
        body = json.loads(event['body'])
        item = {
            'pk': f'TENANT#{tenant_id}',
            'sk': f'RESOURCE#{uuid.uuid4()}',
            'tenantId': tenant_id,
            'data': body,
            'createdAt': datetime.utcnow().isoformat()
        }
        table.put_item(Item=item)
        return response(201, item)

    # Segurança: nunca acessar dados de outro tenant!
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| Multi-tenant | 1 API serve múltiplos clientes isolados |
| Tenant isolation | DynamoDB PK com prefixo tenantId |
| Usage Plans | Planos diferentes por tier de cliente |
| Lambda Authorizer | Resolve tenant + valida permissões |
| Monitoring por tenant | Logs com tenantId para análise por cliente |

---

## Desafio 59: Migração Monolith → Microservices

> **Level:** 400 | **Tempo:** 120 min | **Custo:** ~$5

### Objetivo

Usar API Gateway como **strangler fig** para migrar gradualmente de monolith para microservices.

### Pattern Strangler Fig

```mermaid
graph TB
    subgraph Fase1 ["Fase 1: API Gateway na frente"]
        APIGW1[API Gateway] -->|"/* (default)"| MONO[Monolith<br/>ALB + EC2]
    end

    subgraph Fase2 ["Fase 2: Extrair primeiro serviço"]
        APIGW2[API Gateway]
        APIGW2 -->|"/users/*"| USERS[Users Service<br/>Lambda]
        APIGW2 -->|"/* (default)"| MONO2[Monolith<br/>Restante]
    end

    subgraph Fase3 ["Fase 3: Mais serviços"]
        APIGW3[API Gateway]
        APIGW3 -->|"/users/*"| USERS3[Users<br/>Lambda]
        APIGW3 -->|"/orders/*"| ORDERS3[Orders<br/>ECS]
        APIGW3 -->|"/payments/*"| PAY3[Payments<br/>Lambda]
        APIGW3 -->|"/* (default)"| MONO3[Monolith<br/>Mínimo]
    end

    subgraph Fase4 ["Fase 4: Monolith eliminado"]
        APIGW4[API Gateway]
        APIGW4 -->|"/users/*"| USERS4[Users]
        APIGW4 -->|"/orders/*"| ORDERS4[Orders]
        APIGW4 -->|"/payments/*"| PAY4[Payments]
        APIGW4 -->|"/notifications/*"| NOTIF4[Notifications]
    end

    style Fase1 fill:#16213e,color:#fff
    style Fase2 fill:#533483,color:#fff
    style Fase3 fill:#0f3460,color:#fff
    style Fase4 fill:#e94560,color:#fff
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| Strangler Fig | API GW na frente → extrair serviços um a um |
| Default route | `/*` aponta para monolith (catch-all) |
| Gradual migration | Cada sprint extrai 1 serviço, sem big bang |
| Zero downtime | Client não percebe a migração (mesmo URL) |
| VPC Link | Monolith continua no ALB via VPC Link |

---

## Desafio 60: API Marketplace com Monetização

> **Level:** 400 | **Tempo:** 120 min | **Custo:** ~$2

### Objetivo

Criar uma **API marketplace** com múltiplos planos de preço usando Usage Plans e API Keys.

```mermaid
graph TB
    subgraph Plans
        FREE[Free<br/>100 req/dia<br/>$0/mês]
        BASIC[Basic<br/>10K req/dia<br/>$29/mês]
        PRO[Pro<br/>100K req/dia<br/>$99/mês]
        ENT[Enterprise<br/>Unlimited<br/>Custom pricing]
    end

    subgraph Features
        FREE --> F1[API básica<br/>Rate limit: 10/s]
        BASIC --> F2[API completa<br/>Rate limit: 100/s<br/>+ Webhooks]
        PRO --> F3[API completa<br/>Rate limit: 1000/s<br/>+ Webhooks<br/>+ Priority support]
        ENT --> F4[Tudo<br/>Sem rate limit<br/>+ SLA 99.99%<br/>+ Dedicated support]
    end

    style FREE fill:#16213e,color:#fff
    style BASIC fill:#533483,color:#fff
    style PRO fill:#0f3460,color:#fff
    style ENT fill:#e94560,color:#fff
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| API marketplace | Monetização via Usage Plans + API Keys |
| Tiers | Free → Basic → Pro → Enterprise |
| Metering | `aws apigateway get-usage` para billing |
| Self-service | API Key provisioning via portal do developer |

---

## Desafio 61: Event-Driven Architecture

> **Level:** 400 | **Tempo:** 90 min | **Custo:** ~$1

### Objetivo

Integrar API Gateway com **Amazon EventBridge** para arquitetura event-driven.

```mermaid
graph LR
    CLIENT[Cliente] -->|"POST /events"| APIGW[API Gateway]
    APIGW -->|Direct Integration| EB[EventBridge<br/>Custom Bus]

    EB -->|Rule: order.created| L1[Lambda<br/>Process Order]
    EB -->|Rule: order.created| L2[Lambda<br/>Send Email]
    EB -->|Rule: order.created| SQS[SQS<br/>Analytics Queue]
    EB -->|Rule: payment.failed| L3[Lambda<br/>Alert Team]

    style APIGW fill:#0f3460,color:#fff
    style EB fill:#e94560,color:#fff
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| EventBridge integration | API GW publica eventos diretamente (sem Lambda) |
| Fan-out | 1 evento → múltiplos consumers |
| Decoupling | Producer não conhece consumers |
| Event schema | Estruturar eventos com source, detail-type, detail |

---

## Desafio 62: GraphQL vs REST

> **Level:** 400 | **Tempo:** 60 min | **Custo:** $0

### Decision Framework

| Aspecto | REST (API Gateway) | GraphQL (AppSync) |
|---------|-------------------|-------------------|
| **Quando usar** | CRUD simples, APIs públicas, microservices | Múltiplas fontes, mobile apps, aggregation |
| **Over-fetching** | Comum (retorna campos extras) | Não (client escolhe campos) |
| **Under-fetching** | Comum (múltiplas calls) | Não (1 query = tudo) |
| **Caching** | Simples (por URL) | Complexo (por query) |
| **Real-time** | WebSocket API (separado) | Subscriptions (nativo) |
| **Pricing** | $1-3.50/M calls | $4/M queries + $2/M updates |
| **Learning curve** | Baixa | Média-alta |
| **Tooling** | curl, Postman | Apollo, GraphQL Playground |

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| REST wins | APIs simples, públicas, microservices, padrão da indústria |
| GraphQL wins | Mobile apps com múltiplas fontes, dashboards complexos |
| Não é ou | Pode usar ambos — REST para público, GraphQL para interno |

---

## Desafio 65: Certificação e Próximos Passos

> **Level:** 400 | **Tempo:** 60 min | **Custo:** $0

### O Que Este Workshop Cobriu

```
┌──────────────────────────────────────────────────────────────┐
│               API GATEWAY WORKSHOP — COMPLETO                 │
│                                                               │
│  65 desafios · 10 módulos · Level 100 → 400                 │
│                                                               │
│  REST API: resources, methods, integrations, caching         │
│  HTTP API: simples, rápido, JWT nativo, 70% mais barato     │
│  WebSocket API: real-time, chat, broadcasting                │
│  Auth: Cognito, Lambda Authorizer, IAM, JWT, mTLS, WAF      │
│  Deploy: stages, canary, custom domains, versioning          │
│  Monitoring: CloudWatch, X-Ray, access logs, troubleshooting │
│  Patterns: composition, BFF, multi-region, event-driven      │
│  Expert: SaaS multi-tenant, strangler fig, marketplace       │
│                                                               │
│  De zero a referência técnica em API Gateway.                │
└──────────────────────────────────────────────────────────────┘
```

### Próximos Workshops

```
Workshops complementares neste repo:
├── cloudfront/     → CDN + API Gateway (edge-optimized, caching)
├── aws-security/   → Segurança de APIs (WAF, IAM, Cognito)
└── (futuro) serverless/ → Lambda + API GW + DynamoDB deep dive
```

**Próximo:** Voltar ao [README principal](../README.md) e explorar os outros workshops.
