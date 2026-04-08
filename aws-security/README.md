# AWS Security — Workshop: De Zero a Especialista

> **Level:** 100 → 200 → 300 → 400
> **Tipo:** Hands-on Workshop
> **Durao Total Estimada:** ~120-150 horas de labs prticos
> **Custo Estimado:** ~$20-80 (maioria no Free Tier)
> **ltima Atualizao:** Abril 2026

---

## Sobre Este Workshop

Este workshop contm **70 desafios prticos progressivos** organizados em **10 mdulos** que cobrem **todo o ecossistema de segurana da AWS** — desde IAM bsico at arquiteturas de segurana enterprise com deteco automatizada de ameaas, resposta a incidentes e compliance contnua.

Inspirado no formato dos **AWS Workshops (Advanced 300/400)**, cada desafio inclui:

- Objetivo claro e cenrio real de produo
- Diagramas de arquitetura ASCII
- Passo a passo completo com **AWS CLI** e **Terraform**
- Comandos de validao e testes prticos
- Tabela "O Que Aprendemos" para fixao
- Expert Tips de quem opera segurana em produo
- Estimativa de tempo e custo por desafio

```
         ┌─────────────────────────────────────────────────────────────────┐
         │                                                                 │
         │        AWS SECURITY — WORKSHOP ESPECIALISTA                     │
         │                                                                 │
         │   "De zero a referncia tcnica em segurana AWS"               │
         │                                                                 │
         │   70 Desafios  ·  10 Mdulos  ·  4 Nveis  ·  ~130h Labs      │
         │                                                                 │
         └─────────────────────────────────────────────────────────────────┘
```

---

## Servios AWS Cobertos

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    AWS Security Services — Cobertura                      │
│                                                                           │
│  IDENTITY & ACCESS                    DETECTION & MONITORING             │
│  ├── IAM (Users, Roles, Policies)     ├── GuardDuty                      │
│  ├── IAM Identity Center (SSO)        ├── CloudTrail                     │
│  ├── IAM Access Analyzer              ├── Security Hub                   │
│  ├── STS (Temporary Credentials)      ├── CloudWatch (Security)          │
│  ├── Organizations + SCPs             ├── EventBridge (Security Events)  │
│  └── Resource Access Manager (RAM)    ├── Detective                      │
│                                       └── Inspector                      │
│  INFRASTRUCTURE PROTECTION                                               │
│  ├── WAF (Web Application Firewall)   DATA PROTECTION                   │
│  ├── Shield (DDoS)                    ├── KMS (Key Management)           │
│  ├── Firewall Manager                 ├── CloudHSM                       │
│  ├── Network Firewall                 ├── Secrets Manager                │
│  ├── Security Groups / NACLs          ├── Certificate Manager (ACM)      │
│  ├── VPC (PrivateLink, Endpoints)     ├── Macie                          │
│  └── Route 53 Resolver (DNS FW)       └── S3 (Encryption, Block Public) │
│                                                                           │
│  COMPLIANCE & GOVERNANCE              INCIDENT RESPONSE                  │
│  ├── AWS Config                       ├── Systems Manager (SSM)          │
│  ├── Audit Manager                    ├── Step Functions (Playbooks)     │
│  ├── Control Tower                    ├── Lambda (Automao)              │
│  ├── Service Catalog                  ├── SNS/SQS (Alerting)            │
│  └── License Manager                  └── Forensics (EBS Snapshots)     │
│                                                                           │
│  APPLICATION SECURITY                 NETWORK SECURITY                   │
│  ├── Cognito (AuthN/AuthZ)            ├── VPC Flow Logs                  │
│  ├── Verified Access                  ├── Transit Gateway                │
│  ├── WAF (AppLayer)                   ├── PrivateLink                    │
│  └── CodeGuru Security                ├── VPN / Direct Connect           │
│                                       └── Global Accelerator (Security)  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Mapa de Progresso

