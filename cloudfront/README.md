# Amazon CloudFront — Workshop: De Zero a Especialista

> **Level:** 100 → 200 → 300 → 400
> **Tipo:** Hands-on Workshop
> **Duração Total:** ~80-100 horas de labs práticos
> **Custo Estimado:** ~$15-50 (maioria no Free Tier)
> **Última Atualização:** Março 2026

---

## Sobre Este Workshop

Este workshop contém **60 desafios práticos progressivos** organizados em **10 módulos** que cobrem **100% das funcionalidades do Amazon CloudFront** — desde a criação da primeira distribution até arquiteturas multi-tenant enterprise servindo milhões de requests.

Inspirado no formato dos **AWS Workshops (Advanced 300/400)**, cada desafio inclui:

- Objetivo claro e contexto real de produção
- Diagramas de arquitetura ASCII
- Passo a passo completo com **AWS CLI** e **Terraform**
- Comandos de validação (`curl`, CloudWatch, Athena)
- Tabela "O Que Aprendemos" para fixação
- Expert Tips de quem opera CloudFront em produção
- Estimativa de tempo e custo por desafio

```
         ┌─────────────────────────────────────────────────────────────────┐
         │                                                                 │
         │        AMAZON CLOUDFRONT — WORKSHOP ESPECIALISTA                │
         │                                                                 │
         │   "De zero a referência técnica em CloudFront"                  │
         │                                                                 │
         │   60 Desafios  ·  10 Módulos  ·  4 Níveis  ·  ~100h Labs      │
         │                                                                 │
         └─────────────────────────────────────────────────────────────────┘
```

---

## Mapa de Progressão

```
  LEVEL 100                LEVEL 200                 LEVEL 300                  LEVEL 400
  (Foundational)           (Intermediate)            (Advanced)                 (Expert)
  ──────────────           ──────────────            ──────────────             ──────────────

  ┌──────────┐             ┌──────────┐              ┌──────────┐              ┌──────────┐
  │Módulo 01 │             │Módulo 03 │              │Módulo 05 │              │Módulo 08 │
  │Fundament.│────────────→│Policies &│─────────────→│Segurança │─────────────→│Multitenant│
  │D.1 — D.5 │             │Cache     │              │Avançada  │              │SaaS       │
  │ 8-10h    │             │D.11—D.15 │              │D.23—D.30 │              │D.45—D.50  │
  └──────────┘             │ 8-10h    │              │ 12-16h   │              │ 12-16h    │
                           └──────────┘              └──────────┘              └──────────┘
  ┌──────────┐             ┌──────────┐              ┌──────────┐              ┌──────────┐
  │Módulo 02 │             │Módulo 04 │              │Módulo 06 │              │Módulo 09 │
  │Origins & │────────────→│Functions │─────────────→│Integraçõ.│─────────────→│Performanc│
  │Behaviors │             │Lambda@E  │              │AWS       │              │Savings    │
  │D.6—D.10  │             │D.16—D.22 │              │D.31—D.38 │              │D.51—D.55  │
  │ 8-10h    │             │ 10-14h   │              │ 10-14h   │              │ 10-14h    │
  └──────────┘             └──────────┘              └──────────┘              └──────────┘
                                                     ┌──────────┐              ┌──────────┐
                                                     │Módulo 07 │              │Módulo 10 │
                                                     │Telemetria│─────────────→│Cenários  │
                                                     │Monitoring│              │Expert    │
                                                     │D.39—D.44 │              │D.56—D.60  │
                                                     │ 8-10h    │              │ 12-16h    │
                                                     └──────────┘              └──────────┘
```

---

## Estrutura dos Módulos

### Level 100 — Foundational (Módulos 01-02)

> *Você é novo em CloudFront. Quer entender os conceitos fundamentais, criar sua primeira distribution e aprender a configurar origins e behaviors.*

