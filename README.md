# AWS Labs

Workshops hands-on no estilo AWS (Level 100→400) com desafios progressivos para se tornar especialista em serviços AWS.

Cada workshop contém dezenas de desafios práticos com AWS CLI, Terraform, diagramas de arquitetura, testes de validação e dicas de produção.

---

## Workshops

| Workshop | Desafios | Módulos | Status |
|----------|----------|---------|--------|
| [**CloudFront**](cloudfront/) | 60 | 10 | Completo |
| [**AWS Security**](aws-security/) | 70 | 10 | Completo |
| [**API Gateway**](api-gateway/) | 65 | 10 | Completo |
| [**Cognito**](cognito/) | 60 | 10 | Em andamento |

---

## CloudFront

Workshop completo cobrindo 100% das funcionalidades do Amazon CloudFront — de fundamentos até arquiteturas multi-tenant enterprise.

**Módulos:**

| # | Módulo | Desafios | Nível |
|---|--------|----------|-------|
| 01 | [Fundamentos](cloudfront/modulo-01-fundamentos.md) | 1-5 | 100 |
| 02 | [Origins e Behaviors](cloudfront/modulo-02-origins-behaviors.md) | 6-10 | 100-200 |
| 03 | [Policies e Cache](cloudfront/modulo-03-policies-cache.md) | 11-15 | 200 |
| 04 | [Functions e Lambda@Edge](cloudfront/modulo-04-functions-lambda-edge.md) | 16-22 | 200-300 |
| 05 | [Segurança Avançada](cloudfront/modulo-05-seguranca-avancada.md) | 23-30 | 300 |
| 06 | [Integrações AWS](cloudfront/modulo-06-integracoes-aws.md) | 31-38 | 300 |
| 07 | [Telemetria e Monitoring](cloudfront/modulo-07-telemetria-monitoring.md) | 39-44 | 300 |
| 08 | [Multi-tenant e SaaS](cloudfront/modulo-08-multitenant-saas.md) | 45-50 | 400 |
| 09 | [Performance e Savings](cloudfront/modulo-09-performance-savings.md) | 51-55 | 400 |
| 10 | [Cenários Expert](cloudfront/modulo-10-cenarios-expert.md) | 56-60 | 400 |

---

## AWS Security

Workshop cobrindo todo o ecossistema de segurança da AWS — IAM, GuardDuty, CloudTrail, WAF, KMS, Security Hub, Incident Response e mais.

**Módulos:**

| # | Módulo | Desafios | Nível | Status |
|---|--------|----------|-------|--------|
| 01 | [IAM e Identity](aws-security/modulo-01-iam-identity.md) | 1-7 | 100 | Completo |
| 02 | [Detection e Monitoring](aws-security/modulo-02-detection-monitoring.md) | 8-14 | 100-200 | Completo |
| 03 | [Infrastructure Protection](aws-security/modulo-03-infrastructure-protection.md) | 15-21 | 200 | Completo |
| 04 | [Data Protection](aws-security/modulo-04-data-protection.md) | 22-28 | 200-300 | Completo |
| 05 | [Incident Response](aws-security/modulo-05-incident-response.md) | 29-35 | 300 | Completo |
| 06 | [Compliance e Governance](aws-security/modulo-06-compliance-governance.md) | 36-42 | 300 | Completo |
| 07 | [Network Security](aws-security/modulo-07-network-security.md) | 43-49 | 300 | Completo |
| 08 | [Application Security](aws-security/modulo-08-application-security.md) | 50-55 | 400 | Completo |
| 09 | [Security Operations](aws-security/modulo-09-secops.md) | 56-62 | 400 | Completo |
| 10 | [Cenários Expert](aws-security/modulo-10-cenarios-expert.md) | 63-70 | 400 | Completo |
| Extra | [Permissions, Governance & AI Controls](aws-security/modulo-extra-permissions-governance-ai.md) | E.1-E.6 | 200-300 | Completo |

---

## Como Usar

1. Escolha um workshop
2. Comece pelo Módulo 01 (Level 100)
3. Siga a ordem — cada módulo assume conhecimento dos anteriores
4. Execute todos os comandos — ler não é suficiente
5. Faça cleanup após cada desafio para evitar custos

## Pré-requisitos

- Conta AWS (Free Tier é suficiente para a maioria)
- AWS CLI v2 configurada
- Terraform >= 1.5
- Conhecimentos básicos de redes e HTTP