```
  LEVEL 100                LEVEL 200                 LEVEL 300                  LEVEL 400
  (Foundational)           (Intermediate)            (Advanced)                 (Expert)
  ──────────────           ──────────────            ──────────────             ──────────────

  ┌──────────┐             ┌──────────┐              ┌──────────┐              ┌──────────┐
  │Mdulo 01 │             │Mdulo 03 │              │Mdulo 05 │              │Mdulo 08 │
  │IAM &     │────────────→│Infra     │─────────────→│Incident  │─────────────→│App       │
  │Identity  │             │Protec.   │              │Response  │              │Security  │
  │D.1 — D.7 │             │D.15—D.21 │              │D.29—D.35 │              │D.50—D.55 │
  │ 12-16h   │             │ 12-16h   │              │ 14-18h   │              │ 10-14h   │
  └──────────┘             └──────────┘              └──────────┘              └──────────┘
  ┌──────────┐             ┌──────────┐              ┌──────────┐              ┌──────────┐
  │Mdulo 02 │             │Mdulo 04 │              │Mdulo 06 │              │Mdulo 09 │
  │Detection │────────────→│Data      │─────────────→│Compliance│─────────────→│SecOps    │
  │& Monitor.│             │Protec.   │              │Governance│              │Center    │
  │D.8—D.14  │             │D.22—D.28 │              │D.36—D.42 │              │D.56—D.62 │
  │ 12-16h   │             │ 12-16h   │              │ 12-16h   │              │ 12-16h   │
  └──────────┘             └──────────┘              └──────────┘              └──────────┘
                                                     ┌──────────┐              ┌──────────┐
                                                     │Mdulo 07 │              │Mdulo 10 │
                                                     │Network   │─────────────→│Cenrios  │
                                                     │Security  │              │Expert    │
                                                     │D.43—D.49 │              │D.63—D.70 │
                                                     │ 12-16h   │              │ 14-18h   │
                                                     └──────────┘              └──────────┘
```

---

## Estrutura dos Mdulos

### Level 100 — Foundational (Mdulos 01-02)

> *Voc  novo em segurana AWS. Quer entender IAM, deteco bsica de ameaas e os fundamentos de auditoria.*

| # | Mdulo | Desafios | Tempo | O Que Voc Vai Aprender |
|---|--------|----------|-------|------------------------|
| 01 | [**IAM & Identity**](modulo-01-iam-identity.md) | 1-7 | 12-16h | Users, Groups, Roles, Policies (managed vs inline vs custom), STS e AssumeRole, IAM Access Analyzer, Permission Boundaries, IAM Identity Center (SSO), Cross-account access, MFA enforcement |
| 02 | [**Detection & Monitoring**](modulo-02-detection-monitoring.md) | 8-14 | 12-16h | CloudTrail (setup, Athena queries, integrity), GuardDuty (deteco de ameaas, auto-remediation), Security Hub (postura, findings, standards), CloudWatch Security (metric filters, alarmes), EventBridge (security events automation), Inspector (vulnerabilidades), Detective (investigao) |

**Ao completar Level 100, voc sabe:**
- Criar polticas IAM least privilege para qualquer servio
- Configurar MFA, roles cross-account e permission boundaries
- Monitorar toda atividade da conta via CloudTrail
- Detectar ameaas automaticamente com GuardDuty
- Centralizar findings no Security Hub
- Investigar incidentes com Detective

---

### Level 200 — Intermediate (Mdulos 03-04)

> *Voc j usa IAM e CloudTrail. Quer proteger infraestrutura, redes e dados em repouso/trnsito.*