| # | Módulo | Desafios | Tempo | O Que Você Vai Aprender |
|---|--------|----------|-------|------------------------|
| 01 | [**Fundamentos CloudFront**](modulo-01-fundamentos.md) | 1-5 | 8-10h | Primeira distribution com S3+OAC, anatomia do console (todas as tabs), HTTPS com ACM+Route53, invalidações e cache busting, Terraform IaC completo |
| 02 | [**Origins e Behaviors**](modulo-02-origins-behaviors.md) | 6-10 | 8-10h | Múltiplos origins (S3+ALB), Origin Groups com failover automático, 8 path patterns com cache policies distintas, Blue/Green com origin path, VPC Origins (PrivateLink) |

**Ao completar Level 100, você sabe:**
- Criar e configurar CloudFront distributions
- Usar S3, ALB e origins customizados
- Configurar behaviors com path patterns
- Usar HTTPS com domínio próprio
- Implementar failover com Origin Groups
- Escrever Terraform para toda a infraestrutura

---

### Level 200 — Intermediate (Módulos 03-04)

> *Você já usa CloudFront. Quer dominar cache policies, otimizar performance e executar código no edge.*

| # | Módulo | Desafios | Tempo | O Que Você Vai Aprender |
|---|--------|----------|-------|------------------------|
| 03 | [**Policies e Cache**](modulo-03-policies-cache.md) | 11-15 | 8-10h | Todas as Managed Cache Policies, custom Cache/ORP/RHP, security headers (HSTS, CSP, X-Frame), CORS, 15 técnicas para cache hit ratio 95%+ |
| 04 | [**Functions e Lambda@Edge**](modulo-04-functions-lambda-edge.md) | 16-22 | 10-14h | CF Functions (URL rewrite, A/B test, geo redirect), KeyValueStore, Lambda@Edge (JWT auth, image resize, SSR), arquitetura combinada CF Functions + Lambda@Edge |

**Ao completar Level 200, você sabe:**
- Criar cache policies otimizadas para cada cenário
- Configurar response headers de segurança
- Maximizar cache hit ratio (90%+)
- Executar lógica no edge com CloudFront Functions
- Usar KeyValueStore para feature flags
- Autenticar requests com JWT via Lambda@Edge
- Redimensionar imagens on-the-fly
- Implementar SSR com meta tags dinâmicas

---

### Level 300 — Advanced (Módulos 05-07)

> *Você é proficiente em CloudFront. Quer dominar segurança enterprise, integrar com todos os serviços AWS e implementar observabilidade completa.*

| # | Módulo | Desafios | Tempo | O Que Você Vai Aprender |
|---|--------|----------|-------|------------------------|
| 05 | [**Segurança Avançada**](modulo-05-seguranca-avancada.md) | 23-30 | 12-16h | Signed URLs e Cookies (Canned vs Custom Policy), WAF completo (10 regras), Shield Standard vs Advanced, Geo Restriction (CF vs WAF), Field-Level Encryption (PCI DSS), OAC vs OAI deep dive, mTLS com Trust Stores |
| 06 | [**Integrações AWS**](modulo-06-integracoes-aws.md) | 31-38 | 10-14h | S3 (5 modos), ALB/NLB (proteção 3 camadas), API Gateway (Regional vs Edge), Lambda Function URLs+OAC, EC2 direto, Streaming (MediaPackage/MediaConvert), CloudFront vs Global Accelerator, ECS Fargate completo |
| 07 | [**Telemetria e Monitoring**](modulo-07-telemetria-monitoring.md) | 39-44 | 8-10h | Standard Logs (33 campos, Athena com partition projection, 12 queries SQL), Real-Time Logs (Kinesis+Lambda), CloudWatch (todas as métricas, dashboard Terraform), 7 alarmes + anomaly detection, Console Reports, observabilidade end-to-end + 4 runbooks |

**Ao completar Level 300, você sabe:**
- Proteger conteúdo premium com signed URLs/cookies
- Configurar WAF com 10+ regras (SQLi, XSS, bots, rate limiting)
- Integrar CloudFront com QUALQUER serviço AWS como origin
- Implementar streaming live e VOD
- Analisar logs com Athena (queries SQL prontas)
- Criar dashboards e alarmes de produção
- Diagnosticar problemas com runbooks estruturados

