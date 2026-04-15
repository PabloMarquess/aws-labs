# Módulo 10 — Cenários Expert (Grand Finale)

> **Nível:** 400 (Expert)
> **Tempo Total Estimado:** 12-16 horas de labs
> **Custo Estimado:** ~$5-15
> **Objetivo do Módulo:** Aplicar tudo dos módulos 01-09 em cenários reais — SaaS platform auth completa, B2B multi-org SAML, compliance (HIPAA/PCI), monitoring e audit, comparativo Cognito vs Auth0/Firebase e preparação para certificação.

---

## Mapa do Módulo Final

```mermaid
graph TB
    MOD[Módulo 10<br/>Grand Finale]

    MOD --> D55[D.55 CAPSTONE<br/>SaaS Auth Platform]
    MOD --> D56_C[D.56 B2B Multi-Org<br/>SAML Federation]
    MOD --> D57_C[D.57 Compliance<br/>HIPAA / PCI / LGPD]
    MOD --> D58_C[D.58 Monitoring<br/>& Audit Logging]
    MOD --> D59_C[D.59 Cognito vs<br/>Auth0 / Firebase]
    MOD --> D60_C[D.60 Certificação<br/>Próximos Passos]

    style MOD fill:#e94560,color:#fff
    style D55 fill:#0f3460,color:#fff
    style D56_C fill:#533483,color:#fff
    style D59_C fill:#16213e,color:#fff
```

---

## Desafio 55: CAPSTONE — SaaS Auth Platform

> **Level:** 400 | **Tempo:** 180 min | **Custo:** ~$5

### Objetivo

Projetar a arquitetura de autenticação completa para uma plataforma SaaS multi-tenant.

### Arquitetura

```mermaid
graph TB
    subgraph Tenants
        T1[Tenant A<br/>Free Plan]
        T2[Tenant B<br/>Pro Plan<br/>+ Google SSO]
        T3[Tenant C<br/>Enterprise<br/>+ SAML Okta]
    end

    subgraph Auth ["Cognito Auth Layer"]
        UP[User Pool<br/>Shared]
        GOOGLE[Google IdP]
        SAML[SAML IdP<br/>Okta]
        TRIGGERS[Lambda Triggers<br/>Pre-Token: tenant claims<br/>Post-Confirm: onboarding]
    end

    subgraph API ["API Layer"]
        APIGW[API Gateway<br/>JWT Authorizer]
        LAMBDA[Lambda<br/>Tenant-aware]
    end

    subgraph Data ["Data Layer"]
        DDB[(DynamoDB<br/>PK: TENANT#id)]
        S3[S3<br/>tenant-id/files/]
    end

    T1 --> UP
    T2 --> GOOGLE --> UP
    T3 --> SAML --> UP
    UP --> TRIGGERS
    TRIGGERS --> APIGW
    APIGW --> LAMBDA
    LAMBDA --> DDB
    LAMBDA --> S3

    style UP fill:#0f3460,color:#fff
    style APIGW fill:#533483,color:#fff
    style DDB fill:#16213e,color:#fff
```

### Stack Completo

```
SaaS Auth Stack:
├── Cognito User Pool (shared)
│   ├── Custom attributes: tenant_id, plan, role
│   ├── Groups: admins, editors, viewers (per tenant)
│   ├── MFA: OPTIONAL (Free), ON (Enterprise)
│   ├── Advanced Security: ENFORCED
│   └── Managed Login: custom domain + branding
│
├── Identity Providers
│   ├── Google (social login — all tenants)
│   ├── SAML IdP per enterprise tenant
│   └── Attribute mapping: tenant → custom:tenant_id
│
├── Lambda Triggers
│   ├── Pre-Signup: validate domain, assign tenant
│   ├── Post-Confirmation: create profile, onboarding
│   ├── Pre-Token: inject tenant_id, plan, permissions
│   └── Custom Auth: passwordless for invited users
│
├── API Gateway (HTTP API)
│   ├── JWT Authorizer (Cognito)
│   ├── Routes per service
│   └── Throttling per tenant (via API key)
│
├── App Clients
│   ├── SPA client (public, PKCE)
│   ├── Mobile client (public, PKCE)
│   ├── Server client (confidential, client_credentials)
│   └── Per-tenant SAML clients (enterprise)
│
└── Monitoring
    ├── CloudTrail: auth events
    ├── CloudWatch: login metrics, errors
    └── Advanced security logs: risk events
```