| # | Mdulo | Desafios | Tempo | O Que Voc Vai Aprender |
|---|--------|----------|-------|------------------------|
| 03 | [**Infrastructure Protection**](modulo-03-infrastructure-protection.md) | 15-21 | 12-16h | WAF (rules, managed groups, custom regex, rate limiting, bot control, captcha), Shield (Standard vs Advanced, DRT, cost protection), Firewall Manager (centralizao cross-account), Network Firewall (stateful/stateless, domain filtering), Security Groups vs NACLs (defense in depth), VPC Endpoints e PrivateLink |
| 04 | [**Data Protection**](modulo-04-data-protection.md) | 22-28 | 12-16h | KMS (key types, policies, rotation, grants, multi-region), Secrets Manager (rotao automtica, Lambda integration), ACM (certificates lifecycle, private CA), Macie (PII detection em S3), S3 encryption (SSE-S3, SSE-KMS, SSE-C, bucket keys), S3 Block Public Access e Object Lock, CloudHSM (FIPS 140-2 Level 3) |

**Ao completar Level 200, voc sabe:**
- Configurar WAF completo com 10+ tipos de regra
- Proteger contra DDoS com Shield
- Criptografar dados em repouso e trnsito com KMS
- Gerenciar secrets com rotao automtica
- Detectar dados sensveis com Macie
- Bloquear acesso pblico a S3 em toda a organizao

---

### Level 300 — Advanced (Mdulos 05-07)

> *Voc  proficiente em segurana AWS. Quer dominar incident response, compliance contnua e segurana de rede avanada.*

| # | Mdulo | Desafios | Tempo | O Que Voc Vai Aprender |
|---|--------|----------|-------|------------------------|
| 05 | [**Incident Response**](modulo-05-incident-response.md) | 29-35 | 14-18h | IR Framework (NIST), playbooks automatizados com Step Functions, forensics (EBS snapshots, memory dump), isolamento de instncia comprometida, GuardDuty  Lambda  WAF auto-block, runbooks SSM para containment, post-mortem e lessons learned |
| 06 | [**Compliance & Governance**](modulo-06-compliance-governance.md) | 36-42 | 12-16h | AWS Config (managed + custom rules, conformance packs, auto-remediation), Audit Manager (frameworks: SOC 2, PCI DSS, HIPAA, ISO 27001), Control Tower (landing zone, guardrails, account factory), Organizations (SCPs, tag policies, backup policies), Service Catalog (portfolios aprovados), multi-account strategy |
| 07 | [**Network Security**](modulo-07-network-security.md) | 43-49 | 12-16h | VPC design seguro (pblico/privado/isolado), VPC Flow Logs (Athena analysis), Transit Gateway com inspection VPC, Route 53 Resolver DNS Firewall, PrivateLink e VPC Endpoints (Gateway vs Interface), VPN site-to-site e Client VPN, network segmentation patterns |

**Ao completar Level 300, voc sabe:**
- Responder a incidentes seguindo o framework NIST
- Automatizar containment e forensics
- Manter compliance contnua com Config + Audit Manager
- Implementar landing zone segura com Control Tower
- Desenhar VPCs seguros com segmentao de rede
- Analisar Flow Logs para deteco de anomalias

---

### Level 400 — Expert (Mdulos 08-10)

> *Voc opera segurana AWS em produo. Quer dominar segurana de aplicao, operaes de segurana em escala e arquiteturas enterprise.*

| # | Mdulo | Desafios | Tempo | O Que Voc Vai Aprender |
|---|--------|----------|-------|------------------------|
| 08 | [**Application Security**](modulo-08-application-security.md) | 50-55 | 10-14h | Cognito (User Pools, Identity Pools, OAuth2/OIDC, MFA), Verified Access (zero-trust), WAF para APIs (API Gateway + CloudFront), CodeGuru Security (SAST), container security (ECR scanning, ECS/EKS security), Lambda security (concurrency, VPC, layers) |
| 09 | [**Security Operations Center**](modulo-09-secops.md) | 56-62 | 12-16h | Security Lake (OCSF format), SIEM integration (Splunk, Datadog, OpenSearch), threat hunting com Athena, automated remediation pipelines, security metrics e KPIs, multi-account security aggregation, security chatops (Slack/Teams) |
| 10 | [**Cenrios Expert**](modulo-10-cenarios-expert.md) | 63-70 | 14-18h | **CAPSTONE:** Empresa fintech completa (multi-account, PCI DSS), migrao de on-prem para AWS segura, zero-trust architecture, supply chain security, red team / blue team exercises, AWS Security Specialty certification prep |