---

### Level 400 — Expert (Módulos 08-10)

> *Você opera CloudFront em produção. Quer dominar arquiteturas multi-tenant, otimizar custos, automatizar operações Day-2 e construir sistemas enterprise que servem milhões de requests.*

| # | Módulo | Desafios | Tempo | O Que Você Vai Aprender |
|---|--------|----------|-------|------------------------|
| 08 | [**Multi-tenant, SaaS e Features Avançadas**](modulo-08-multitenant-saas.md) | 45-50 | 12-16h | Distribution Tenants (5.000 tenants), arquitetura SaaS com isolamento, Continuous Deployment (blue/green, canary), Static IPs, WebSocket, HTTP/2 e HTTP/3 (QUIC) |
| 09 | [**Performance, Savings e Operações**](modulo-09-performance-savings.md) | 51-55 | 10-14h | Savings Bundle (cálculo de economia), Precificação deep dive (11 componentes), AWS RAM (compartilhamento cross-account), Performance tuning expert (8 técnicas, 95%+ hit ratio), Operações Day-2 (5 runbooks, incident matrix, automação) |
| 10 | [**Cenários Reais de Produção**](modulo-10-cenarios-expert.md) | 56-60 | 12-16h | **CAPSTONE:** E-commerce Black Friday (100k req/s), Multi-region active-active, Migração de CDN (Akamai/Cloudflare), Plataforma de Video Streaming, Guia de certificação + próximos passos |

**Ao completar Level 400, você é:**
- Capaz de arquitetar soluções CloudFront para qualquer cenário
- Referência técnica em CDN e edge computing na AWS
- Preparado para cenários enterprise de alta escala
- Operacionalmente maduro com runbooks e automação
- Pronto para certificações AWS (SAA, SAP, Networking Specialty)

---

## Cobertura Completa do Console CloudFront

Este workshop cobre **100% dos itens do menu do CloudFront Console**:

```
CloudFront Console
├── Distributions ─────────────── Módulos 01, 02, 08
│   ├── General ──────────────── Módulo 01 (D.2)
│   ├── Origins ──────────────── Módulo 02 (D.6, D.7, D.10)
│   ├── Behaviors ────────────── Módulo 02 (D.8, D.9)
│   ├── Error Pages ──────────── Módulo 01 (D.2)
│   ├── Invalidations ────────── Módulo 01 (D.4)
│   └── Logging ──────────────── Módulo 07 (D.39)
│
├── Policies ──────────────────── Módulo 03
│   ├── Cache Policy ─────────── Módulo 03 (D.11, D.12)
│   ├── Origin Request Policy ── Módulo 03 (D.13)
│   └── Response Headers ─────── Módulo 03 (D.14)
│
├── Functions ─────────────────── Módulo 04 (D.16, D.17, D.21)
│
├── Static IPs ────────────────── Módulo 08 (D.48)
│
├── VPC Origins ───────────────── Módulo 02 (D.10)
│
├── SaaS ──────────────────────── Módulo 08
│   ├── Multi-tenant Distrib. ── Módulo 08 (D.45)
│   └── Distribution Tenants ─── Módulo 08 (D.45, D.46)
│
├── Telemetry ─────────────────── Módulo 07
│   ├── Monitoring ───────────── Módulo 07 (D.41, D.42)
│   ├── Alarms ───────────────── Módulo 07 (D.42)
│   ├── Logs ─────────────────── Módulo 07 (D.39, D.40)
│   └── Reports & Analytics
│       ├── Cache Statistics ─── Módulo 07 (D.43)
│       ├── Popular Objects ──── Módulo 07 (D.43)
│       ├── Top Referrers ────── Módulo 07 (D.43)
│       ├── Usage ────────────── Módulo 07 (D.43)
│       └── Viewers ──────────── Módulo 07 (D.43)
│
├── Security ──────────────────── Módulo 05
│   ├── WAF ──────────────────── Módulo 05 (D.25)
│   ├── Origin Access ────────── Módulo 01 (D.1), Módulo 05 (D.29)
│   ├── Trust Stores ─────────── Módulo 05 (D.30)
│   ├── Field-Level Encryption ─ Módulo 05 (D.28)
│   └── Key Management
│       ├── Public Keys ──────── Módulo 05 (D.23)
│       └── Key Groups ──────── Módulo 05 (D.23)
│
├── Savings Bundle ────────────── Módulo 09 (D.51)
│
└── Settings ──────────────────── Módulo 01 (D.2)
```