---

## Desafio 59: Cognito vs Auth0 vs Firebase Auth

> **Level:** 400 | **Tempo:** 60 min | **Custo:** $0

### Comparativo

| Aspecto | Cognito | Auth0 | Firebase Auth |
|---------|---------|-------|---------------|
| **Preço (50K MAU)** | $0 (Lite) | ~$240/mês | $0 |
| **Preço (500K MAU)** | ~$7.500 (Lite) | Enterprise | $0 (limits) |
| **AWS integration** | Nativa (IAM, API GW, ALB) | Via JWT | Via JWT |
| **Social login** | Google, Facebook, Apple, Amazon | 30+ providers | Google, Facebook, Apple, etc. |
| **SAML/OIDC** | Sim | Sim | Não nativo |
| **Machine-to-Machine** | Client Credentials | Sim | Não |
| **Passkeys** | Sim (Essentials+) | Sim | Sim |
| **Custom domain** | Sim | Sim | Não |
| **Lambda triggers** | 7+ triggers | Actions/Rules | Cloud Functions |
| **Managed login** | Sim (customizável) | Sim (Universal Login) | Sim (FirebaseUI) |
| **Multi-tenant** | Groups + custom attrs | Organizations | Não nativo |
| **Vendor lock-in** | AWS ecosystem | Independente | Google ecosystem |
| **Self-hosted** | Não | Não | Não |

### Decision Framework

```
Use Cognito quando:
├── Já está no ecossistema AWS (API GW, ALB, S3, DynamoDB)
├── Precisa de Identity Pools (AWS credentials temporárias)
├── Budget é prioridade (50K MAU grátis!)
├── SAML federation para B2B
└── Machine-to-Machine (client_credentials)

Use Auth0 quando:
├── Multi-cloud ou cloud-agnostic
├── Precisa de 30+ social providers
├── UX do login é diferencial (Universal Login é excelente)
├── Time não quer gerenciar Lambda triggers
└── Enterprise features OOTB (Organizations, Branding)

Use Firebase Auth quando:
├── App mobile-first com Google ecosystem
├── Simplicidade é prioridade
├── Budget zero (gratuito até limites altos)
├── Não precisa de SAML/M2M
└── Já usa Firestore/Cloud Functions
```

---

## Desafio 60: Certificação e Próximos Passos

> **Level:** 400 | **Tempo:** 60 min | **Custo:** $0

### O Que Este Workshop Cobriu

```
┌──────────────────────────────────────────────────────────────┐
│               COGNITO WORKSHOP — COMPLETO                     │
│                                                               │
│  60 desafios · 10 módulos · Level 100 → 400                 │
│                                                               │
│  User Pools: signup, login, MFA, tokens, groups              │
│  Identity Pools: AWS credentials, role mapping               │
│  OAuth2/OIDC: Auth Code + PKCE, Client Credentials           │
│  Lambda Triggers: 7 triggers, passwordless, migration        │
│  Security: Adaptive auth, WAF, threat protection             │
│  Integration: API GW, ALB, Amplify, React                    │
│  Federation: Google, Facebook, SAML, OIDC                    │
│  Patterns: Multi-tenant, passkeys, step-up auth             │
│  Expert: SaaS platform, B2B SAML, compliance                │
│                                                               │
│  De zero a referência técnica em Amazon Cognito.             │
└──────────────────────────────────────────────────────────────┘
```

### Próximos Workshops

```
Workshops complementares neste repo:
├── cloudfront/     → CDN (integração com Cognito signed URLs)
├── aws-security/   → Security (IAM, GuardDuty, Config)
├── api-gateway/    → APIs (Cognito Authorizer, JWT)
└── (futuro) serverless/ → Lambda + API GW + DynamoDB + Cognito
```

**Próximo:** Voltar ao [README principal](../README.md) e explorar os outros workshops.