**Ao completar Level 400, voc :**
- Capaz de arquitetar segurana AWS para qualquer cenrio
- Referncia tcnica em security operations
- Preparado para cenrios enterprise regulados (fintech, sade)
- Operacionalmente maduro com SIEM, threat hunting e automao
- Pronto para AWS Security Specialty (SCS-C02) certification

---

## ndice Completo de Desafios

### Mdulo 01 — IAM & Identity (Level 100)
| # | Desafio | Tempo | Servios |
|---|---------|-------|----------|
| 1 | IAM Fundamentals — Users, Groups e Policies Managed | 60 min | IAM |
| 2 | Custom IAM Policies — Least Privilege na Prtica | 90 min | IAM |
| 3 | IAM Roles e STS — AssumeRole Cross-Account | 90 min | IAM, STS |
| 4 | Permission Boundaries — Delegao Segura | 60 min | IAM |
| 5 | IAM Access Analyzer — Detectar Acesso Externo | 60 min | IAM Access Analyzer |
| 6 | IAM Identity Center (SSO) — Single Sign-On | 90 min | IAM Identity Center |
| 7 | MFA Enforcement e Password Policies | 45 min | IAM |

### Mdulo 02 — Detection & Monitoring (Level 100-200)
| # | Desafio | Tempo | Servios |
|---|---------|-------|----------|
| 8 | CloudTrail — Setup Completo e Log Integrity | 90 min | CloudTrail, S3 |
| 9 | CloudTrail + Athena — Queries de Auditoria de Segurana | 90 min | CloudTrail, Athena |
| 10 | GuardDuty — Deteco de Ameaas e Findings | 90 min | GuardDuty |
| 11 | GuardDuty — Auto-Remediation com Lambda | 90 min | GuardDuty, Lambda, EventBridge |
| 12 | Security Hub — Postura de Segurana Centralizada | 90 min | Security Hub, Config |
| 13 | Inspector — Scan de Vulnerabilidades (EC2, Lambda, ECR) | 60 min | Inspector |
| 14 | Detective — Investigao de Incidentes | 60 min | Detective |

### Mdulo 03 — Infrastructure Protection (Level 200)
| # | Desafio | Tempo | Servios |
|---|---------|-------|----------|
| 15 | WAF — Rules, Managed Groups e Custom Regex | 120 min | WAF, CloudFront/ALB |
| 16 | WAF — Bot Control, Captcha e Fraud Control | 90 min | WAF |
| 17 | WAF — Rate Limiting Avanado com Custom Keys | 60 min | WAF |
| 18 | Shield — Standard vs Advanced e DRT Engagement | 60 min | Shield |
| 19 | Firewall Manager — WAF Centralizado Cross-Account | 90 min | Firewall Manager, Organizations |
| 20 | Network Firewall — Stateful Rules e Domain Filtering | 120 min | Network Firewall, VPC |
| 21 | Security Groups vs NACLs — Defense in Depth | 60 min | VPC, EC2 |

### Mdulo 04 — Data Protection (Level 200)
| # | Desafio | Tempo | Servios |
|---|---------|-------|----------|
| 22 | KMS — Key Types, Policies e Rotation | 90 min | KMS |
| 23 | KMS — Grants, Cross-Account e Multi-Region Keys | 90 min | KMS |
| 24 | Secrets Manager — Rotao Automtica e Lambda Integration | 90 min | Secrets Manager, Lambda, RDS |
| 25 | ACM — Certificate Lifecycle e Private CA | 60 min | ACM |
| 26 | Macie — Deteco de PII e Dados Sensveis em S3 | 90 min | Macie, S3 |
| 27 | S3 Security — Encryption, Block Public, Object Lock | 90 min | S3, KMS |
| 28 | CloudHSM — Hardware Security Module (FIPS 140-2 L3) | 60 min | CloudHSM |