---

## Tecnologias e Ferramentas Utilizadas

| Ferramenta | Uso no Workshop |
|-----------|-----------------|
| **AWS CLI v2** | Todos os desafios — commands completos e testáveis |
| **Terraform** | Todos os desafios — IaC para cada recurso criado |
| **Python** | Lambda@Edge, calculadoras de custo, scripts de automação |
| **Node.js** | CloudFront Functions (cloudfront-js-2.0), Lambda@Edge |
| **Bash** | Runbooks operacionais, scripts de validação |
| **curl** | Testes de validação em cada desafio |
| **Amazon Athena** | Análise de Standard Logs (SQL queries prontas) |
| **CloudWatch** | Dashboards e alarmes (JSON/Terraform) |
| **OpenSSL** | mTLS, certificados, chaves RSA |
| **k6** | Load testing (Módulo 10) |

---

## Integrações AWS Cobertas

```
                            ┌─────────────────┐
                            │   CloudFront     │
                            │   (Este Workshop)│
                            └────────┬────────┘
                                     │
         ┌───────────┬───────────┬───┴───┬───────────┬───────────┐
         │           │           │       │           │           │
    ┌────┴──┐   ┌────┴──┐  ┌────┴──┐ ┌──┴────┐ ┌───┴───┐ ┌────┴──┐
    │  S3   │   │  ALB  │  │  API  │ │Lambda │ │  EC2  │ │  ECS  │
    │       │   │  NLB  │  │  GW   │ │FuncURL│ │       │ │Fargate│
    │ D.1,6 │   │ D.6,32│  │ D.33  │ │ D.34  │ │ D.35  │ │ D.38  │
    │  31   │   │  38   │  │       │ │       │ │       │ │       │
    └───────┘   └───────┘  └───────┘ └───────┘ └───────┘ └───────┘

    ┌───────┐   ┌───────┐  ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
    │  ACM  │   │Route53│  │  WAF  │ │Shield │ │Kinesis│ │  SNS  │
    │       │   │       │  │       │ │       │ │  DS   │ │       │
    │ D.3   │   │ D.3   │  │ D.25  │ │ D.26  │ │ D.40  │ │ D.42  │
    └───────┘   └───────┘  └───────┘ └───────┘ └───────┘ └───────┘

    ┌───────┐   ┌───────┐  ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
    │ Cloud │   │Athena │  │  RAM  │ │Media  │ │Media  │ │Global │
    │ Watch │   │       │  │       │ │Package│ │Convert│ │Accel. │
    │ D.41  │   │ D.39  │  │ D.53  │ │ D.36  │ │ D.36  │ │ D.37  │
    └───────┘   └───────┘  └───────┘ └───────┘ └───────┘ └───────┘
```

---

## Pré-requisitos

### Obrigatórios

- **Conta AWS** — Free Tier é suficiente para a maioria dos desafios
- **AWS CLI v2** — Instalada e configurada (`aws configure`)
- **Terraform >= 1.5** — Para todos os desafios de IaC
- **Conhecimentos básicos** — HTTP/HTTPS, DNS, redes TCP/IP

### Recomendados

- **Domínio próprio** — Para desafios com HTTPS/ACM (Módulo 01+)
- **Node.js 18+** — Para Lambda@Edge e CloudFront Functions
- **Python 3.12+** — Para scripts e Lambda consumers
- **Editor com suporte HCL** — VS Code com extensão Terraform

### Custos Estimados

| Level | Custo Estimado | Nota |
|-------|---------------|------|
| 100 (Módulos 01-02) | ~$0-2 | Maioria no Free Tier |
| 200 (Módulos 03-04) | ~$2-5 | Lambda@Edge tem custo mínimo |
| 300 (Módulos 05-07) | ~$5-15 | WAF, Kinesis, Athena |
| 400 (Módulos 08-10) | ~$10-30 | Multi-tenant, load testing |

> **Importante:** Sempre execute o cleanup ao final de cada desafio para evitar cobranças inesperadas. Cada módulo inclui instruções de cleanup.

---

## Navegação Rápida

### Level 100 — Foundational
- [Módulo 01 — Fundamentos CloudFront](modulo-01-fundamentos.md) (Desafios 1-5)
- [Módulo 02 — Origins e Behaviors](modulo-02-origins-behaviors.md) (Desafios 6-10)

### Level 200 — Intermediate
- [Módulo 03 — Policies e Cache](modulo-03-policies-cache.md) (Desafios 11-15)
- [Módulo 04 — Functions e Lambda@Edge](modulo-04-functions-lambda-edge.md) (Desafios 16-22)

### Level 300 — Advanced
- [Módulo 05 — Segurança Avançada](modulo-05-seguranca-avancada.md) (Desafios 23-30)
- [Módulo 06 — Integrações AWS](modulo-06-integracoes-aws.md) (Desafios 31-38)
- [Módulo 07 — Telemetria e Monitoring](modulo-07-telemetria-monitoring.md) (Desafios 39-44)

### Level 400 — Expert
- [Módulo 08 — Multi-tenant, SaaS e Features Avançadas](modulo-08-multitenant-saas.md) (Desafios 45-50)
- [Módulo 09 — Performance, Savings e Operações](modulo-09-performance-savings.md) (Desafios 51-55)
- [Módulo 10 — Cenários Reais de Produção](modulo-10-cenarios-expert.md) (Desafios 56-60)

---

## Índice Completo de Desafios

### Módulo 01 — Fundamentos (Level 100)
| # | Desafio | Tempo | Serviços |
|---|---------|-------|----------|
| 1 | Primeira Distribution com S3 + OAC | 60 min | CloudFront, S3 |
| 2 | Anatomia do Console — Todas as Tabs | 45 min | CloudFront |
| 3 | Domínio Customizado com HTTPS | 90 min | CloudFront, ACM, Route 53 |
| 4 | Invalidações e Cache Busting | 45 min | CloudFront |
| 5 | Infraestrutura Completa com Terraform | 90 min | CloudFront, S3, ACM, Route 53 |

### Módulo 02 — Origins e Behaviors (Level 100-200)
| # | Desafio | Tempo | Serviços |
|---|---------|-------|----------|
| 6 | Múltiplos Origins: S3 + ALB | 90 min | CloudFront, S3, ALB |
| 7 | Origin Groups com Failover Automático | 60 min | CloudFront, S3 |
| 8 | Behaviors Complexos com 8 Path Patterns | 90 min | CloudFront |
| 9 | Origin Path e Blue/Green Deployments | 60 min | CloudFront, S3 |
| 10 | VPC Origins com PrivateLink | 90 min | CloudFront, VPC, ALB |

### Módulo 03 — Policies e Cache (Level 200)
| # | Desafio | Tempo | Serviços |
|---|---------|-------|----------|
| 11 | Todas as Managed Cache Policies | 60 min | CloudFront |
| 12 | Custom Cache Policies (E-commerce, API, i18n) | 90 min | CloudFront |
| 13 | Origin Request Policies | 60 min | CloudFront |
| 14 | Response Headers Policies (Security, CORS) | 90 min | CloudFront |
| 15 | Cache Hit Ratio — 15 Técnicas de Otimização | 120 min | CloudFront, CloudWatch |