### Mdulo 05 — Incident Response (Level 300)
| # | Desafio | Tempo | Servios |
|---|---------|-------|----------|
| 29 | IR Framework — Preparao e Plano de Resposta (NIST) | 60 min | Documentao |
| 30 | Playbook Automatizado — EC2 Comprometida | 120 min | Step Functions, Lambda, SSM |
| 31 | Playbook Automatizado — Credencial IAM Vazada | 120 min | Step Functions, Lambda, IAM |
| 32 | Forensics — Captura de Evidncias (EBS, Memory, Logs) | 120 min | EC2, EBS, SSM |
| 33 | GuardDuty → Lambda → WAF — Auto-Block Pipeline | 90 min | GuardDuty, Lambda, WAF |
| 34 | Containment — Isolamento de Recursos Comprometidos | 90 min | VPC, Security Groups, IAM |
| 35 | Post-Mortem e Lessons Learned Template | 60 min | Documentao |

### Mdulo 06 — Compliance & Governance (Level 300)
| # | Desafio | Tempo | Servios |
|---|---------|-------|----------|
| 36 | AWS Config — Managed Rules e Compliance Dashboard | 90 min | Config |
| 37 | AWS Config — Custom Rules e Auto-Remediation | 120 min | Config, Lambda, SSM |
| 38 | Audit Manager — SOC 2 e PCI DSS Assessment | 90 min | Audit Manager |
| 39 | Control Tower — Landing Zone Segura | 120 min | Control Tower |
| 40 | Organizations — SCPs para Security Guardrails | 90 min | Organizations |
| 41 | Tag Policies e Backup Policies — Governana de Recursos | 60 min | Organizations |
| 42 | Multi-Account Strategy — Hub-Spoke Security | 90 min | Organizations, RAM |

### Mdulo 07 — Network Security (Level 300)
| # | Desafio | Tempo | Servios |
|---|---------|-------|----------|
| 43 | VPC Design Seguro — Pblico, Privado e Isolado | 90 min | VPC |
| 44 | VPC Flow Logs — Captura e Anlise com Athena | 90 min | VPC, Athena |
| 45 | Transit Gateway — Inspection VPC com Network Firewall | 120 min | Transit Gateway, Network Firewall |
| 46 | DNS Firewall — Route 53 Resolver e Domain Filtering | 60 min | Route 53 |
| 47 | VPC Endpoints — Gateway vs Interface, PrivateLink | 90 min | VPC |
| 48 | VPN Site-to-Site e Client VPN | 90 min | VPN, VPC |
| 49 | Network Segmentation — Micro-Segmentao com SG | 60 min | VPC, EC2 |

### Mdulo 08 — Application Security (Level 400)
| # | Desafio | Tempo | Servios |
|---|---------|-------|----------|
| 50 | Cognito — User Pools, MFA e OAuth2/OIDC | 120 min | Cognito |
| 51 | Cognito — Identity Pools e Federated Access | 90 min | Cognito, STS |
| 52 | Verified Access — Zero-Trust sem VPN | 90 min | Verified Access |
| 53 | Container Security — ECR Scanning e ECS/EKS Hardening | 120 min | ECR, ECS, Inspector |
| 54 | Lambda Security — VPC, Layers, Concurrency e Permissions | 60 min | Lambda, IAM |
| 55 | API Security — API Gateway + WAF + Cognito Authorizer | 90 min | API Gateway, WAF, Cognito |

### Mdulo 09 — Security Operations (Level 400)
| # | Desafio | Tempo | Servios |
|---|---------|-------|----------|
| 56 | Security Lake — Centralizao de Logs (OCSF) | 120 min | Security Lake |
| 57 | SIEM Integration — OpenSearch + Security Dashboards | 120 min | OpenSearch |
| 58 | Threat Hunting — Hipteses e Queries Athena | 90 min | Athena, CloudTrail, VPC Flow Logs |
| 59 | Automated Remediation Pipeline — EventBridge + Step Functions | 120 min | EventBridge, Step Functions |
| 60 | Security Metrics e KPIs — Dashboard Executivo | 90 min | CloudWatch, QuickSight |
| 61 | Multi-Account Security Aggregation | 90 min | Security Hub, Organizations |
| 62 | Security ChatOps — Slack/Teams Integration | 60 min | SNS, Lambda, Chatbot |

### Mdulo 10 — Cenrios Expert (Level 400)
| # | Desafio | Tempo | Servios |
|---|---------|-------|----------|
| 63 | **CAPSTONE:** Fintech Multi-Account (PCI DSS Compliant) | 180 min | Todos |
| 64 | Migrao On-Prem → AWS Segura | 120 min | Migration Hub, CloudEndure |
| 65 | Zero-Trust Architecture na AWS | 120 min | Verified Access, PrivateLink, IAM |
| 66 | Supply Chain Security — CI/CD Seguro | 90 min | CodePipeline, CodeBuild, ECR |
| 67 | Red Team — Simulao de Ataques com Pacu/Prowler | 120 min | EC2, IAM |
| 68 | Blue Team — Deteco e Resposta ao Red Team | 120 min | GuardDuty, Detective, Security Hub |
| 69 | Disaster Recovery Security — Backup e Criptografia Cross-Region | 90 min | Backup, KMS, S3 |
| 70 | AWS Security Specialty (SCS-C02) — Prep e Prximos Passos | 60 min | — |

---

## Alinhamento com Pilares AWS Well-Architected (Security Pillar)

```
┌──────────────────────────────────────────────────────────────────────┐
│          AWS Well-Architected — Security Pillar  Este Workshop       │
│                                                                       │
│  SEC 1: Operate workloads securely                                   │
│  └── Mdulos 06 (Config, Audit Manager), 09 (SecOps)                │
│                                                                       │
│  SEC 2: Manage identities for people and machines                    │
│  └── Mdulo 01 (IAM, SSO, Roles, MFA)                               │
│                                                                       │
│  SEC 3: Manage permissions for people and machines                   │
│  └── Mdulo 01 (Policies, Permission Boundaries, Access Analyzer)    │
│                                                                       │
│  SEC 4: Detect and investigate security events                       │
│  └── Mdulo 02 (CloudTrail, GuardDuty, Security Hub, Detective)     │
│                                                                       │
│  SEC 5: Protect network resources                                    │
│  └── Mdulos 03 (WAF, Shield, Firewall), 07 (VPC, Flow Logs)       │
│                                                                       │
│  SEC 6: Protect compute resources                                    │
│  └── Mdulos 03 (SG/NACL), 08 (Container/Lambda Security)          │
│                                                                       │
│  SEC 7: Classify data                                                │
│  └── Mdulo 04 (Macie, S3 tagging)                                  │
│                                                                       │
│  SEC 8: Protect data at rest                                         │
│  └── Mdulo 04 (KMS, S3 encryption, CloudHSM)                       │
│                                                                       │
│  SEC 9: Protect data in transit                                      │
│  └── Mdulos 04 (ACM, TLS), 07 (VPN, PrivateLink)                  │
│                                                                       │
│  SEC 10: Anticipate, respond to, and recover from incidents          │
│  └── Mdulo 05 (IR playbooks, forensics, containment)               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Pr-requisitos

### Obrigatrios

- **Conta AWS** — Free Tier  suficiente para maioria dos desafios
- **AWS CLI v2** — Instalada e configurada (`aws configure`)
- **Terraform >= 1.5** — Para todos os desafios de IaC
- **Conhecimentos bsicos** — Linux, networking TCP/IP, HTTP

### Recomendados

- **Duas contas AWS** — Para desafios cross-account (Mdulos 01, 06, 09)
- **Python 3.12+** — Para Lambda functions e scripts
- **Docker** — Para desafios de container security (Mdulo 08)
- **Prowler** — Ferramenta open-source de security assessment
- **Pacu** — Framework de pentest AWS (Red Team, Mdulo 10)

### Custos Estimados

| Level | Custo Estimado | Nota |
|-------|---------------|------|
| 100 (Mdulos 01-02) | ~$0-5 | IAM  gratuito, GuardDuty tem 30 dias free |
| 200 (Mdulos 03-04) | ~$5-15 | WAF, Network Firewall, Macie |
| 300 (Mdulos 05-07) | ~$10-25 | EC2 para forensics, Transit Gateway |
| 400 (Mdulos 08-10) | ~$15-40 | Cognito, Security Lake, OpenSearch |

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
│  │ OBJETIVO   │ O que voc vai construir             │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ CENRIO    │ Situao real de produo              │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ARQUITETURA │ Diagrama ASCII da soluo            │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ PASSO A    │ AWS CLI commands                     │
│  │ PASSO      │ Terraform code                       │
│  │            │ Cdigo (Python/Node.js)              │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ VALIDAO  │ Commands para testar                  │
│  │            │ Simular ataque para verificar defesa │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ O QUE      │ Tabela de conceitos aprendidos       │
│  │ APRENDEMOS │                                      │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ EXPERT TIP │ Insight de produo real             │
│  └────────────┘                                      │
│  ┌────────────┐                                      │
│  │ CLEANUP    │ Destruir recursos para evitar custos │
│  └────────────┘                                      │
└─────────────────────────────────────────────────────┘
```