### Módulo 04 — Functions e Lambda@Edge (Level 200-300)
| # | Desafio | Tempo | Serviços |
|---|---------|-------|----------|
| 16 | CF Functions: URL Rewrites (SPA, Geo, WWW) | 90 min | CloudFront Functions |
| 17 | CF Functions: A/B Testing com KeyValueStore | 90 min | CloudFront Functions, KVS |
| 18 | Lambda@Edge: Autenticação JWT | 120 min | Lambda@Edge |
| 19 | Lambda@Edge: Resize de Imagens On-the-Fly | 120 min | Lambda@Edge, S3 |
| 20 | Lambda@Edge: SSR com Open Graph | 90 min | Lambda@Edge |
| 21 | CF Functions: Security Headers e API Key com KVS | 60 min | CloudFront Functions, KVS |
| 22 | Arquitetura Combinada: CF Functions + Lambda@Edge | 90 min | CF Functions, Lambda@Edge |

### Módulo 05 — Segurança Avançada (Level 300)
| # | Desafio | Tempo | Serviços |
|---|---------|-------|----------|
| 23 | Signed URLs (RSA, Key Groups, Canned vs Custom) | 120 min | CloudFront, Key Management |
| 24 | Signed Cookies (Wildcard, Express.js Backend) | 90 min | CloudFront |
| 25 | WAF Completo (10 Regras: SQLi, XSS, Bots, Rate Limit) | 120 min | WAF, CloudFront |
| 26 | Shield Standard vs Advanced | 60 min | Shield, CloudFront |
| 27 | Geo Restriction (CloudFront vs WAF) | 60 min | CloudFront, WAF |
| 28 | Field-Level Encryption (PCI DSS) | 90 min | CloudFront |
| 29 | OAC vs OAI — Deep Dive e Migração | 60 min | CloudFront, S3 |
| 30 | mTLS com Trust Stores | 120 min | CloudFront, ACM |

### Módulo 06 — Integrações AWS (Level 300)
| # | Desafio | Tempo | Serviços |
|---|---------|-------|----------|
| 31 | S3: Todos os Modos (OAC, Website, Upload, SSE-KMS) | 90 min | S3, CloudFront, KMS |
| 32 | ALB/NLB: Proteção 3 Camadas | 90 min | ALB, NLB, CloudFront, WAF |
| 33 | API Gateway: Regional vs Edge, REST vs HTTP | 90 min | API Gateway, CloudFront |
| 34 | Lambda Function URLs com OAC | 60 min | Lambda, CloudFront |
| 35 | EC2 Direto com Security Group | 60 min | EC2, CloudFront |
| 36 | Streaming: Live (MediaPackage) + VOD (MediaConvert) | 120 min | MediaPackage, MediaConvert |
| 37 | CloudFront vs Global Accelerator (15 Critérios) | 60 min | CloudFront, Global Accelerator |
| 38 | ECS Fargate + ALB + CloudFront | 120 min | ECS, ALB, CloudFront |

### Módulo 07 — Telemetria e Monitoring (Level 300)
| # | Desafio | Tempo | Serviços |
|---|---------|-------|----------|
| 39 | Standard Logs (33 Campos, Athena, 12 Queries SQL) | 120 min | S3, Athena, Glue |
| 40 | Real-Time Logs (Kinesis + Lambda Consumer) | 90 min | Kinesis Data Streams, Lambda |
| 41 | CloudWatch Metrics e Dashboard Completo | 90 min | CloudWatch |
| 42 | 7 Alarmes + Anomaly Detection | 60 min | CloudWatch, SNS |
| 43 | Console Reports & Analytics | 45 min | CloudFront Console |
| 44 | Observabilidade End-to-End + 4 Runbooks | 120 min | Todos |

### Módulo 08 — Multi-tenant e SaaS (Level 400)
| # | Desafio | Tempo | Serviços |
|---|---------|-------|----------|
| 45 | Multi-tenant Distributions (5.000 Tenants) | 90 min | CloudFront |
| 46 | Arquitetura SaaS com Isolamento por Tenant | 120 min | CloudFront, Lambda@Edge |
| 47 | Continuous Deployment: Blue/Green e Canary | 120 min | CloudFront |
| 48 | Static IPs para Firewall Whitelisting | 60 min | CloudFront |
| 49 | WebSocket via CloudFront | 60 min | CloudFront, ALB |
| 50 | HTTP/2 e HTTP/3 (QUIC) — Performance Moderna | 90 min | CloudFront |

### Módulo 09 — Performance e Savings (Level 400)
| # | Desafio | Tempo | Serviços |
|---|---------|-------|----------|
| 51 | Savings Bundle — Compromisso e Economia | 60 min | CloudFront, Savings Plans |
| 52 | Precificação Deep Dive (11 Componentes) | 90 min | Cost Explorer |
| 53 | AWS RAM — Compartilhamento Cross-Account | 60 min | RAM, CloudFront |
| 54 | Performance Tuning Expert (8 Técnicas, 95%+ HR) | 120 min | CloudFront, CloudWatch |
| 55 | Operações Day-2 e Runbooks de Produção | 120 min | CloudFront, EventBridge |

### Módulo 10 — Cenários Expert (Level 400)
| # | Desafio | Tempo | Serviços |
|---|---------|-------|----------|
| 56 | **CAPSTONE:** E-commerce Black Friday (100k req/s) | 180 min | Todos |
| 57 | Multi-Region Active-Active | 120 min | CloudFront, Route 53, DynamoDB |
| 58 | Migração de CDN (Akamai/Cloudflare → CloudFront) | 90 min | CloudFront |
| 59 | Plataforma de Video Streaming | 120 min | MediaPackage, MediaConvert, S3 |
| 60 | Certificação e Próximos Passos | 60 min | — |

---

## Como Usar Este Workshop

### Estrutura de Cada Desafio

```
┌─────────────────────────────────────────────────────┐
│  DESAFIO N: Nome do Desafio                         │
│                                                      │
│  > Level: XXX | Tempo: XX min | Custo: ~$X          │
│                                                      │
│  ┌────────────┐                                      │
│  │ OBJETIVO   │ O que você vai construir             │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ CONTEXTO   │ Por que isso importa na vida real    │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ARQUITETURA │ Diagrama ASCII da solução            │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ PASSO A    │ AWS CLI commands                     │
│  │ PASSO      │ Terraform code                       │
│  │            │ Código (Python/Node.js)              │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ VALIDAÇÃO  │ curl commands para testar            │
│  │            │ CloudWatch metrics para verificar    │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ O QUE      │ Tabela de conceitos aprendidos       │
│  │ APRENDEMOS │                                      │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ EXPERT TIP │ Insight de produção real             │
│  └────────────┘                                      │
└─────────────────────────────────────────────────────┘
```

### Recomendações

1. **Siga a ordem** — Os módulos são progressivos. Cada um assume conhecimento dos anteriores.
2. **Execute tudo** — Ler não é suficiente. Crie os recursos, teste com curl, veja os logs.
3. **Use Terraform** — Todo desafio tem versão CLI e Terraform. Pratique ambos.
4. **Faça cleanup** — Destrua recursos após cada desafio para evitar custos.
5. **Anote dúvidas** — Se algo não ficou claro, releia o contexto e a tabela "O Que Aprendemos".
6. **Experimente variações** — Após completar um desafio, tente variações por conta própria.

---

## Referências

- [Documentação Oficial CloudFront](https://docs.aws.amazon.com/cloudfront/)
- [CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
- [CloudFront API Reference](https://docs.aws.amazon.com/cloudfront/latest/APIReference/)
- [Terraform AWS CloudFront](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudfront_distribution)
- [AWS Well-Architected — Performance Efficiency](https://docs.aws.amazon.com/wellarchitected/latest/performance-efficiency-pillar/)
- [CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/)

---

> **Workshop criado para transformar você em referência técnica em Amazon CloudFront.
> 60 desafios. 10 módulos. Do zero ao expert.**