### Recomendaes

1. **Siga a ordem** — Os mdulos so progressivos. Cada um assume conhecimento dos anteriores.
2. **Execute tudo** — Ler no  suficiente. Crie os recursos, simule ataques, veja os findings.
3. **Use Terraform** — Todo desafio tem verso CLI e Terraform. Pratique ambos.
4. **Faa cleanup** — Destrua recursos aps cada desafio para evitar custos.
5. **Simule ataques** — Use ferramentas como Prowler e Pacu para testar suas defesas.
6. **Documente** — Crie seus prprios runbooks e playbooks ao longo do workshop.

---

## Navegao Rpida

### Level 100 — Foundational
- [Mdulo 01 — IAM & Identity](modulo-01-iam-identity.md) (Desafios 1-7)
- [Mdulo 02 — Detection & Monitoring](modulo-02-detection-monitoring.md) (Desafios 8-14)

### Level 200 — Intermediate
- [Mdulo 03 — Infrastructure Protection](modulo-03-infrastructure-protection.md) (Desafios 15-21)
- [Mdulo 04 — Data Protection](modulo-04-data-protection.md) (Desafios 22-28)

### Level 300 — Advanced
- [Mdulo 05 — Incident Response](modulo-05-incident-response.md) (Desafios 29-35)
- [Mdulo 06 — Compliance & Governance](modulo-06-compliance-governance.md) (Desafios 36-42)
- [Mdulo 07 — Network Security](modulo-07-network-security.md) (Desafios 43-49)

### Level 400 — Expert
- [Mdulo 08 — Application Security](modulo-08-application-security.md) (Desafios 50-55)
- [Mdulo 09 — Security Operations](modulo-09-secops.md) (Desafios 56-62)
- [Mdulo 10 — Cenrios Expert](modulo-10-cenarios-expert.md) (Desafios 63-70)

---

## Referncias

- [AWS Well-Architected — Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)
- [AWS Security Best Practices](https://docs.aws.amazon.com/security/)
- [AWS Security Hub Controls Reference](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-controls-reference.html)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [CIS AWS Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services)
- [AWS Security Specialty (SCS-C02) Exam Guide](https://aws.amazon.com/certification/certified-security-specialty/)
- [Terraform AWS Provider — Security Services](https://registry.terraform.io/providers/hashicorp/aws/latest/)
- [Prowler — Open Source Security Assessment](https://github.com/prowler-cloud/prowler)

---

> **Workshop criado para transformar voc em referncia tcnica em segurana AWS.
> 70 desafios. 10 mdulos. Do zero ao expert.**
