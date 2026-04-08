# Módulo 10 — Cenários Reais de Produção (Level 400 - Expert)

> **Nível**: 400 (Expert) | **Pré-requisitos**: Módulos 1-9 completos
> **Formato**: AWS Workshop — Hands-on com cenários reais de produção
> **Objetivo**: Aplicar TUDO que aprendemos em cenários complexos de produção real

---

## Visão Geral do Módulo Final

Este é o módulo culminante do guia. Aqui você enfrentará cenários que combinam **todos** os
conhecimentos adquiridos nos módulos anteriores. Cada desafio simula uma situação real que
arquitetos de soluções e engenheiros DevOps enfrentam em empresas de grande porte.

```
╔══════════════════════════════════════════════════════════════════╗
║                   MÓDULO 10 — O GRAND FINALE                    ║
║                                                                  ║
║  Desafio 56: E-commerce Black Friday (Capstone)                 ║
║  Desafio 57: Multi-Region Active-Active                         ║
║  Desafio 58: Migração de CDN                                    ║
║  Desafio 59: Video Streaming Platform                           ║
║  Desafio 60: Certificação e Próximos Passos                     ║
║                                                                  ║
║  "De estudante a especialista — a jornada completa."            ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## Desafio 56: Arquitetura Completa — E-commerce Black Friday (Level 400)

### Classificação

| Atributo         | Valor                                            |
|------------------|--------------------------------------------------|
| **Nível**        | 400 — Expert                                     |
| **Tipo**         | Capstone Project                                 |
| **Tempo**        | 4-6 horas                                        |
| **Serviços**     | CloudFront, WAF, Shield, S3, ALB, ECS, ACM, R53 |
| **Complexidade** | Muito Alta                                       |

### Objetivo

Projetar e implementar a arquitetura **completa** de CDN para um e-commerce que precisa
suportar **100.000 requisições por segundo** durante a Black Friday, com alta disponibilidade,
segurança robusta e performance otimizada globalmente.

### Cenário Real

> Você é o Arquiteto de Soluções responsável pela infraestrutura de um grande e-commerce
> brasileiro. A Black Friday está chegando e o CTO exige que o sistema aguente 100x o tráfego
> normal sem degradação. O orçamento foi aprovado, mas a janela de implementação é curta.
> Cada minuto de indisponibilidade custa R$ 500.000 em vendas perdidas.

### Requisitos de Negócio

1. **Performance**: Latência P99 < 200ms para qualquer região do Brasil
2. **Disponibilidade**: 99.99% uptime durante o evento (52s de downtime máximo em 6 dias)
3. **Segurança**: Proteção contra DDoS L3/L4/L7, bot mitigation, WAF
4. **Escalabilidade**: 100k req/s sustentado, picos de 200k req/s
5. **Observabilidade**: Dashboards em tempo real, alertas proativos
6. **Custo**: Otimizado — cache hit ratio > 95%

### Diagrama de Arquitetura

```
                            ┌─────────────────────────────────────────────────────────────┐
                            │                     INTERNET / CLIENTES                      │
                            └───────────────────────────┬─────────────────────────────────┘
                                                        │
                                                        ▼
                            ┌─────────────────────────────────────────────────────────────┐
                            │                    Route 53 (DNS)                            │
                            │              loja.exemplo.com.br                             │
                            │         ALIAS → CloudFront Distribution                     │
                            └───────────────────────────┬─────────────────────────────────┘
                                                        │
                                                        ▼
                    ┌───────────────────────────────────────────────────────────────────┐
                    │                        AWS Shield Advanced                        │
                    │                   Proteção DDoS L3/L4 automática                  │
                    └───────────────────────────┬───────────────────────────────────────┘
                                                │
                                                ▼
        ┌───────────────────────────────────────────────────────────────────────────────────┐
        │                              AWS WAF v2                                           │
        │                                                                                   │
        │   ┌─────────────┐  ┌──────────────┐  ┌────────────┐  ┌─────────────────────────┐ │
        │   │ Rate Limit  │  │  SQL/XSS     │  │  Bot       │  │  Geo-Block              │ │
        │   │ 2000 req/   │  │  Injection   │  │  Control   │  │  (países sancionados)   │ │
        │   │ 5min/IP     │  │  Protection  │  │  Managed   │  │                         │ │
        │   └─────────────┘  └──────────────┘  └────────────┘  └─────────────────────────┘ │
        │                                                                                   │
        │   ┌─────────────────────┐  ┌────────────────────────────────────────────────────┐ │
        │   │  IP Reputation      │  │  Custom Rules (Header validation, URI patterns)    │ │
        │   │  (AWS Managed)      │  │                                                    │ │
        │   └─────────────────────┘  └────────────────────────────────────────────────────┘ │
        └───────────────────────────────────┬───────────────────────────────────────────────┘
                                            │
                                            ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                             CloudFront Distribution                                          │
│                          d1234abcdef.cloudfront.net                                          │
│                                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │                            Origin Shield (sa-east-1)                                    │ │
│  │                     Camada extra de cache — reduz carga na origem                       │ │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │
│  │  CF Function:    │  │  CF Function:    │  │  CF Function:    │  │  Lambda@Edge:       │  │
│  │  URL Normalize   │  │  A/B Testing     │  │  Geo-Redirect    │  │  JWT Auth           │  │
│  │  (Viewer Req)    │  │  (Viewer Req)    │  │  (Viewer Req)    │  │  (Viewer Req)       │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  └─────────────────────┘  │
│                                                                                              │
│  BEHAVIORS:                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────────────────┐    │
│  │ Path Pattern          │ Origin          │ Cache   │ TTL    │ Funções               │    │
│  │───────────────────────│─────────────────│─────────│────────│───────────────────────│    │
│  │ /api/v1/config        │ ALB (API)       │ 60s     │ 60s    │ JWT Auth              │    │
│  │ /api/*                │ ALB (API)       │ Disabled│ 0      │ JWT Auth              │    │
│  │ /ws/*                 │ ALB (WebSocket) │ Disabled│ 0      │ —                     │    │
│  │ /media/*              │ S3 (Media)      │ 30 dias │ 2592000│ —                     │    │
│  │ /static/*             │ S3 (SPA)        │ 365 dias│ 31536000│ URL Normalize        │    │
│  │ /* (default)          │ S3 (SPA)        │ 1 hora  │ 3600   │ A/B, Geo-Redirect    │    │
│  └──────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                              │
│  ORIGIN GROUPS (Failover):                                                                   │
│  ┌────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ API Origin Group:    Primary → ALB sa-east-1  │  Secondary → ALB us-east-1           │  │
│  │ Media Origin Group:  Primary → S3 sa-east-1   │  Secondary → S3 us-east-1 (replica)  │  │
│  └────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                              │
└──────────────┬───────────────────┬──────────────────────┬──────────────────┬─────────────────┘
               │                   │                      │                  │
               ▼                   ▼                      ▼                  ▼
    ┌──────────────────┐ ┌──────────────────┐  ┌──────────────────┐ ┌──────────────────┐
    │   S3 Bucket      │ │   ALB + ECS      │  │   S3 Bucket      │ │   ALB            │
    │   (SPA/Frontend) │ │   (API Backend)  │  │   (Media Assets) │ │   (WebSocket)    │
    │                  │ │                  │  │                  │ │                  │
    │ - React SPA      │ │ - 3 services     │  │ - Product images │ │ - Notifications  │
    │ - index.html     │ │ - Auto Scaling   │  │ - Videos         │ │ - Live cart      │
    │ - assets/        │ │ - 4 vCPU/8GB     │  │ - Thumbnails     │ │ - Price updates  │
    │                  │ │ - min: 10        │  │ - CDN optimized  │ │                  │
    │ OAC enabled      │ │ - max: 200       │  │ - OAC enabled    │ │ Sticky sessions  │
    └──────────────────┘ └──────────────────┘  └──────────────────┘ └──────────────────┘
                                  │
                                  ▼
                       ┌──────────────────┐
                       │   CloudWatch     │
                       │                  │
                       │ - Dashboard RT   │
                       │ - Alarmes        │
                       │ - Logs (sampled) │
                       │ - Métricas       │
                       └──────────────────┘
```

### Passo a Passo — Implementação Completa

#### Fase 1: Infraestrutura Base (VPC e Rede)

```hcl
# ==============================================================================
# providers.tf — Configuração dos providers
# ==============================================================================

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "terraform-state-ecommerce-prod"
    key            = "cloudfront-blackfriday/terraform.tfstate"
    region         = "sa-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = "sa-east-1"

  default_tags {
    tags = {
      Project     = "ecommerce-blackfriday"
      Environment = "production"
      ManagedBy   = "terraform"
      CostCenter  = "engineering"
    }
  }
}

# Provider para recursos globais (CloudFront, WAF, ACM)
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"

  default_tags {
    tags = {
      Project     = "ecommerce-blackfriday"
      Environment = "production"
      ManagedBy   = "terraform"
      CostCenter  = "engineering"
    }
  }
}
```

```hcl
# ==============================================================================
# variables.tf — Variáveis do projeto
# ==============================================================================

variable "project_name" {
  description = "Nome do projeto"
  type        = string
  default     = "ecommerce-bf"
}

variable "environment" {
  description = "Ambiente (production, staging)"
  type        = string
  default     = "production"
}

variable "domain_name" {
  description = "Domínio principal"
  type        = string
  default     = "loja.exemplo.com.br"
}

variable "hosted_zone_id" {
  description = "ID da hosted zone no Route 53"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block da VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "AZs para deploy"
  type        = list(string)
  default     = ["sa-east-1a", "sa-east-1b", "sa-east-1c"]
}

variable "ecs_min_capacity" {
  description = "Mínimo de tasks ECS"
  type        = number
  default     = 10
}

variable "ecs_max_capacity" {
  description = "Máximo de tasks ECS"
  type        = number
  default     = 200
}

variable "ab_test_percentage" {
  description = "Percentual de tráfego para variante B"
  type        = number
  default     = 10
}

variable "waf_rate_limit" {
  description = "Rate limit (requisições por 5 minutos por IP)"
  type        = number
  default     = 2000
}
```

```hcl
# ==============================================================================
# vpc.tf — VPC e Networking
# ==============================================================================

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# Subnets públicas (ALB)
resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-${var.availability_zones[count.index]}"
    Tier = "public"
  }
}

# Subnets privadas (ECS)
resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.project_name}-private-${var.availability_zones[count.index]}"
    Tier = "private"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Elastic IPs para NAT Gateways
resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"

  tags = {
    Name = "${var.project_name}-nat-eip-${count.index}"
  }
}

# NAT Gateways (um por AZ para alta disponibilidade)
resource "aws_nat_gateway" "main" {
  count = length(var.availability_zones)

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.project_name}-nat-${var.availability_zones[count.index]}"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route table pública
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Route tables privadas (uma por AZ)
resource "aws_route_table" "private" {
  count = length(var.availability_zones)

  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${var.project_name}-private-rt-${var.availability_zones[count.index]}"
  }
}

resource "aws_route_table_association" "private" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

#### Fase 2: S3 Buckets

```hcl
# ==============================================================================
# s3.tf — Buckets S3 para SPA e Media
# ==============================================================================

# Bucket do SPA (Frontend React)
resource "aws_s3_bucket" "spa" {
  bucket = "${var.project_name}-spa-${var.environment}"

  tags = {
    Name    = "SPA Frontend"
    Purpose = "Single Page Application hosting"
  }
}

resource "aws_s3_bucket_versioning" "spa" {
  bucket = aws_s3_bucket.spa.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "spa" {
  bucket = aws_s3_bucket.spa.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "spa" {
  bucket = aws_s3_bucket.spa.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Bucket de Media (Imagens, Vídeos de produtos)
resource "aws_s3_bucket" "media" {
  bucket = "${var.project_name}-media-${var.environment}"

  tags = {
    Name    = "Media Assets"
    Purpose = "Product images and videos"
  }
}

resource "aws_s3_bucket_versioning" "media" {
  bucket = aws_s3_bucket.media.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "media" {
  bucket = aws_s3_bucket.media.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "media" {
  bucket = aws_s3_bucket.media.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "media" {
  bucket = aws_s3_bucket.media.id

  rule {
    id     = "move-old-versions-to-glacier"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER"
    }

    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}

# Bucket de logs do CloudFront
resource "aws_s3_bucket" "cf_logs" {
  bucket = "${var.project_name}-cf-logs-${var.environment}"

  tags = {
    Name    = "CloudFront Access Logs"
    Purpose = "Standard and real-time logs"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "cf_logs" {
  bucket = aws_s3_bucket.cf_logs.id

  rule {
    id     = "archive-logs"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }
}

# S3 bucket para replica (failover de media)
resource "aws_s3_bucket" "media_replica" {
  provider = aws.us_east_1
  bucket   = "${var.project_name}-media-replica-${var.environment}"

  tags = {
    Name    = "Media Assets Replica"
    Purpose = "Cross-region replica for failover"
  }
}

resource "aws_s3_bucket_versioning" "media_replica" {
  provider = aws.us_east_1
  bucket   = aws_s3_bucket.media_replica.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

#### Fase 3: ACM (Certificados TLS)

```hcl
# ==============================================================================
# acm.tf — Certificados SSL/TLS
# ==============================================================================

# Certificado para CloudFront (DEVE ser em us-east-1)
resource "aws_acm_certificate" "cloudfront" {
  provider = aws.us_east_1

  domain_name               = var.domain_name
  subject_alternative_names = ["*.${var.domain_name}"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "${var.project_name}-cf-cert"
  }
}

# Validação DNS do certificado
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.cloudfront.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = var.hosted_zone_id
}

resource "aws_acm_certificate_validation" "cloudfront" {
  provider = aws.us_east_1

  certificate_arn         = aws_acm_certificate.cloudfront.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# Certificado para ALB (sa-east-1)
resource "aws_acm_certificate" "alb" {
  domain_name               = "api.${var.domain_name}"
  subject_alternative_names = ["ws.${var.domain_name}"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "${var.project_name}-alb-cert"
  }
}
```

#### Fase 4: ALB e ECS

```hcl
# ==============================================================================
# alb.tf — Application Load Balancer
# ==============================================================================

resource "aws_security_group" "alb" {
  name_prefix = "${var.project_name}-alb-"
  vpc_id      = aws_vpc.main.id

  # Permitir tráfego apenas dos IPs do CloudFront
  # Usa o prefix list managed da AWS
  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    prefix_list_ids = [data.aws_ec2_managed_prefix_list.cloudfront.id]
    description     = "HTTPS from CloudFront"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-alb-sg"
  }
}

data "aws_ec2_managed_prefix_list" "cloudfront" {
  name = "com.amazonaws.global.cloudfront.origin-facing"
}

# ALB principal (API)
resource "aws_lb" "api" {
  name               = "${var.project_name}-api-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = true
  enable_http2               = true

  access_logs {
    bucket  = aws_s3_bucket.cf_logs.id
    prefix  = "alb-logs"
    enabled = true
  }

  tags = {
    Name = "${var.project_name}-api-alb"
  }
}

# Listener HTTPS
resource "aws_lb_listener" "api_https" {
  load_balancer_arn = aws_lb.api.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.alb.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

# Custom header check (apenas CloudFront pode acessar)
resource "aws_lb_listener_rule" "verify_cloudfront" {
  listener_arn = aws_lb_listener.api_https.arn
  priority     = 1

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    http_header {
      http_header_name = "X-Custom-Origin-Secret"
      values           = [random_password.origin_secret.result]
    }
  }
}

resource "random_password" "origin_secret" {
  length  = 64
  special = false
}

# Target Group API
resource "aws_lb_target_group" "api" {
  name        = "${var.project_name}-api-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 10
    path                = "/health"
    matcher             = "200"
  }

  deregistration_delay = 30

  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
    enabled         = false
  }

  tags = {
    Name = "${var.project_name}-api-tg"
  }
}

# ALB para WebSocket
resource "aws_lb" "websocket" {
  name               = "${var.project_name}-ws-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = true
  enable_http2               = true
  idle_timeout               = 3600  # 1 hora para WebSocket

  tags = {
    Name = "${var.project_name}-ws-alb"
  }
}

# Target Group WebSocket
resource "aws_lb_target_group" "websocket" {
  name        = "${var.project_name}-ws-tg"
  port        = 8081
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 10
    path                = "/ws/health"
    matcher             = "200"
  }

  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
    enabled         = true  # Sticky para WebSocket
  }

  tags = {
    Name = "${var.project_name}-ws-tg"
  }
}
```

```hcl
# ==============================================================================
# ecs.tf — ECS Cluster e Services
# ==============================================================================

resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  configuration {
    execute_command_configuration {
      logging = "OVERRIDE"

      log_configuration {
        cloud_watch_log_group_name = aws_cloudwatch_log_group.ecs.name
      }
    }
  }

  tags = {
    Name = "${var.project_name}-cluster"
  }
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    base              = 10
    weight            = 70
    capacity_provider = "FARGATE"
  }

  default_capacity_provider_strategy {
    weight            = 30
    capacity_provider = "FARGATE_SPOT"
  }
}

resource "aws_cloudwatch_log_group" "ecs" {
  name              = "/ecs/${var.project_name}"
  retention_in_days = 30
}

# Security Group das tasks ECS
resource "aws_security_group" "ecs_tasks" {
  name_prefix = "${var.project_name}-ecs-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8081
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "Allow traffic from ALB"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-ecs-tasks-sg"
  }
}

# Task Definition — API
resource "aws_ecs_task_definition" "api" {
  family                   = "${var.project_name}-api"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 4096   # 4 vCPU
  memory                   = 8192   # 8 GB
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "api"
      image = "${aws_ecr_repository.api.repository_url}:latest"

      portMappings = [
        {
          containerPort = 8080
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "NODE_ENV"
          value = "production"
        },
        {
          name  = "PORT"
          value = "8080"
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.ecs.name
          "awslogs-region"        = "sa-east-1"
          "awslogs-stream-prefix" = "api"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
        interval    = 10
        timeout     = 5
        retries     = 3
        startPeriod = 30
      }
    }
  ])

  tags = {
    Name = "${var.project_name}-api-task"
  }
}

# ECS Service — API
resource "aws_ecs_service" "api" {
  name            = "${var.project_name}-api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = var.ecs_min_capacity

  deployment_minimum_healthy_percent = 100
  deployment_maximum_percent         = 200
  health_check_grace_period_seconds  = 60
  enable_execute_command             = true

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 70
    base              = 10
  }

  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 30
  }

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "api"
    container_port   = 8080
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  tags = {
    Name = "${var.project_name}-api-service"
  }
}

# ECR Repository
resource "aws_ecr_repository" "api" {
  name                 = "${var.project_name}-api"
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }
}

# IAM Roles
resource "aws_iam_role" "ecs_execution" {
  name = "${var.project_name}-ecs-execution"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_execution" {
  role       = aws_iam_role.ecs_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role" "ecs_task" {
  name = "${var.project_name}-ecs-task"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}
```

#### Fase 5: Auto Scaling

```hcl
# ==============================================================================
# autoscaling.tf — ECS Auto Scaling
# ==============================================================================

resource "aws_appautoscaling_target" "api" {
  max_capacity       = var.ecs_max_capacity
  min_capacity       = var.ecs_min_capacity
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.api.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

# Scaling por CPU
resource "aws_appautoscaling_policy" "api_cpu" {
  name               = "${var.project_name}-api-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension
  service_namespace  = aws_appautoscaling_target.api.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 60.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# Scaling por Request Count
resource "aws_appautoscaling_policy" "api_requests" {
  name               = "${var.project_name}-api-request-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension
  service_namespace  = aws_appautoscaling_target.api.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.api.arn_suffix}/${aws_lb_target_group.api.arn_suffix}"
    }
    target_value       = 1000.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 30
  }
}

# Scheduled Scaling — Pre-warming para Black Friday
resource "aws_appautoscaling_scheduled_action" "bf_warmup" {
  name               = "${var.project_name}-bf-warmup"
  service_namespace  = aws_appautoscaling_target.api.service_namespace
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension

  # Escalar 2 horas antes da Black Friday
  schedule = "cron(0 22 * 11 THU *)"  # Quinta 22h UTC (19h BRT)

  scalable_target_action {
    min_capacity = 100
    max_capacity = var.ecs_max_capacity
  }
}

# Voltar ao normal após Black Friday
resource "aws_appautoscaling_scheduled_action" "bf_cooldown" {
  name               = "${var.project_name}-bf-cooldown"
  service_namespace  = aws_appautoscaling_target.api.service_namespace
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension

  # Voltar ao normal na segunda-feira
  schedule = "cron(0 6 * 12 MON *)"

  scalable_target_action {
    min_capacity = var.ecs_min_capacity
    max_capacity = var.ecs_max_capacity
  }
}
```

#### Fase 6: CloudFront Functions e Lambda@Edge

```hcl
# ==============================================================================
# functions.tf — CloudFront Functions e Lambda@Edge
# ==============================================================================

# --- CloudFront Function: URL Normalization ---
resource "aws_cloudfront_function" "url_normalize" {
  name    = "${var.project_name}-url-normalize"
  runtime = "cloudfront-js-2.0"
  publish = true
  code    = <<-EOF
    function handler(event) {
      var request = event.request;
      var uri = request.uri;

      // Remover trailing slash (exceto root)
      if (uri.length > 1 && uri.endsWith('/')) {
        uri = uri.slice(0, -1);
      }

      // Lowercase no path
      uri = uri.toLowerCase();

      // Normalizar query strings — ordenar parametros
      if (request.querystring) {
        var params = Object.keys(request.querystring).sort();
        var sorted = {};
        for (var i = 0; i < params.length; i++) {
          sorted[params[i]] = request.querystring[params[i]];
        }
        request.querystring = sorted;
      }

      // Remover extensão .html
      if (uri.endsWith('.html') && uri !== '/index.html') {
        uri = uri.slice(0, -5);
      }

      request.uri = uri;
      return request;
    }
  EOF
}

# --- CloudFront Function: A/B Testing ---
resource "aws_cloudfront_function" "ab_test" {
  name    = "${var.project_name}-ab-test"
  runtime = "cloudfront-js-2.0"
  publish = true
  code    = <<-EOF
    function handler(event) {
      var request = event.request;
      var headers = request.headers;
      var cookies = request.cookies;

      // Verificar se já tem cookie de A/B test
      var abCookie = cookies['x-ab-variant'];
      var variant = 'A';

      if (abCookie) {
        variant = abCookie.value;
      } else {
        // Novo visitante: atribuir variante
        // ${var.ab_test_percentage}% vai para B
        var rand = Math.random() * 100;
        variant = rand < ${var.ab_test_percentage} ? 'B' : 'A';
      }

      // Adicionar header para a origem saber a variante
      request.headers['x-ab-variant'] = { value: variant };

      // Se variante B, redirecionar para pasta /b/
      if (variant === 'B' && request.uri === '/') {
        request.uri = '/b/index.html';
      }

      return request;
    }
  EOF
}

# --- CloudFront Function: Geo-Redirect ---
resource "aws_cloudfront_function" "geo_redirect" {
  name    = "${var.project_name}-geo-redirect"
  runtime = "cloudfront-js-2.0"
  publish = true
  code    = <<-EOF
    function handler(event) {
      var request = event.request;
      var headers = request.headers;
      var country = headers['cloudfront-viewer-country']
        ? headers['cloudfront-viewer-country'].value
        : 'BR';

      // Redirecionar para sub-domínio regional
      var regionMap = {
        'AR': 'ar',
        'CL': 'cl',
        'CO': 'co',
        'MX': 'mx',
        'PE': 'pe'
      };

      if (regionMap[country] && request.uri === '/') {
        var response = {
          statusCode: 302,
          statusDescription: 'Found',
          headers: {
            'location': { value: '/' + regionMap[country] + '/' },
            'cache-control': { value: 'max-age=3600' }
          }
        };
        return response;
      }

      // Adicionar headers de localização para a origem
      request.headers['x-viewer-country'] = { value: country };

      return request;
    }
  EOF
}

# --- Lambda@Edge: JWT Authentication ---
resource "aws_iam_role" "lambda_edge" {
  provider = aws.us_east_1
  name     = "${var.project_name}-lambda-edge-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = [
            "lambda.amazonaws.com",
            "edgelambda.amazonaws.com"
          ]
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_edge_basic" {
  provider   = aws.us_east_1
  role       = aws_iam_role.lambda_edge.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

data "archive_file" "jwt_auth" {
  type        = "zip"
  output_path = "${path.module}/lambda/jwt-auth.zip"

  source {
    content  = <<-EOF
      'use strict';

      const crypto = require('crypto');

      // Chave pública para verificar JWT (em produção, use Secrets Manager)
      const JWT_SECRET = process.env.JWT_SECRET || 'your-256-bit-secret';
      const ISSUER = 'auth.exemplo.com.br';

      // Caminhos que não precisam de autenticação
      const PUBLIC_PATHS = [
        '/api/v1/auth/login',
        '/api/v1/auth/register',
        '/api/v1/auth/refresh',
        '/api/v1/products',
        '/api/v1/categories',
        '/api/v1/health',
      ];

      exports.handler = async (event) => {
        const request = event.Records[0].cf.request;
        const uri = request.uri;

        // Verificar se o path é público
        if (PUBLIC_PATHS.some(path => uri.startsWith(path) && request.method === 'GET')) {
          return request;
        }

        // Extrair token do header Authorization
        const authHeader = request.headers['authorization'];
        if (!authHeader || !authHeader[0]) {
          return unauthorizedResponse('Token não fornecido');
        }

        const token = authHeader[0].value.replace('Bearer ', '');

        try {
          // Decodificar e validar JWT (simplificado)
          const parts = token.split('.');
          if (parts.length !== 3) {
            return unauthorizedResponse('Token inválido');
          }

          const header = JSON.parse(Buffer.from(parts[0], 'base64url').toString());
          const payload = JSON.parse(Buffer.from(parts[1], 'base64url').toString());

          // Verificar expiração
          if (payload.exp && payload.exp < Math.floor(Date.now() / 1000)) {
            return unauthorizedResponse('Token expirado');
          }

          // Verificar issuer
          if (payload.iss !== ISSUER) {
            return unauthorizedResponse('Issuer inválido');
          }

          // Adicionar informações do usuário nos headers
          request.headers['x-user-id'] = [{ key: 'X-User-Id', value: payload.sub }];
          request.headers['x-user-role'] = [{ key: 'X-User-Role', value: payload.role || 'user' }];

          return request;
        } catch (err) {
          console.error('JWT validation error:', err);
          return unauthorizedResponse('Erro na validação do token');
        }
      };

      function unauthorizedResponse(message) {
        return {
          status: '401',
          statusDescription: 'Unauthorized',
          headers: {
            'content-type': [{ key: 'Content-Type', value: 'application/json' }],
            'cache-control': [{ key: 'Cache-Control', value: 'no-store' }],
          },
          body: JSON.stringify({
            error: 'Unauthorized',
            message: message,
          }),
        };
      }
    EOF
    filename = "index.js"
  }
}

resource "aws_lambda_function" "jwt_auth" {
  provider = aws.us_east_1

  filename         = data.archive_file.jwt_auth.output_path
  function_name    = "${var.project_name}-jwt-auth"
  role             = aws_iam_role.lambda_edge.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = data.archive_file.jwt_auth.output_base64sha256
  publish          = true
  timeout          = 5
  memory_size      = 128

  tags = {
    Name = "${var.project_name}-jwt-auth"
  }
}
```

#### Fase 7: WAF (Web Application Firewall)

```hcl
# ==============================================================================
# waf.tf — AWS WAF v2 para CloudFront
# ==============================================================================

resource "aws_wafv2_web_acl" "main" {
  provider = aws.us_east_1

  name        = "${var.project_name}-waf"
  description = "WAF para e-commerce Black Friday"
  scope       = "CLOUDFRONT"

  default_action {
    allow {}
  }

  # Regra 1: Rate Limiting por IP
  rule {
    name     = "rate-limit-per-ip"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = var.waf_rate_limit
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitPerIP"
      sampled_requests_enabled   = true
    }
  }

  # Regra 2: AWS Managed Rules — Common Rule Set
  rule {
    name     = "aws-managed-common"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"

        rule_action_override {
          name = "SizeRestrictions_BODY"
          action_to_use {
            count {}
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedCommon"
      sampled_requests_enabled   = true
    }
  }

  # Regra 3: SQL Injection Protection
  rule {
    name     = "aws-managed-sqli"
    priority = 3

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedSQLi"
      sampled_requests_enabled   = true
    }
  }

  # Regra 4: Known Bad Inputs
  rule {
    name     = "aws-managed-bad-inputs"
    priority = 4

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedBadInputs"
      sampled_requests_enabled   = true
    }
  }

  # Regra 5: IP Reputation
  rule {
    name     = "aws-managed-ip-reputation"
    priority = 5

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAmazonIpReputationList"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedIPReputation"
      sampled_requests_enabled   = true
    }
  }

  # Regra 6: Bot Control
  rule {
    name     = "aws-managed-bot-control"
    priority = 6

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesBotControlRuleSet"
        vendor_name = "AWS"

        managed_rule_group_configs {
          aws_managed_rules_bot_control_rule_set {
            inspection_level = "TARGETED"
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedBotControl"
      sampled_requests_enabled   = true
    }
  }

  # Regra 7: Geo-Block (países sancionados)
  rule {
    name     = "geo-block-sanctioned"
    priority = 7

    action {
      block {}
    }

    statement {
      geo_match_statement {
        country_codes = ["KP", "IR", "SY", "CU"]
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "GeoBlockSanctioned"
      sampled_requests_enabled   = true
    }
  }

  # Regra 8: Rate limit no login (prevenir brute force)
  rule {
    name     = "rate-limit-login"
    priority = 8

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 100
        aggregate_key_type = "IP"

        scope_down_statement {
          byte_match_statement {
            search_string         = "/api/v1/auth/login"
            field_to_match {
              uri_path {}
            }
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
            positional_constraint = "STARTS_WITH"
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitLogin"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "MainWAF"
    sampled_requests_enabled   = true
  }

  tags = {
    Name = "${var.project_name}-waf"
  }
}
```

#### Fase 8: CloudFront Distribution (O Centro de Tudo)

```hcl
# ==============================================================================
# cloudfront.tf — A distribuição CloudFront completa
# ==============================================================================

# OAC para S3
resource "aws_cloudfront_origin_access_control" "s3" {
  name                              = "${var.project_name}-s3-oac"
  description                       = "OAC para buckets S3"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# Cache Policy — SPA (HTML)
resource "aws_cloudfront_cache_policy" "spa_html" {
  name        = "${var.project_name}-spa-html"
  comment     = "Cache policy para HTML do SPA"
  default_ttl = 3600     # 1 hora
  max_ttl     = 86400    # 1 dia
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config {
      cookie_behavior = "none"
    }
    headers_config {
      header_behavior = "none"
    }
    query_strings_config {
      query_string_behavior = "none"
    }
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true
  }
}

# Cache Policy — Static Assets (JS, CSS, imagens com hash)
resource "aws_cloudfront_cache_policy" "static_assets" {
  name        = "${var.project_name}-static-assets"
  comment     = "Cache policy para assets estáticos com hash no nome"
  default_ttl = 31536000  # 365 dias
  max_ttl     = 31536000
  min_ttl     = 31536000

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config {
      cookie_behavior = "none"
    }
    headers_config {
      header_behavior = "none"
    }
    query_strings_config {
      query_string_behavior = "none"
    }
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true
  }
}

# Cache Policy — Media
resource "aws_cloudfront_cache_policy" "media" {
  name        = "${var.project_name}-media"
  comment     = "Cache policy para media assets"
  default_ttl = 2592000  # 30 dias
  max_ttl     = 2592000
  min_ttl     = 86400    # Mínimo 1 dia

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config {
      cookie_behavior = "none"
    }
    headers_config {
      header_behavior = "none"
    }
    query_strings_config {
      query_string_behavior = "whitelist"
      query_strings {
        items = ["v"]  # Apenas parâmetro de versão
      }
    }
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true
  }
}

# Cache Policy — API Config (cacheável por 60s)
resource "aws_cloudfront_cache_policy" "api_config" {
  name        = "${var.project_name}-api-config"
  comment     = "Cache policy para configurações cacheáveis da API"
  default_ttl = 60
  max_ttl     = 300
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config {
      cookie_behavior = "none"
    }
    headers_config {
      header_behavior = "whitelist"
      headers {
        items = ["Accept-Language"]
      }
    }
    query_strings_config {
      query_string_behavior = "all"
    }
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true
  }
}

# Origin Request Policy — API
resource "aws_cloudfront_origin_request_policy" "api" {
  name    = "${var.project_name}-api-origin-request"
  comment = "Forward necessário para a API"

  cookies_config {
    cookie_behavior = "all"
  }
  headers_config {
    header_behavior = "allViewer"
  }
  query_strings_config {
    query_string_behavior = "all"
  }
}

# Response Headers Policy
resource "aws_cloudfront_response_headers_policy" "security" {
  name    = "${var.project_name}-security-headers"
  comment = "Headers de segurança para todas as respostas"

  security_headers_config {
    content_type_options {
      override = true
    }

    frame_options {
      frame_option = "DENY"
      override     = true
    }

    referrer_policy {
      referrer_policy = "strict-origin-when-cross-origin"
      override        = true
    }

    strict_transport_security {
      access_control_max_age_sec = 63072000
      include_subdomains         = true
      preload                    = true
      override                   = true
    }

    xss_protection {
      mode_block = true
      protection = true
      override   = true
    }

    content_security_policy {
      content_security_policy = "default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.exemplo.com.br; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' https://fonts.gstatic.com; connect-src 'self' https://api.exemplo.com.br wss://ws.exemplo.com.br"
      override                = true
    }
  }

  custom_headers_config {
    items {
      header   = "Permissions-Policy"
      override = true
      value    = "camera=(), microphone=(), geolocation=(self)"
    }
    items {
      header   = "X-Powered-By"
      override = true
      value    = ""
    }
  }
}

# A DISTRIBUICAO PRINCIPAL
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "E-commerce Black Friday — ${var.project_name}"
  default_root_object = "index.html"
  price_class         = "PriceClass_All"
  http_version        = "http2and3"
  web_acl_id          = aws_wafv2_web_acl.main.arn
  aliases             = [var.domain_name, "www.${var.domain_name}"]

  # ============================================================
  # ORIGENS
  # ============================================================

  # Origem 1: S3 SPA (Frontend)
  origin {
    domain_name              = aws_s3_bucket.spa.bucket_regional_domain_name
    origin_id                = "s3-spa"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id

    origin_shield {
      enabled              = true
      origin_shield_region = "sa-east-1"
    }
  }

  # Origem 2: S3 Media
  origin {
    domain_name              = aws_s3_bucket.media.bucket_regional_domain_name
    origin_id                = "s3-media-primary"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id

    origin_shield {
      enabled              = true
      origin_shield_region = "sa-east-1"
    }
  }

  # Origem 3: S3 Media Replica (failover)
  origin {
    domain_name              = aws_s3_bucket.media_replica.bucket_regional_domain_name
    origin_id                = "s3-media-secondary"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  # Origem 4: ALB API
  origin {
    domain_name = aws_lb.api.dns_name
    origin_id   = "alb-api"

    custom_origin_config {
      http_port                = 80
      https_port               = 443
      origin_protocol_policy   = "https-only"
      origin_ssl_protocols     = ["TLSv1.2"]
      origin_keepalive_timeout = 60
      origin_read_timeout      = 30
    }

    custom_header {
      name  = "X-Custom-Origin-Secret"
      value = random_password.origin_secret.result
    }

    origin_shield {
      enabled              = true
      origin_shield_region = "sa-east-1"
    }
  }

  # Origem 5: ALB WebSocket
  origin {
    domain_name = aws_lb.websocket.dns_name
    origin_id   = "alb-websocket"

    custom_origin_config {
      http_port                = 80
      https_port               = 443
      origin_protocol_policy   = "https-only"
      origin_ssl_protocols     = ["TLSv1.2"]
      origin_keepalive_timeout = 60
      origin_read_timeout      = 60
    }

    custom_header {
      name  = "X-Custom-Origin-Secret"
      value = random_password.origin_secret.result
    }
  }

  # ============================================================
  # ORIGIN GROUPS (Failover)
  # ============================================================

  origin_group {
    origin_id = "media-failover-group"

    failover_criteria {
      status_codes = [500, 502, 503, 504, 403, 404]
    }

    member {
      origin_id = "s3-media-primary"
    }

    member {
      origin_id = "s3-media-secondary"
    }
  }

  # ============================================================
  # BEHAVIOR 1: /api/v1/config — Config cacheável
  # ============================================================
  ordered_cache_behavior {
    path_pattern     = "/api/v1/config"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "alb-api"
    compress         = true

    cache_policy_id          = aws_cloudfront_cache_policy.api_config.id
    origin_request_policy_id = aws_cloudfront_origin_request_policy.api.id

    viewer_protocol_policy = "https-only"

    lambda_function_association {
      event_type   = "viewer-request"
      lambda_arn   = aws_lambda_function.jwt_auth.qualified_arn
      include_body = false
    }
  }

  # ============================================================
  # BEHAVIOR 2: /api/* — API (sem cache)
  # ============================================================
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "alb-api"
    compress         = true

    cache_policy_id          = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"  # CachingDisabled
    origin_request_policy_id = aws_cloudfront_origin_request_policy.api.id

    viewer_protocol_policy = "https-only"

    lambda_function_association {
      event_type   = "viewer-request"
      lambda_arn   = aws_lambda_function.jwt_auth.qualified_arn
      include_body = true
    }
  }

  # ============================================================
  # BEHAVIOR 3: /ws/* — WebSocket
  # ============================================================
  ordered_cache_behavior {
    path_pattern     = "/ws/*"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "alb-websocket"
    compress         = false

    cache_policy_id          = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"  # CachingDisabled
    origin_request_policy_id = aws_cloudfront_origin_request_policy.api.id

    viewer_protocol_policy = "https-only"
  }

  # ============================================================
  # BEHAVIOR 4: /media/* — Media Assets (Origin Group failover)
  # ============================================================
  ordered_cache_behavior {
    path_pattern     = "/media/*"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD", "OPTIONS"]
    target_origin_id = "media-failover-group"
    compress         = true

    cache_policy_id            = aws_cloudfront_cache_policy.media.id
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security.id

    viewer_protocol_policy = "https-only"
  }

  # ============================================================
  # BEHAVIOR 5: /static/* — Static Assets (long cache)
  # ============================================================
  ordered_cache_behavior {
    path_pattern     = "/static/*"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-spa"
    compress         = true

    cache_policy_id            = aws_cloudfront_cache_policy.static_assets.id
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security.id

    viewer_protocol_policy = "https-only"

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.url_normalize.arn
    }
  }

  # ============================================================
  # BEHAVIOR DEFAULT: /* — SPA HTML
  # ============================================================
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-spa"
    compress         = true

    cache_policy_id            = aws_cloudfront_cache_policy.spa_html.id
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security.id

    viewer_protocol_policy = "redirect-to-https"

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.ab_test.arn
    }

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.geo_redirect.arn
    }
  }

  # ============================================================
  # CUSTOM ERROR RESPONSES (SPA routing)
  # ============================================================
  custom_error_response {
    error_code            = 403
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 10
  }

  custom_error_response {
    error_code            = 404
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 10
  }

  # ============================================================
  # SSL/TLS
  # ============================================================
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cloudfront.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # ============================================================
  # GEO RESTRICTION
  # ============================================================
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  # ============================================================
  # LOGGING
  # ============================================================
  logging_config {
    include_cookies = false
    bucket          = aws_s3_bucket.cf_logs.bucket_domain_name
    prefix          = "cloudfront/"
  }

  tags = {
    Name = "${var.project_name}-distribution"
  }

  depends_on = [aws_acm_certificate_validation.cloudfront]
}

# S3 Bucket Policies (permitir acesso do CloudFront via OAC)
resource "aws_s3_bucket_policy" "spa" {
  bucket = aws_s3_bucket.spa.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCloudFrontOAC"
        Effect = "Allow"
        Principal = {
          Service = "cloudfront.amazonaws.com"
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.spa.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.main.arn
          }
        }
      }
    ]
  })
}

resource "aws_s3_bucket_policy" "media" {
  bucket = aws_s3_bucket.media.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCloudFrontOAC"
        Effect = "Allow"
        Principal = {
          Service = "cloudfront.amazonaws.com"
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.media.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.main.arn
          }
        }
      }
    ]
  })
}

# Route 53 Record
resource "aws_route53_record" "main" {
  zone_id = var.hosted_zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}

resource "aws_route53_record" "www" {
  zone_id = var.hosted_zone_id
  name    = "www.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}
```

#### Fase 9: CloudWatch — Observabilidade

```hcl
# ==============================================================================
# monitoring.tf — CloudWatch Dashboard e Alarmes
# ==============================================================================

resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "${var.project_name}-black-friday"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "CloudFront - Requests"
          region = "us-east-1"
          metrics = [
            ["AWS/CloudFront", "Requests", "DistributionId", aws_cloudfront_distribution.main.id, "Region", "Global", { stat = "Sum", period = 60 }]
          ]
          view = "timeSeries"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "CloudFront - Error Rate"
          region = "us-east-1"
          metrics = [
            ["AWS/CloudFront", "4xxErrorRate", "DistributionId", aws_cloudfront_distribution.main.id, "Region", "Global", { stat = "Average", period = 60, color = "#ff9900" }],
            ["AWS/CloudFront", "5xxErrorRate", "DistributionId", aws_cloudfront_distribution.main.id, "Region", "Global", { stat = "Average", period = 60, color = "#d13212" }]
          ]
          view = "timeSeries"
          yAxis = { left = { min = 0, max = 10 } }
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 12
        height = 6
        properties = {
          title  = "CloudFront - Cache Hit Rate"
          region = "us-east-1"
          metrics = [
            ["AWS/CloudFront", "CacheHitRate", "DistributionId", aws_cloudfront_distribution.main.id, "Region", "Global", { stat = "Average", period = 60 }]
          ]
          view = "timeSeries"
          yAxis = { left = { min = 0, max = 100 } }
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 6
        width  = 12
        height = 6
        properties = {
          title  = "CloudFront - Bytes Downloaded"
          region = "us-east-1"
          metrics = [
            ["AWS/CloudFront", "BytesDownloaded", "DistributionId", aws_cloudfront_distribution.main.id, "Region", "Global", { stat = "Sum", period = 60 }]
          ]
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
          title  = "ECS API - CPU e Memória"
          metrics = [
            ["AWS/ECS", "CPUUtilization", "ServiceName", aws_ecs_service.api.name, "ClusterName", aws_ecs_cluster.main.name, { stat = "Average", period = 60 }],
            ["AWS/ECS", "MemoryUtilization", "ServiceName", aws_ecs_service.api.name, "ClusterName", aws_ecs_cluster.main.name, { stat = "Average", period = 60 }]
          ]
          view = "timeSeries"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 12
        width  = 12
        height = 6
        properties = {
          title  = "ALB - Latência e Healthy Hosts"
          metrics = [
            ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", aws_lb.api.arn_suffix, { stat = "p99", period = 60 }],
            ["AWS/ApplicationELB", "HealthyHostCount", "TargetGroup", aws_lb_target_group.api.arn_suffix, "LoadBalancer", aws_lb.api.arn_suffix, { stat = "Average", period = 60 }]
          ]
          view = "timeSeries"
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 18
        width  = 12
        height = 6
        properties = {
          title  = "WAF - Blocked Requests"
          region = "us-east-1"
          metrics = [
            ["AWS/WAFV2", "BlockedRequests", "WebACL", "${var.project_name}-waf", "Region", "us-east-1", "Rule", "ALL", { stat = "Sum", period = 60 }]
          ]
          view = "timeSeries"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 18
        width  = 12
        height = 6
        properties = {
          title  = "ECS - Running Task Count"
          metrics = [
            ["ECS/ContainerInsights", "RunningTaskCount", "ServiceName", aws_ecs_service.api.name, "ClusterName", aws_ecs_cluster.main.name, { stat = "Average", period = 60 }]
          ]
          view = "timeSeries"
        }
      }
    ]
  })
}

# Alarmes
resource "aws_cloudwatch_metric_alarm" "cf_5xx_high" {
  provider = aws.us_east_1

  alarm_name          = "${var.project_name}-cf-5xx-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "5xxErrorRate"
  namespace           = "AWS/CloudFront"
  period              = 60
  statistic           = "Average"
  threshold           = 5
  alarm_description   = "CloudFront 5xx error rate acima de 5% por 3 minutos"
  treat_missing_data  = "notBreaching"

  dimensions = {
    DistributionId = aws_cloudfront_distribution.main.id
    Region         = "Global"
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "cf_cache_hit_low" {
  provider = aws.us_east_1

  alarm_name          = "${var.project_name}-cf-cache-hit-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 5
  metric_name         = "CacheHitRate"
  namespace           = "AWS/CloudFront"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "Cache hit rate abaixo de 80% por 25 minutos"
  treat_missing_data  = "notBreaching"

  dimensions = {
    DistributionId = aws_cloudfront_distribution.main.id
    Region         = "Global"
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "ecs_cpu_high" {
  alarm_name          = "${var.project_name}-ecs-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 60
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "ECS CPU acima de 80% por 3 minutos"

  dimensions = {
    ServiceName = aws_ecs_service.api.name
    ClusterName = aws_ecs_cluster.main.name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "alb_latency_high" {
  alarm_name          = "${var.project_name}-alb-latency-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "TargetResponseTime"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  extended_statistic  = "p99"
  threshold           = 2
  alarm_description   = "ALB P99 latency acima de 2s por 3 minutos"

  dimensions = {
    LoadBalancer = aws_lb.api.arn_suffix
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "waf_blocked_spike" {
  provider = aws.us_east_1

  alarm_name          = "${var.project_name}-waf-blocked-spike"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "BlockedRequests"
  namespace           = "AWS/WAFV2"
  period              = 300
  statistic           = "Sum"
  threshold           = 10000
  alarm_description   = "WAF bloqueou mais de 10000 requests em 10 minutos"

  dimensions = {
    WebACL = "${var.project_name}-waf"
    Region = "us-east-1"
    Rule   = "ALL"
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

# SNS Topic para alertas
resource "aws_sns_topic" "alerts" {
  name = "${var.project_name}-alerts"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = "oncall@exemplo.com.br"
}
```

#### Fase 10: Load Testing com k6

```javascript
// ==============================================================================
// load-test.js — Script k6 para teste de carga Black Friday
// ==============================================================================
// Executar: k6 run --vus 1000 --duration 30m load-test.js

import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Counter, Rate, Trend } from 'k6/metrics';

// Métricas customizadas
const cacheHits = new Counter('cache_hits');
const cacheMisses = new Counter('cache_misses');
const errorRate = new Rate('errors');
const spaLatency = new Trend('spa_latency');
const apiLatency = new Trend('api_latency');
const mediaLatency = new Trend('media_latency');

// Configuração de stages (simula tráfego Black Friday)
export const options = {
  stages: [
    // Warm-up gradual
    { duration: '2m', target: 100 },     // Ramp up para 100 VUs
    { duration: '3m', target: 500 },     // Ramp up para 500 VUs
    { duration: '5m', target: 1000 },    // Ramp up para 1000 VUs

    // Pico sustentado (simula abertura da Black Friday)
    { duration: '10m', target: 2000 },   // PICO: 2000 VUs

    // Tráfego sustentado
    { duration: '5m', target: 1500 },    // Sustentado
    { duration: '3m', target: 500 },     // Cool down
    { duration: '2m', target: 0 },       // Ramp down
  ],

  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<2000'],  // P95 < 500ms, P99 < 2s
    errors: ['rate<0.01'],                             // < 1% erro
    spa_latency: ['p(95)<200'],                        // SPA P95 < 200ms
    api_latency: ['p(95)<1000'],                       // API P95 < 1s
    media_latency: ['p(95)<300'],                      // Media P95 < 300ms
  },
};

const BASE_URL = 'https://loja.exemplo.com.br';
const API_TOKEN = __ENV.API_TOKEN || 'test-token';

// Dados de teste
const PRODUCT_IDS = Array.from({ length: 1000 }, (_, i) => i + 1);
const CATEGORIES = ['eletronicos', 'moda', 'casa', 'esportes', 'livros'];

export default function () {
  // Simular comportamento real de usuário

  group('01 - Página Inicial (SPA)', () => {
    const res = http.get(`${BASE_URL}/`);
    spaLatency.add(res.timings.duration);

    check(res, {
      'status 200': (r) => r.status === 200,
      'has content': (r) => r.body.length > 0,
      'cache hit': (r) => {
        const hit = r.headers['X-Cache'] === 'Hit from cloudfront';
        if (hit) cacheHits.add(1);
        else cacheMisses.add(1);
        return hit;
      },
    }) || errorRate.add(1);

    sleep(0.5);
  });

  group('02 - Assets Estáticos', () => {
    const assets = [
      `${BASE_URL}/static/js/main.abc123.js`,
      `${BASE_URL}/static/css/app.def456.css`,
      `${BASE_URL}/static/images/logo.svg`,
    ];

    const responses = http.batch(assets.map(url => ['GET', url]));
    responses.forEach(res => {
      check(res, {
        'asset loaded': (r) => r.status === 200,
        'asset cached': (r) => r.headers['X-Cache'] === 'Hit from cloudfront',
      });
    });

    sleep(0.3);
  });

  group('03 - Listagem de Produtos (API)', () => {
    const category = CATEGORIES[Math.floor(Math.random() * CATEGORIES.length)];
    const res = http.get(`${BASE_URL}/api/v1/products?category=${category}&page=1&limit=20`, {
      headers: { 'Authorization': `Bearer ${API_TOKEN}` },
    });
    apiLatency.add(res.timings.duration);

    check(res, {
      'products loaded': (r) => r.status === 200,
      'has products': (r) => {
        try { return JSON.parse(r.body).data.length > 0; }
        catch { return false; }
      },
    }) || errorRate.add(1);

    sleep(1);
  });

  group('04 - Detalhes do Produto', () => {
    const productId = PRODUCT_IDS[Math.floor(Math.random() * PRODUCT_IDS.length)];
    const res = http.get(`${BASE_URL}/api/v1/products/${productId}`, {
      headers: { 'Authorization': `Bearer ${API_TOKEN}` },
    });
    apiLatency.add(res.timings.duration);

    check(res, {
      'product details loaded': (r) => r.status === 200,
    }) || errorRate.add(1);

    sleep(0.5);
  });

  group('05 - Imagens do Produto', () => {
    const productId = PRODUCT_IDS[Math.floor(Math.random() * PRODUCT_IDS.length)];
    const images = [
      `${BASE_URL}/media/products/${productId}/main.webp`,
      `${BASE_URL}/media/products/${productId}/thumb-1.webp`,
      `${BASE_URL}/media/products/${productId}/thumb-2.webp`,
    ];

    const responses = http.batch(images.map(url => ['GET', url]));
    responses.forEach(res => {
      mediaLatency.add(res.timings.duration);
      check(res, {
        'image loaded': (r) => r.status === 200 || r.status === 304,
      });
    });

    sleep(0.5);
  });

  group('06 - Adicionar ao Carrinho', () => {
    const productId = PRODUCT_IDS[Math.floor(Math.random() * PRODUCT_IDS.length)];
    const res = http.post(
      `${BASE_URL}/api/v1/cart/items`,
      JSON.stringify({ productId, quantity: 1 }),
      {
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${API_TOKEN}`,
        },
      }
    );
    apiLatency.add(res.timings.duration);

    check(res, {
      'item added': (r) => r.status === 200 || r.status === 201,
    }) || errorRate.add(1);

    sleep(2);
  });

  group('07 - Config Endpoint (Cached)', () => {
    const res = http.get(`${BASE_URL}/api/v1/config`);
    check(res, {
      'config loaded': (r) => r.status === 200,
    });
  });

  sleep(Math.random() * 3 + 1);  // Think time aleatório 1-4s
}

export function handleSummary(data) {
  return {
    'summary.json': JSON.stringify(data),
    stdout: textSummary(data, { indent: ' ', enableColors: true }),
  };
}

function textSummary(data) {
  const metrics = data.metrics;
  return `
========================================
  BLACK FRIDAY LOAD TEST RESULTS
========================================

Requests Total:     ${metrics.http_reqs.values.count}
Req/s:              ${metrics.http_reqs.values.rate.toFixed(0)}
Error Rate:         ${(metrics.errors?.values?.rate * 100 || 0).toFixed(2)}%

Latency P50:        ${metrics.http_req_duration.values.med.toFixed(0)}ms
Latency P95:        ${metrics.http_req_duration.values['p(95)'].toFixed(0)}ms
Latency P99:        ${metrics.http_req_duration.values['p(99)'].toFixed(0)}ms

Cache Hits:         ${metrics.cache_hits?.values?.count || 0}
Cache Misses:       ${metrics.cache_misses?.values?.count || 0}

SPA P95:            ${metrics.spa_latency?.values?.['p(95)']?.toFixed(0) || 'N/A'}ms
API P95:            ${metrics.api_latency?.values?.['p(95)']?.toFixed(0) || 'N/A'}ms
Media P95:          ${metrics.media_latency?.values?.['p(95)']?.toFixed(0) || 'N/A'}ms
========================================
  `;
}
```

#### Go-Live Checklist (Black Friday)

```markdown
## Checklist de Go-Live — Black Friday E-commerce

### Pré-evento (1 semana antes)
- [ ] Certificados SSL/TLS validados e sem expiração próxima
- [ ] Origin Shield habilitado em sa-east-1
- [ ] Cache policies otimizadas por behavior
- [ ] Cache warming executado (URLs principais)
- [ ] WAF rules testadas em modo COUNT primeiro, depois BLOCK
- [ ] Shield Advanced habilitado com DRT (DDoS Response Team) configurado
- [ ] Rate limits validados (não bloquear tráfego legítimo)
- [ ] Bot Control testado com bots legítimos (Googlebot etc)
- [ ] Lambda@Edge publicada e testada em staging
- [ ] CloudFront Functions testadas com test events

### Pré-evento (1 dia antes)
- [ ] ECS Auto Scaling pre-warm executado (min capacity = 100)
- [ ] Load test final com 50% da carga esperada
- [ ] Verificar que todas as origens respondem corretamente
- [ ] CloudWatch dashboard aberto em tela dedicada
- [ ] Alarmes SNS testados (enviar test notification)
- [ ] Runbook de incidentes revisado pela equipe
- [ ] Contato AWS Premium Support verificado
- [ ] DNS TTL reduzido para 60s (permitir rollback rápido)

### Durante o evento
- [ ] Monitorar cache hit rate (meta: >95%)
- [ ] Monitorar 5xx error rate (meta: <0.1%)
- [ ] Monitorar latência P99 (meta: <200ms)
- [ ] Monitorar WAF blocked requests (buscar falsos positivos)
- [ ] Monitorar ECS task count e CPU
- [ ] Monitorar ALB healthy host count
- [ ] Manter canal de comunicação aberto (Slack/Teams)

### Pós-evento
- [ ] Reverter ECS min capacity para normal
- [ ] Analisar logs de WAF blocked requests
- [ ] Gerar relatório de performance
- [ ] Calcular custo total do evento
- [ ] Documentar lições aprendidas
- [ ] Reverter DNS TTL para valor normal (300s)
```

### Validação

```bash
# Verificar distribuição
aws cloudfront get-distribution --id $DIST_ID \
  --query 'Distribution.{Status:Status,DomainName:DomainName}' \
  --output table

# Verificar comportamentos
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.CacheBehaviors.Items[].PathPattern' \
  --output table

# Verificar WAF
aws wafv2 get-web-acl --name ecommerce-bf-waf --scope CLOUDFRONT \
  --id $WAF_ID --region us-east-1 \
  --query 'WebACL.Rules[].Name' --output table

# Testar cada behavior
echo "=== Testando SPA ==="
curl -sI https://loja.exemplo.com.br/ | grep -E "x-cache|age|x-amz"

echo "=== Testando API ==="
curl -sI https://loja.exemplo.com.br/api/v1/products | grep -E "x-cache|status"

echo "=== Testando Media ==="
curl -sI https://loja.exemplo.com.br/media/hero.webp | grep -E "x-cache|age"

echo "=== Testando Static ==="
curl -sI https://loja.exemplo.com.br/static/js/main.js | grep -E "x-cache|age"

echo "=== Testando Config (cache 60s) ==="
curl -sI https://loja.exemplo.com.br/api/v1/config | grep -E "x-cache|age"

# Rodar load test básico
k6 run --vus 10 --duration 1m load-test.js
```

### O Que Aprendemos

1. **Separação de behaviors** permite otimizar cada tipo de conteúdo individualmente
2. **Origin Groups** garantem failover automático sem intervenção manual
3. **Origin Shield** reduz dramaticamente a carga nas origens durante picos
4. **Pre-warming** do Auto Scaling evita cold starts durante o pico
5. **Scheduled Scaling** garante capacidade antes do tráfego chegar
6. **WAF com Bot Control** protege contra scrapers e bots maliciosos
7. **Custom origin headers** impedem bypass direto ao ALB
8. **Observabilidade** proativa com dashboards e alarmes é essencial

### Dica Expert

> **Converse com o AWS Solutions Architect antes da Black Friday.** Para clientes com AWS
> Premium Support, é possível agendar uma sessão de "Infrastructure Event Management" (IEM)
> onde a AWS revisa toda a sua arquitetura, faz recomendações e monitora ativamente durante
> o evento. Para CloudFront especificamente, solicite o "pre-warming" dos edge caches nas
> regiões onde você espera mais tráfego. Isso pode ser feito via suporte e garante que o
> conteúdo estático já esteja cacheado quando os primeiros usuários chegarem.

---

## Desafio 57: Multi-Region Active-Active (Level 400)

### Classificação

| Atributo         | Valor                                                  |
|------------------|--------------------------------------------------------|
| **Nível**        | 400 — Expert                                           |
| **Tipo**         | Arquitetura Global                                     |
| **Tempo**        | 3-4 horas                                              |
| **Serviços**     | CloudFront, Route 53, ALB, ECS, DynamoDB Global Tables |
| **Complexidade** | Muito Alta                                             |

### Objetivo

Implementar uma aplicação globalmente distribuída com latência inferior a 100ms para
usuários em qualquer continente, usando arquitetura active-active em múltiplas regiões.

### Cenário Real

> Sua empresa está expandindo globalmente. Clientes na Europa reclamam de latência de 400ms+
> para APIs que rodam em sa-east-1. O objetivo é garantir que nenhum usuário, em qualquer
> lugar do mundo, experimente mais que 100ms de latência na camada de CDN + API.

### Diagrama de Arquitetura

```
                              ┌─────────────────────┐
                              │    USUÁRIOS GLOBAIS  │
                              │                      │
                              │  SA  NA  EU  AP  AF  │
                              └──────────┬──────────┘
                                         │
                                         ▼
                          ┌─────────────────────────────┐
                          │       Route 53 DNS           │
                          │   app.exemplo.com.br         │
                          │                              │
                          │   Routing Policy:            │
                          │   Latency-Based Routing      │
                          │                              │
                          │   Health Checks ativos       │
                          │   em ambas as regiões        │
                          └──────┬──────────────┬───────┘
                                 │              │
                    ┌────────────┘              └────────────┐
                    │                                        │
                    ▼                                        ▼
    ┌───────────────────────────────┐      ┌───────────────────────────────┐
    │    CloudFront Distribution    │      │    CloudFront Distribution    │
    │    (Região Americas)          │      │    (Região EMEA/APAC)         │
    │                               │      │                               │
    │  Aliases:                     │      │  Aliases:                     │
    │  app-us.exemplo.com.br        │      │  app-eu.exemplo.com.br        │
    │                               │      │                               │
    │  Origin Shield: us-east-1     │      │  Origin Shield: eu-west-1     │
    │                               │      │                               │
    │  WAF: Regional rules          │      │  WAF: Regional rules + GDPR   │
    │                               │      │                               │
    │  Functions: Geo, A/B, Auth    │      │  Functions: Geo, A/B, Auth    │
    └───────────────┬───────────────┘      └───────────────┬───────────────┘
                    │                                      │
                    ▼                                      ▼
    ┌───────────────────────────────┐      ┌───────────────────────────────┐
    │       us-east-1               │      │       eu-west-1               │
    │                               │      │                               │
    │  ┌─────────┐  ┌───────────┐  │      │  ┌─────────┐  ┌───────────┐  │
    │  │   ALB   │  │    S3     │  │      │  │   ALB   │  │    S3     │  │
    │  │  (API)  │  │  (SPA)   │  │      │  │  (API)  │  │ (Replica) │  │
    │  └────┬────┘  └───────────┘  │      │  └────┬────┘  └───────────┘  │
    │       │                      │      │       │                      │
    │       ▼                      │      │       ▼                      │
    │  ┌─────────┐                 │      │  ┌─────────┐                 │
    │  │   ECS   │                 │      │  │   ECS   │                 │
    │  │ Fargate │                 │      │  │ Fargate │                 │
    │  │ 10-100  │                 │      │  │ 10-100  │                 │
    │  └────┬────┘                 │      │  └────┬────┘                 │
    │       │                      │      │       │                      │
    │       ▼                      │      │       ▼                      │
    │  ┌─────────────────────┐     │      │  ┌─────────────────────┐     │
    │  │  DynamoDB            │◄───┼──────┼─►│  DynamoDB            │     │
    │  │  Global Tables       │    │      │  │  Global Tables       │     │
    │  │  (Active-Active)     │    │      │  │  (Active-Active)     │     │
    │  └─────────────────────┘     │      │  └─────────────────────┘     │
    │                               │      │                               │
    │  ┌─────────────────────┐     │      │  ┌─────────────────────┐     │
    │  │  ElastiCache Redis  │     │      │  │  ElastiCache Redis  │     │
    │  │  (Session Store)    │     │      │  │  (Session Store)    │     │
    │  └─────────────────────┘     │      │  └─────────────────────┘     │
    └───────────────────────────────┘      └───────────────────────────────┘
                    │                                      │
                    └──────────────┬───────────────────────┘
                                   │
                              S3 Cross-Region
                              Replication (bidirectional)
```

### Alternativa: Single Distribution com Lambda@Edge

```
                              ┌─────────────────────┐
                              │    USUÁRIOS GLOBAIS  │
                              └──────────┬──────────┘
                                         │
                                         ▼
                          ┌─────────────────────────────┐
                          │   CloudFront (SINGLE DIST)  │
                          │   app.exemplo.com.br         │
                          │                              │
                          │   Lambda@Edge:               │
                          │   origin-request event       │
                          │   Seleciona origem pela      │
                          │   geolocalização do viewer   │
                          └──────────────┬──────────────┘
                                         │
                        Lambda@Edge decide:
                        │                              │
            cloudfront-viewer-country     cloudfront-viewer-country
                 in [BR,US,CA,MX]             in [DE,FR,GB,PT]
                        │                              │
                        ▼                              ▼
              ┌──────────────────┐           ┌──────────────────┐
              │  ALB us-east-1   │           │  ALB eu-west-1   │
              └──────────────────┘           └──────────────────┘
```

### Passo a Passo

#### Terraform Multi-Region

```hcl
# ==============================================================================
# multi-region/main.tf — Infraestrutura Multi-Region
# ==============================================================================

locals {
  regions = {
    primary = {
      region = "us-east-1"
      name   = "americas"
      alias  = "app-us.exemplo.com.br"
    }
    secondary = {
      region = "eu-west-1"
      name   = "emea"
      alias  = "app-eu.exemplo.com.br"
    }
  }
}

# Módulo reutilizável para cada região
module "region" {
  source   = "./modules/regional-stack"
  for_each = local.regions

  region          = each.value.region
  region_name     = each.value.name
  cf_alias        = each.value.alias
  project_name    = var.project_name
  vpc_cidr        = each.key == "primary" ? "10.0.0.0/16" : "10.1.0.0/16"
  ecs_min         = 10
  ecs_max         = 100
  domain_name     = var.domain_name
  hosted_zone_id  = var.hosted_zone_id
}
```

```hcl
# ==============================================================================
# multi-region/dynamodb-global.tf — DynamoDB Global Tables
# ==============================================================================

resource "aws_dynamodb_table" "main" {
  provider = aws.us_east_1

  name         = "${var.project_name}-data"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  attribute {
    name = "GSI1PK"
    type = "S"
  }

  attribute {
    name = "GSI1SK"
    type = "S"
  }

  global_secondary_index {
    name            = "GSI1"
    hash_key        = "GSI1PK"
    range_key       = "GSI1SK"
    projection_type = "ALL"
  }

  # Replica na Europa
  replica {
    region_name = "eu-west-1"
    point_in_time_recovery {
      enabled = true
    }
  }

  point_in_time_recovery {
    enabled = true
  }

  server_side_encryption {
    enabled = true
  }

  tags = {
    Name = "${var.project_name}-global-table"
  }
}
```

```hcl
# ==============================================================================
# multi-region/s3-replication.tf — S3 Cross-Region Replication
# ==============================================================================

# Bucket primário (us-east-1)
resource "aws_s3_bucket" "spa_primary" {
  provider = aws.us_east_1
  bucket   = "${var.project_name}-spa-us"
}

resource "aws_s3_bucket_versioning" "spa_primary" {
  provider = aws.us_east_1
  bucket   = aws_s3_bucket.spa_primary.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Bucket réplica (eu-west-1)
resource "aws_s3_bucket" "spa_replica" {
  provider = aws.eu_west_1
  bucket   = "${var.project_name}-spa-eu"
}

resource "aws_s3_bucket_versioning" "spa_replica" {
  provider = aws.eu_west_1
  bucket   = aws_s3_bucket.spa_replica.id
  versioning_configuration {
    status = "Enabled"
  }
}

# IAM Role para replicação
resource "aws_iam_role" "replication" {
  name = "${var.project_name}-s3-replication"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "s3.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "replication" {
  name = "${var.project_name}-s3-replication-policy"
  role = aws_iam_role.replication.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetReplicationConfiguration",
          "s3:ListBucket"
        ]
        Resource = aws_s3_bucket.spa_primary.arn
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObjectVersionForReplication",
          "s3:GetObjectVersionAcl",
          "s3:GetObjectVersionTagging"
        ]
        Resource = "${aws_s3_bucket.spa_primary.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:ReplicateObject",
          "s3:ReplicateDelete",
          "s3:ReplicateTags"
        ]
        Resource = "${aws_s3_bucket.spa_replica.arn}/*"
      }
    ]
  })
}

# Regra de replicação
resource "aws_s3_bucket_replication_configuration" "spa" {
  provider = aws.us_east_1

  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.spa_primary.id

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.spa_replica.arn
      storage_class = "STANDARD"
    }
  }

  depends_on = [
    aws_s3_bucket_versioning.spa_primary,
    aws_s3_bucket_versioning.spa_replica
  ]
}
```

```hcl
# ==============================================================================
# multi-region/route53.tf — Route 53 Latency-Based Routing
# ==============================================================================

# Health Checks
resource "aws_route53_health_check" "us" {
  fqdn              = module.region["primary"].alb_dns_name
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 10

  tags = {
    Name = "${var.project_name}-health-us"
  }
}

resource "aws_route53_health_check" "eu" {
  fqdn              = module.region["secondary"].alb_dns_name
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 10

  tags = {
    Name = "${var.project_name}-health-eu"
  }
}

# Latency-based records
resource "aws_route53_record" "app_us" {
  zone_id = var.hosted_zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  set_identifier = "us-east-1"

  latency_routing_policy {
    region = "us-east-1"
  }

  alias {
    name                   = module.region["primary"].cloudfront_domain
    zone_id                = module.region["primary"].cloudfront_zone_id
    evaluate_target_health = true
  }

  health_check_id = aws_route53_health_check.us.id
}

resource "aws_route53_record" "app_eu" {
  zone_id = var.hosted_zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  set_identifier = "eu-west-1"

  latency_routing_policy {
    region = "eu-west-1"
  }

  alias {
    name                   = module.region["secondary"].cloudfront_domain
    zone_id                = module.region["secondary"].cloudfront_zone_id
    evaluate_target_health = true
  }

  health_check_id = aws_route53_health_check.eu.id
}
```

### Lambda@Edge para Seleção de Origem (Alternativa Single Distribution)

```javascript
// origin-selector.js — Lambda@Edge (origin-request)
'use strict';

const ORIGINS = {
  americas: {
    domainName: 'api-us.exemplo.com.br',
    region: 'us-east-1',
    countries: ['BR', 'US', 'CA', 'MX', 'AR', 'CL', 'CO', 'PE', 'VE', 'EC', 'UY', 'PY', 'BO']
  },
  emea: {
    domainName: 'api-eu.exemplo.com.br',
    region: 'eu-west-1',
    countries: ['DE', 'FR', 'GB', 'IT', 'ES', 'PT', 'NL', 'BE', 'IE', 'PL', 'SE', 'NO', 'DK', 'FI', 'AT', 'CH', 'ZA', 'NG', 'KE', 'EG', 'IL', 'AE', 'SA', 'IN']
  }
};

exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;

  // Obter país do viewer
  const country = headers['cloudfront-viewer-country']
    ? headers['cloudfront-viewer-country'][0].value
    : 'US';

  // Determinar a melhor origem
  let selectedOrigin = ORIGINS.americas; // default

  for (const [region, config] of Object.entries(ORIGINS)) {
    if (config.countries.includes(country)) {
      selectedOrigin = config;
      break;
    }
  }

  // Modificar a origem da requisição
  request.origin = {
    custom: {
      domainName: selectedOrigin.domainName,
      port: 443,
      protocol: 'https',
      sslProtocols: ['TLSv1.2'],
      readTimeout: 30,
      keepaliveTimeout: 60,
      path: ''
    }
  };

  // Atualizar o Host header
  request.headers['host'] = [{ key: 'host', value: selectedOrigin.domainName }];

  // Adicionar header de debug
  request.headers['x-origin-region'] = [{ key: 'X-Origin-Region', value: selectedOrigin.region }];

  return request;
};
```

### Validação

```bash
# Testar latência de diferentes regiões com curl
echo "=== Teste US East ==="
curl -o /dev/null -s -w "DNS: %{time_namelookup}s | Connect: %{time_connect}s | TTFB: %{time_starttransfer}s | Total: %{time_total}s\n" \
  https://app.exemplo.com.br/api/v1/health

# Testar resolução DNS (qual região é selecionada)
dig app.exemplo.com.br +short

# Verificar Global Tables
aws dynamodb describe-table --table-name ecommerce-bf-data \
  --query 'Table.Replicas' --output table

# Verificar replicação S3
aws s3api head-object --bucket ecommerce-bf-spa-eu --key index.html

# Testar failover: desabilitar health check US
aws route53 update-health-check \
  --health-check-id $HC_ID \
  --disabled
```

### O Que Aprendemos

1. **Latency-based routing** direciona usuários para a região mais próxima automaticamente
2. **DynamoDB Global Tables** fornecem replicação active-active com consistência eventual
3. **S3 Cross-Region Replication** mantém assets sincronizados entre regiões
4. **Health checks** com failover automático garantem disponibilidade global
5. **Lambda@Edge para origin selection** é uma alternativa poderosa com single distribution

### Dica Expert

> **Cuidado com conflitos de escrita em Global Tables.** O DynamoDB usa "last writer wins"
> para resolver conflitos. Para dados onde a ordem importa (carrinho de compras, estoque),
> considere usar um campo de versão ou timestamp e resolver conflitos na aplicação.
> Para session management, use ElastiCache Global Datastore com Redis ao invés de DynamoDB
> para garantir consistência de sessão quando o usuário muda de região.

---

## Desafio 58: Migração de CDN (Level 400)

### Classificação

| Atributo         | Valor                                         |
|------------------|-----------------------------------------------|
| **Nível**        | 400 — Expert                                  |
| **Tipo**         | Migração                                      |
| **Tempo**        | 2-4 semanas (projeto completo)                |
| **Serviços**     | CloudFront, WAF, Route 53, ACM                |
| **Complexidade** | Alta (risco operacional)                      |

### Objetivo

Migrar uma aplicação em produção de Akamai (ou Cloudflare) para Amazon CloudFront
com **zero downtime** e rollback instantâneo.

### Cenário Real

> Seu CTO decidiu migrar de Akamai para CloudFront para consolidar a infraestrutura na AWS,
> reduzir custos de licenciamento e simplificar operações. O site tem 50 milhões de pageviews
> por mês e qualquer segundo de indisponibilidade é inaceitável. A Akamai precisa ser
> descomissionada em 90 dias.

### Plano de Migração em 8 Fases

```
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌─────────┐
│ Fase 1  │→ │ Fase 2  │→ │ Fase 3  │→ │ Fase 4   │→ │ Fase 5  │→ │ Fase 6   │→ │ Fase 7  │→ │ Fase 8  │
│         │  │         │  │         │  │          │  │         │  │          │  │         │  │         │
│ ASSESS  │  │ DESIGN  │  │ BUILD   │  │ VALIDATE │  │ WARM    │  │ CUTOVER  │  │ MONITOR │  │ CLEANUP │
│         │  │         │  │         │  │          │  │         │  │          │  │         │  │         │
│ 1 sem   │  │ 1 sem   │  │ 2 sem   │  │ 1 sem    │  │ 2 dias  │  │ 1 dia    │  │ 2 sem   │  │ 1 sem   │
└─────────┘  └─────────┘  └─────────┘  └──────────┘  └─────────┘  └──────────┘  └─────────┘  └─────────┘
```

#### Fase 1: Assessment (Levantamento)

```bash
# Exportar configuração atual da CDN (exemplo Akamai)
# Listar todas as propriedades
akamai property-manager list-properties --contractId ctr_XXXXX --groupId grp_XXXXX

# Exportar configuração
akamai property-manager get-property-rules --property prp_XXXXX --version latest > akamai-rules.json

# Listar todos os hostnames
akamai property-manager list-hostnames --property prp_XXXXX

# Verificar features em uso
cat akamai-rules.json | jq '.rules.behaviors[].name' | sort | uniq -c | sort -rn
```

**Inventário a coletar:**
- Lista de hostnames (CNAMEs)
- Regras de cache (TTLs por tipo de conteúdo)
- Regras de redirect
- Regras de rewrite
- Configurações de WAF
- Edge workers / Cloudflare Workers
- Page Rules
- SSL/TLS settings
- Origin configurations
- Rate limiting rules
- Bot management rules
- Custom error pages
- Geo-restriction rules

#### Feature Mapping Table

| Feature Akamai/Cloudflare        | Equivalente CloudFront                           | Notas                                              |
|----------------------------------|--------------------------------------------------|----------------------------------------------------|
| Edge Workers / CF Workers        | Lambda@Edge ou CloudFront Functions              | CF Functions para lógica simples, L@E para complexa|
| Property Rules / Page Rules      | Cache Behaviors + Policies                       | 25 behaviors por distribuição (soft limit)         |
| Site Shield                      | Origin Shield                                    | Conceito similar, config mais simples              |
| Web App Firewall                 | AWS WAF v2                                       | Managed rules + custom rules                       |
| Bot Manager                      | AWS WAF Bot Control                              | Common + Targeted inspection levels                |
| Rate Limiting                    | AWS WAF Rate-based rules                         | Até 10.000 req/5min por IP                         |
| DDoS Protection                  | AWS Shield Standard/Advanced                     | Standard grátis, Advanced para L7                  |
| Image & Video Manager           | Lambda@Edge + S3 ou CloudFront Image Optimization| Não há equivalente nativo completo                 |
| EdgeConnect / Argo Tunnel        | CloudFront Origin custom headers                 | Custom header secret para validação                |
| Akamai NetStorage / R2           | Amazon S3 + OAC                                  | OAC é mais seguro que OAI                          |
| mPulse / Web Analytics           | CloudFront Standard Logs + Real-time Logs        | Integrar com Athena ou Kinesis                     |
| Global Traffic Mgmt / Load Bal   | Route 53 + CloudFront Origin Groups              | Latency routing + failover                         |
| Certificate Management           | ACM (Certificate Manager)                        | Grátis, renovação automática                       |
| Purge / Cache Invalidation       | CloudFront Invalidation API                      | 1000 paths grátis/mês, depois $0.005/path          |
| Token Authentication             | CloudFront Signed URLs/Cookies                   | Ou Lambda@Edge para custom tokens                  |
| True Client IP                   | CloudFront-Viewer-Address header                 | Disponível nativamente                             |

#### Fase 2: Design

```
ANTES (Akamai):                        DEPOIS (CloudFront):

Akamai Property                        CloudFront Distribution
├── Default Rule                       ├── Default Behavior (/*) → S3 SPA
│   ├── Caching: 1h                    │   ├── Cache Policy: CachingOptimized
│   └── Origin: origin.site.com        │   └── Response Headers Policy: Security
│                                      │
├── Rule: /api/*                       ├── Behavior: /api/*  → ALB
│   ├── No Cache                       │   ├── Cache Policy: CachingDisabled
│   ├── Forward headers                │   └── Origin Request: AllViewer
│   └── Origin: api.site.com           │
│                                      │
├── Rule: /images/*                    ├── Behavior: /images/* → S3 Media
│   ├── Cache: 30d                     │   ├── Cache Policy: 30 dias
│   ├── Image Optimization             │   └── Origin Group (failover)
│   └── Origin: images.site.com        │
│                                      │
├── Edge Worker: Auth                  ├── Lambda@Edge: JWT Auth
├── Edge Worker: A/B                   ├── CF Function: A/B Test
├── Site Shield: US                    ├── Origin Shield: us-east-1
├── WAF: OWASP Rules                  ├── WAF: Managed Rules + Custom
└── Bot Manager                        └── WAF: Bot Control
```

#### Fase 3: Build

```hcl
# ==============================================================================
# migration/cloudfront.tf — CloudFront para migração
# ==============================================================================

# Criar distribuição CloudFront com domínio temporário primeiro
resource "aws_cloudfront_distribution" "migration" {
  enabled         = true
  is_ipv6_enabled = true
  comment         = "Migration from Akamai — testing phase"
  http_version    = "http2and3"
  price_class     = "PriceClass_All"

  # IMPORTANTE: NÃO adicionar o CNAME de produção ainda!
  # Usar apenas o domínio cloudfront.net para testes
  # aliases = [] # Vazio durante testes!

  origin {
    domain_name = "api.site.com"
    origin_id   = "api-origin"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "api-origin"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    cache_policy_id = "658327ea-f89d-4fab-a63d-7e88639e58f6" # CachingOptimized
  }

  viewer_certificate {
    cloudfront_default_certificate = true  # Usar cert padrão para testes
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
}

output "test_domain" {
  value       = aws_cloudfront_distribution.migration.domain_name
  description = "Use este domínio para testar antes do cutover"
}
```

#### Fase 4: Validate

```bash
#!/bin/bash
# ==============================================================================
# validate-migration.sh — Script de validação completa
# ==============================================================================

AKAMAI_DOMAIN="www.site.com"
CLOUDFRONT_DOMAIN="d1234abcdef.cloudfront.net"
REPORT_FILE="migration-validation-$(date +%Y%m%d).md"

echo "# Relatório de Validação de Migração CDN" > $REPORT_FILE
echo "Data: $(date)" >> $REPORT_FILE
echo "" >> $REPORT_FILE

# Lista de URLs para testar
URLS=(
  "/"
  "/about"
  "/products"
  "/products/123"
  "/api/v1/products"
  "/api/v1/categories"
  "/images/hero.jpg"
  "/images/products/123/main.webp"
  "/static/js/main.js"
  "/static/css/app.css"
  "/robots.txt"
  "/sitemap.xml"
  "/favicon.ico"
)

echo "## Comparação de Respostas" >> $REPORT_FILE
echo "" >> $REPORT_FILE
echo "| URL | Akamai Status | CF Status | Akamai TTL | CF TTL | Match? |" >> $REPORT_FILE
echo "|-----|---------------|-----------|------------|--------|--------|" >> $REPORT_FILE

PASS=0
FAIL=0

for url in "${URLS[@]}"; do
  # Testar Akamai
  AKAMAI_RESULT=$(curl -sI "https://${AKAMAI_DOMAIN}${url}" 2>/dev/null)
  AKAMAI_STATUS=$(echo "$AKAMAI_RESULT" | head -1 | awk '{print $2}')
  AKAMAI_CACHE=$(echo "$AKAMAI_RESULT" | grep -i "cache-control" | head -1 | awk '{print $2}')

  # Testar CloudFront
  CF_RESULT=$(curl -sI "https://${CLOUDFRONT_DOMAIN}${url}" 2>/dev/null)
  CF_STATUS=$(echo "$CF_RESULT" | head -1 | awk '{print $2}')
  CF_CACHE=$(echo "$CF_RESULT" | grep -i "cache-control" | head -1 | awk '{print $2}')
  CF_HIT=$(echo "$CF_RESULT" | grep -i "x-cache" | awk '{print $2,$3,$4}')

  # Comparar
  if [ "$AKAMAI_STATUS" = "$CF_STATUS" ]; then
    MATCH="OK"
    PASS=$((PASS + 1))
  else
    MATCH="FAIL"
    FAIL=$((FAIL + 1))
  fi

  echo "| ${url} | ${AKAMAI_STATUS} | ${CF_STATUS} | ${AKAMAI_CACHE} | ${CF_CACHE} | ${MATCH} |" >> $REPORT_FILE
done

echo "" >> $REPORT_FILE
echo "## Resumo: ${PASS} OK / ${FAIL} FAIL de ${#URLS[@]} URLs" >> $REPORT_FILE

# Teste de headers de segurança
echo "" >> $REPORT_FILE
echo "## Headers de Segurança" >> $REPORT_FILE
echo "" >> $REPORT_FILE

SECURITY_HEADERS=(
  "strict-transport-security"
  "x-content-type-options"
  "x-frame-options"
  "x-xss-protection"
  "content-security-policy"
  "referrer-policy"
  "permissions-policy"
)

for header in "${SECURITY_HEADERS[@]}"; do
  CF_VALUE=$(curl -sI "https://${CLOUDFRONT_DOMAIN}/" | grep -i "$header" | cut -d: -f2-)
  if [ -n "$CF_VALUE" ]; then
    echo "- [x] $header: $CF_VALUE" >> $REPORT_FILE
  else
    echo "- [ ] **MISSING**: $header" >> $REPORT_FILE
  fi
done

echo "" >> $REPORT_FILE
echo "## Performance" >> $REPORT_FILE

# Testar latência
for region in "sa-east-1" "us-east-1" "eu-west-1"; do
  echo "### Region: $region" >> $REPORT_FILE

  AKAMAI_TTFB=$(curl -o /dev/null -s -w "%{time_starttransfer}" "https://${AKAMAI_DOMAIN}/")
  CF_TTFB=$(curl -o /dev/null -s -w "%{time_starttransfer}" "https://${CLOUDFRONT_DOMAIN}/")

  echo "- Akamai TTFB: ${AKAMAI_TTFB}s" >> $REPORT_FILE
  echo "- CloudFront TTFB: ${CF_TTFB}s" >> $REPORT_FILE
  echo "" >> $REPORT_FILE
done

cat $REPORT_FILE
```

#### Fase 5: Cache Warming

```bash
#!/bin/bash
# ==============================================================================
# cache-warm.sh — Script de aquecimento de cache
# ==============================================================================

DOMAIN="d1234abcdef.cloudfront.net"
CONCURRENT=50
SITEMAP_URL="https://www.site.com/sitemap.xml"
LOG_FILE="cache-warm-$(date +%Y%m%d-%H%M%S).log"

echo "Iniciando cache warming para $DOMAIN" | tee $LOG_FILE
echo "Timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)" | tee -a $LOG_FILE

# Extrair URLs do sitemap
echo "Extraindo URLs do sitemap..." | tee -a $LOG_FILE
curl -s "$SITEMAP_URL" | \
  grep -oP '<loc>\K[^<]+' | \
  sed "s|https://www.site.com|https://${DOMAIN}|g" > /tmp/urls-to-warm.txt

URL_COUNT=$(wc -l < /tmp/urls-to-warm.txt)
echo "Total de URLs para aquecer: $URL_COUNT" | tee -a $LOG_FILE

# Aquecer com xargs (paralelo)
echo "" | tee -a $LOG_FILE
echo "Aquecendo cache com $CONCURRENT conexões simultâneas..." | tee -a $LOG_FILE

cat /tmp/urls-to-warm.txt | xargs -P $CONCURRENT -I {} sh -c '
  STATUS=$(curl -o /dev/null -s -w "%{http_code}" -H "Accept-Encoding: gzip, br" "{}")
  CACHE=$(curl -sI -H "Accept-Encoding: gzip, br" "{}" | grep -i "x-cache" | tr -d "\r")
  echo "[$STATUS] $CACHE - {}"
' 2>&1 | tee -a $LOG_FILE

# Aquecer assets estáticos críticos
CRITICAL_ASSETS=(
  "/static/js/main.abc123.js"
  "/static/css/app.def456.css"
  "/static/images/logo.svg"
  "/static/fonts/custom.woff2"
  "/images/hero-desktop.webp"
  "/images/hero-mobile.webp"
)

echo "" | tee -a $LOG_FILE
echo "Aquecendo assets críticos..." | tee -a $LOG_FILE

for asset in "${CRITICAL_ASSETS[@]}"; do
  for encoding in "gzip" "br" "identity"; do
    STATUS=$(curl -o /dev/null -s -w "%{http_code}" \
      -H "Accept-Encoding: $encoding" \
      "https://${DOMAIN}${asset}")
    echo "[$STATUS] $encoding - $asset" | tee -a $LOG_FILE
  done
done

# Segundo pass — verificar se está tudo cacheado
echo "" | tee -a $LOG_FILE
echo "Verificando cache hits (segundo pass)..." | tee -a $LOG_FILE

HITS=0
MISSES=0

head -100 /tmp/urls-to-warm.txt | while read url; do
  CACHE=$(curl -sI "$url" | grep -i "x-cache" | grep -c "Hit")
  if [ "$CACHE" -gt 0 ]; then
    HITS=$((HITS + 1))
  else
    MISSES=$((MISSES + 1))
  fi
done

echo "" | tee -a $LOG_FILE
echo "Resultado: Hits=$HITS Misses=$MISSES" | tee -a $LOG_FILE
echo "Cache warming concluído!" | tee -a $LOG_FILE
```

#### Fase 6: DNS Cutover (Zero Downtime)

```bash
#!/bin/bash
# ==============================================================================
# dns-cutover.sh — Migração DNS com zero downtime
# ==============================================================================

HOSTED_ZONE_ID="Z1234567890ABC"
DOMAIN="www.site.com"
CF_DOMAIN="d1234abcdef.cloudfront.net"
CF_ZONE_ID="Z2FDTNDATAQYW2"  # Zone ID fixo do CloudFront

echo "=== PLANO DE CUTOVER DNS ==="
echo ""
echo "Passo 1: Reduzir TTL do DNS (fazer 48h antes!)"
echo ""

# 48 horas antes: reduzir TTL para 60 segundos
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "'$DOMAIN'",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "www.site.com.akamai.net."}]
      }
    }]
  }'

echo ""
echo "Passo 2: Adicionar CNAME ao CloudFront (antes do cutover)"
echo ""

# Adicionar o alias na distribuição CloudFront
aws cloudfront update-distribution \
  --id E1234567890 \
  --if-match $ETAG \
  --distribution-config file://updated-config.json

echo ""
echo "Passo 3: CUTOVER — Mudar o DNS"
echo ""

# O MOMENTO CRÍTICO: mudar de Akamai para CloudFront
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "'$DOMAIN'",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "'$CF_ZONE_ID'",
          "DNSName": "'$CF_DOMAIN'",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'

echo ""
echo "Passo 4: Monitorar"
echo ""

# Monitorar a propagação
for i in $(seq 1 60); do
  RESOLVED=$(dig +short $DOMAIN)
  echo "[$(date +%H:%M:%S)] $DOMAIN -> $RESOLVED"
  sleep 10
done

echo ""
echo "Passo 5: Após estabilização (1 semana), restaurar TTL"
echo ""

# Restaurar TTL normal
# aws route53 change-resource-record-sets ... TTL: 300
```

#### Plano de Rollback

```bash
#!/bin/bash
# ==============================================================================
# rollback.sh — Rollback para Akamai em caso de problemas
# ==============================================================================

echo "!!! ROLLBACK INICIADO !!!"
echo "Voltando DNS para Akamai..."

HOSTED_ZONE_ID="Z1234567890ABC"
DOMAIN="www.site.com"
AKAMAI_CNAME="www.site.com.akamai.net."

# Rollback instantâneo (TTL de 60s = 1 min para propagar)
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "'$DOMAIN'",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "'$AKAMAI_CNAME'"}]
      }
    }]
  }'

echo "DNS revertido. Propagação em ~60 segundos."
echo "Monitorando..."

for i in $(seq 1 30); do
  RESOLVED=$(dig +short $DOMAIN)
  echo "[$(date +%H:%M:%S)] $DOMAIN -> $RESOLVED"
  sleep 5
done
```

#### Checklist de Validação Completa (30+ itens)

```markdown
## Checklist de Validação — Migração CDN

### Funcionalidade Básica
- [ ] Homepage carrega corretamente
- [ ] Todas as páginas do sitemap retornam 200
- [ ] Redirects (301/302) funcionam corretamente
- [ ] Custom error pages (404, 500) funcionam
- [ ] SPA routing funciona (deep links)
- [ ] Formulários POST funcionam
- [ ] Upload de arquivos funciona
- [ ] WebSocket connections funcionam
- [ ] API endpoints retornam dados corretos
- [ ] Autenticação funciona (login/logout)

### Cache e Performance
- [ ] Cache-Control headers corretos em todas as respostas
- [ ] Content-Encoding (gzip/brotli) funciona
- [ ] Cache hit rate > 80% após warming
- [ ] TTFB comparable ou melhor que CDN anterior
- [ ] Imagens servidas com WebP/AVIF quando suportado
- [ ] Static assets com immutable cache

### Segurança
- [ ] HTTPS funciona em todos os endpoints
- [ ] HTTP redireciona para HTTPS
- [ ] HSTS header presente
- [ ] X-Content-Type-Options: nosniff
- [ ] X-Frame-Options ou CSP frame-ancestors
- [ ] Content-Security-Policy header presente
- [ ] WAF bloqueia SQL injection
- [ ] WAF bloqueia XSS
- [ ] Rate limiting funciona
- [ ] Bot protection ativo
- [ ] Geo-restriction funciona (se aplicável)

### DNS e Certificados
- [ ] Certificado SSL válido e correto
- [ ] SANs incluem todos os domínios necessários
- [ ] DNS resolve corretamente
- [ ] DNSSEC funciona (se usado)
- [ ] IPv6 (AAAA records) funciona

### Monitoramento
- [ ] Logs sendo gerados no S3
- [ ] CloudWatch métricas aparecendo
- [ ] Alarmes configurados e testados
- [ ] Dashboard operacional funcional
- [ ] Real-time logs funcionando (se configurado)

### Compliance
- [ ] Headers de CORS corretos
- [ ] Cookies SameSite configurados
- [ ] GDPR compliance mantida
- [ ] PCI compliance mantida (se e-commerce)
```

### O Que Aprendemos

1. **Reduza o TTL do DNS 48h antes** — garante que o rollback será rápido
2. **Cache warming** é essencial para evitar thundering herd na origem
3. **Validação paralela** (Akamai e CloudFront rodando simultaneamente) minimiza risco
4. **Feature mapping** detalhado evita surpresas durante o cutover
5. **Rollback plan** testado é obrigatório — nunca migre sem um

### Dica Expert

> **Use "dual-stack" durante a migração.** Configure o CloudFront mas mantenha o Akamai ativo
> por 2 semanas após o cutover. Use Route 53 weighted routing para enviar 10% do tráfego
> para CloudFront primeiro, monitorar, e gradualmente aumentar. Isso transforma uma migração
> "big bang" em uma migração gradual muito mais segura. Custo extra de manter duas CDNs por
> 2 semanas é insignificante comparado ao risco de downtime.

---

## Desafio 59: Video Streaming Platform (Level 400)

### Classificação

| Atributo         | Valor                                                  |
|------------------|--------------------------------------------------------|
| **Nível**        | 400 — Expert                                           |
| **Tipo**         | Streaming de Vídeo                                     |
| **Tempo**        | 4-6 horas                                              |
| **Serviços**     | CloudFront, MediaLive, MediaPackage, MediaConvert, S3  |
| **Complexidade** | Muito Alta                                             |

### Objetivo

Construir uma plataforma de streaming de vídeo completa, suportando tanto **Live streaming**
quanto **Video on Demand (VOD)**, com DRM, adaptive bitrate, e geo-restriction por
licenciamento de conteúdo.

### Cenário Real

> Você está construindo uma plataforma de streaming para uma emissora de TV brasileira.
> O sistema precisa suportar transmissões ao vivo de eventos esportivos (10.000+ viewers
> simultâneos) e um catálogo VOD com 5.000+ títulos. Cada título tem restrições geográficas
> diferentes baseadas nos contratos de licenciamento.

### Arquitetura — Live Streaming

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│              │     │                  │     │                  │     │                  │
│   Câmera /   │────►│  AWS Elemental   │────►│  AWS Elemental   │────►│   CloudFront     │
│   Encoder    │     │  MediaLive       │     │  MediaPackage    │     │   Distribution   │
│   (RTMP)     │     │                  │     │                  │     │                  │
│              │     │  Input:          │     │  Channel:        │     │  Behavior:       │
│  OBS Studio  │     │  - RTMP Push     │     │  - HLS Origin    │     │  /live/*         │
│  ou          │     │  - 1080p60       │     │  - DASH Origin   │     │  TTL: 1-6s       │
│  Wirecast    │     │                  │     │  - CMAF          │     │                  │
│              │     │  Output:         │     │                  │     │  Origin:          │
│              │     │  - HLS           │     │  Endpoints:      │     │  MediaPackage     │
│              │     │  - 1080/720/480  │     │  - HLS           │     │                  │
│              │     │  - 360/240       │     │  - DASH           │     │  Signed URLs     │
│              │     │                  │     │  - CMAF          │     │                  │
└──────────────┘     └──────────────────┘     └──────────────────┘     └────────┬─────────┘
                                                                                │
                                                                                ▼
                                                                      ┌──────────────────┐
                                                                      │   Viewers         │
                                                                      │                  │
                                                                      │  - Web (HLS.js)  │
                                                                      │  - Mobile (Exo)  │
                                                                      │  - Smart TV      │
                                                                      │  - Fire TV       │
                                                                      └──────────────────┘
```

### Arquitetura — VOD Pipeline

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│              │     │                  │     │                  │     │                  │
│  Upload      │────►│  S3 Bucket       │────►│  AWS Elemental   │────►│  S3 Bucket       │
│  (S3/API)    │     │  (Input)         │     │  MediaConvert    │     │  (Output)        │
│              │     │                  │     │                  │     │                  │
│  Formato:    │     │  Trigger:        │     │  Job Template:   │     │  Estrutura:      │
│  - MP4       │     │  S3 Event →      │     │  - HLS (6 qual) │     │  /vod/{id}/      │
│  - MOV       │     │  Lambda →        │     │  - DASH          │     │    hls/          │
│  - MKV       │     │  MediaConvert    │     │  - Thumbnails    │     │    dash/         │
│              │     │                  │     │  - MP4 (preview) │     │    thumbs/       │
└──────────────┘     └──────────────────┘     └──────────────────┘     └────────┬─────────┘
                                                                                │
                                                                                ▼
                                                                      ┌──────────────────┐
                                                                      │   CloudFront     │
                                                                      │   Distribution   │
                                                                      │                  │
                                                                      │  Behavior:       │
                                                                      │  /vod/*          │
                                                                      │                  │
                                                                      │  Signed Cookies  │
                                                                      │  Geo-Restriction │
                                                                      │                  │
                                                                      │  Cache:          │
                                                                      │  manifest: 2s    │
                                                                      │  segments: 1 ano │
                                                                      └──────────────────┘
```

### HLS vs DASH — Comparação

| Característica       | HLS (HTTP Live Streaming)       | DASH (Dynamic Adaptive Streaming) |
|----------------------|---------------------------------|-----------------------------------|
| **Desenvolvido por** | Apple                           | MPEG (padrão aberto)              |
| **Formato container**| MPEG-TS ou fMP4                 | fMP4 (ISO BMFF)                   |
| **Manifest**         | .m3u8                           | .mpd (XML)                        |
| **Codecs**           | H.264, H.265, VP9              | H.264, H.265, VP9, AV1           |
| **DRM**              | FairPlay (Apple)               | Widevine (Google), PlayReady (MS) |
| **Latência**         | 6-30s (LL-HLS: 2-4s)          | 2-10s (LL-DASH: 2-4s)            |
| **Compatibilidade**  | iOS nativo, web via hls.js     | Android nativo, web via dash.js   |
| **Recomendação**     | Obrigatório para iOS/Safari    | Melhor para Android/Chrome        |

**Na prática:** Use CMAF (Common Media Application Format) que permite servir o mesmo
conteúdo para HLS e DASH, reduzindo armazenamento e custo de encoding.

### Adaptive Bitrate (ABR) — Como Funciona

```
Qualidade disponível (Encoding Ladder):

┌─────────────────────────────────────────────────────────────┐
│  Resolução    │ Bitrate  │ Codec  │ Uso                     │
│───────────────│──────────│────────│─────────────────────────│
│  1920x1080    │ 6.0 Mbps │ H.264  │ Desktop/Smart TV        │
│  1280x720     │ 3.5 Mbps │ H.264  │ Desktop/Tablet          │
│  960x540      │ 2.0 Mbps │ H.264  │ Tablet/Mobile WiFi      │
│  640x360      │ 1.0 Mbps │ H.264  │ Mobile 4G               │
│  480x270      │ 0.5 Mbps │ H.264  │ Mobile 3G               │
│  320x180      │ 0.2 Mbps │ H.264  │ Conexão ruim            │
└─────────────────────────────────────────────────────────────┘

O player (hls.js/dash.js/ExoPlayer) monitora:
  1. Banda disponível (throughput medido)
  2. Buffer level (quanto vídeo já baixou)
  3. Estabilidade da conexão

E automaticamente sobe/desce a qualidade:

Banda: 10 Mbps ──────────────── Qualidade: 1080p
                \
Banda: 3 Mbps   \────────────── Qualidade: 720p
                   \
Banda: 1 Mbps      \─────────── Qualidade: 360p
                      \
Banda: 0.3 Mbps       \──────── Qualidade: 180p

Manifesto HLS exemplo (.m3u8):

#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=6000000,RESOLUTION=1920x1080
1080p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=3500000,RESOLUTION=1280x720
720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2000000,RESOLUTION=960x540
540p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=640x360
360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=500000,RESOLUTION=480x270
270p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=200000,RESOLUTION=320x180
180p/playlist.m3u8
```

### DRM e Proteção de Conteúdo

```
Estratégia de proteção em camadas:

Camada 1: Signed URLs/Cookies (CloudFront)
├── Usuário faz login → API gera signed cookie
├── Cookie inclui: Expiration + Policy + Signature
├── CloudFront valida o cookie a cada request
└── Protege contra: link sharing, hotlinking

Camada 2: Token por Segmento (Lambda@Edge)
├── Cada request de segmento (.ts/.m4s) é validado
├── Token inclui: session_id + timestamp + hash
├── Previne: download em massa, scraping
└── Revogação em tempo real possível

Camada 3: Geo-Restriction por Conteúdo
├── Lambda@Edge verifica licenciamento
├── Consulta DynamoDB: content_id + country → allowed?
├── Retorna 451 (Unavailable for Legal Reasons)
└── Diferentes restrições por título
```

#### Token Authentication por Segmento

```javascript
// segment-token-validator.js — Lambda@Edge (viewer-request)
'use strict';

const crypto = require('crypto');
const TOKEN_SECRET = 'your-hmac-secret-key';  // Em produção: Secrets Manager

exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const uri = request.uri;

  // Manifestos passam sem token (o player precisa ler primeiro)
  if (uri.endsWith('.m3u8') || uri.endsWith('.mpd')) {
    return request;
  }

  // Segmentos precisam de token
  if (uri.endsWith('.ts') || uri.endsWith('.m4s') || uri.endsWith('.mp4')) {
    const querystring = request.querystring;
    const params = new URLSearchParams(querystring);

    const token = params.get('token');
    const expires = params.get('exp');
    const sessionId = params.get('sid');

    if (!token || !expires || !sessionId) {
      return forbiddenResponse('Token ausente');
    }

    // Verificar expiração
    if (parseInt(expires) < Math.floor(Date.now() / 1000)) {
      return forbiddenResponse('Token expirado');
    }

    // Verificar HMAC
    const expectedToken = crypto
      .createHmac('sha256', TOKEN_SECRET)
      .update(`${uri}:${expires}:${sessionId}`)
      .digest('hex');

    if (token !== expectedToken) {
      return forbiddenResponse('Token inválido');
    }

    // Remover token da query string (não enviar para origem)
    params.delete('token');
    params.delete('exp');
    params.delete('sid');
    request.querystring = params.toString();

    return request;
  }

  return request;
};

function forbiddenResponse(reason) {
  return {
    status: '403',
    statusDescription: 'Forbidden',
    headers: {
      'content-type': [{ key: 'Content-Type', value: 'application/json' }],
      'cache-control': [{ key: 'Cache-Control', value: 'no-store' }],
    },
    body: JSON.stringify({ error: 'Forbidden', reason }),
  };
}
```

#### Geo-Restriction por Licenciamento

```javascript
// geo-license-check.js — Lambda@Edge (viewer-request)
'use strict';

const AWS = require('aws-sdk');

// Cache local para evitar consultas repetidas ao DynamoDB
const licenseCache = new Map();
const CACHE_TTL_MS = 60000; // 1 minuto

exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const uri = request.uri;
  const headers = request.headers;

  // Extrair content_id do path: /vod/{content_id}/...
  const match = uri.match(/^\/vod\/([^/]+)\//);
  if (!match) return request;

  const contentId = match[1];
  const country = headers['cloudfront-viewer-country']
    ? headers['cloudfront-viewer-country'][0].value
    : 'BR';

  // Verificar cache local
  const cacheKey = `${contentId}:${country}`;
  const cached = licenseCache.get(cacheKey);

  if (cached && Date.now() - cached.timestamp < CACHE_TTL_MS) {
    if (!cached.allowed) {
      return geoBlockedResponse(country, contentId);
    }
    return request;
  }

  // Consultar DynamoDB
  try {
    const dynamodb = new AWS.DynamoDB.DocumentClient({ region: 'us-east-1' });
    const result = await dynamodb.get({
      TableName: 'content-licenses',
      Key: {
        PK: `CONTENT#${contentId}`,
        SK: `LICENSE#${country}`,
      },
    }).promise();

    const allowed = result.Item && result.Item.active === true;

    // Atualizar cache
    licenseCache.set(cacheKey, { allowed, timestamp: Date.now() });

    if (!allowed) {
      return geoBlockedResponse(country, contentId);
    }

    return request;
  } catch (err) {
    console.error('License check error:', err);
    // Em caso de erro, permitir (fail open) — ou bloquear (fail closed)
    return request;
  }
};

function geoBlockedResponse(country, contentId) {
  return {
    status: '451',
    statusDescription: 'Unavailable For Legal Reasons',
    headers: {
      'content-type': [{ key: 'Content-Type', value: 'application/json' }],
      'cache-control': [{ key: 'Cache-Control', value: 'no-store' }],
    },
    body: JSON.stringify({
      error: 'Content not available',
      message: `Este conteúdo não está disponível no seu país (${country}).`,
      contentId: contentId,
    }),
  };
}
```

### Cache Settings — Manifest vs Segments

```
Tipo de Arquivo      │ TTL Cache     │ Razão
─────────────────────│───────────────│──────────────────────────────────
Live .m3u8 (master)  │ 1-2s          │ Atualiza constantemente
Live .m3u8 (media)   │ 1-2s          │ Novos segmentos a cada 2-6s
Live .ts/.m4s        │ 1 ano         │ Segmento nunca muda após criado
                     │               │
VOD .m3u8 (master)   │ 1 hora        │ Não muda, mas precisa invalidar
VOD .m3u8 (media)    │ 1 hora        │ Não muda, mas precisa invalidar
VOD .ts/.m4s         │ 1 ano         │ Imutável (nome inclui sequence)
                     │               │
.mpd (DASH manifest) │ 2-5s (live)   │ Similar ao .m3u8
                     │ 1h (VOD)      │
Thumbnails (.jpg)    │ 30 dias       │ Não mudam após geração
Subtitles (.vtt)     │ 1 dia         │ Podem ser atualizados
```

### Estimativa de Custo — 10.000 Viewers Simultâneos

```
Premissas:
- 10.000 viewers simultâneos
- Duração do evento: 3 horas
- Bitrate médio: 3.5 Mbps (adaptativo)
- Segmentos de 6 segundos

Cálculo de Tráfego:
- Tráfego por viewer: 3.5 Mbps × 3h × 3600s = 37.8 GB
- Tráfego total: 37.8 GB × 10.000 = 378 TB

Custos CloudFront (sa-east-1 - South America):
- Primeiro 10 TB:          10 TB × $0.110/GB = $1,100
- Próximos 40 TB:          40 TB × $0.090/GB = $3,600
- Próximos 100 TB:        100 TB × $0.070/GB = $7,000
- Próximos 228 TB:        228 TB × $0.060/GB = $13,680
- CloudFront Total:                            ~$25,380

Custos MediaLive:
- Channel (single pipeline): ~$1.66/h × 3h = $4.98
- Com redundância (2 pipelines): × 2 = $9.96

Custos MediaPackage:
- Ingest: 378 TB × $0.10/GB = $37,800  (!)
  (Nota: negociar preço para este volume)

Custo S3 (para VOD):
- Armazenamento: depende do catálogo
- Requests: ~100M requests × $0.005/1000 = $500

Custo Total Estimado (evento de 3h):
─────────────────────────────
CloudFront:   ~$25,380
MediaLive:    ~$10
MediaPackage: ~$37,800 (negociável)
S3/requests:  ~$500
─────────────────────────────
TOTAL:        ~$63,690

Custo por viewer/hora: ~$2.12

NOTA: Com Reserved Capacity pricing, os custos podem
cair 40-60%. Para eventos recorrentes, vale a pena.
```

### Terraform — Pipeline VOD

```hcl
# ==============================================================================
# streaming/vod-pipeline.tf — Pipeline VOD completo
# ==============================================================================

# Bucket de input (uploads)
resource "aws_s3_bucket" "vod_input" {
  bucket = "${var.project_name}-vod-input"
}

# Bucket de output (conteúdo processado)
resource "aws_s3_bucket" "vod_output" {
  bucket = "${var.project_name}-vod-output"
}

resource "aws_s3_bucket_cors_configuration" "vod_output" {
  bucket = aws_s3_bucket.vod_output.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "HEAD"]
    allowed_origins = ["https://${var.domain_name}"]
    max_age_seconds = 3600
  }
}

# IAM Role para MediaConvert
resource "aws_iam_role" "mediaconvert" {
  name = "${var.project_name}-mediaconvert"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "mediaconvert.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "mediaconvert" {
  name = "${var.project_name}-mediaconvert-policy"
  role = aws_iam_role.mediaconvert.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = [
          "${aws_s3_bucket.vod_input.arn}/*",
          "${aws_s3_bucket.vod_output.arn}/*"
        ]
      }
    ]
  })
}

# Lambda para trigger de encoding
resource "aws_lambda_function" "vod_trigger" {
  filename         = "${path.module}/lambda/vod-trigger.zip"
  function_name    = "${var.project_name}-vod-trigger"
  role             = aws_iam_role.vod_lambda.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  timeout          = 60

  environment {
    variables = {
      OUTPUT_BUCKET     = aws_s3_bucket.vod_output.id
      MEDIACONVERT_ROLE = aws_iam_role.mediaconvert.arn
      MEDIACONVERT_ENDPOINT = "https://abc123.mediaconvert.sa-east-1.amazonaws.com"
    }
  }
}

# S3 Event → Lambda
resource "aws_s3_bucket_notification" "vod_trigger" {
  bucket = aws_s3_bucket.vod_input.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.vod_trigger.arn
    events              = ["s3:ObjectCreated:*"]
    filter_suffix       = ".mp4"
  }

  lambda_function {
    lambda_function_arn = aws_lambda_function.vod_trigger.arn
    events              = ["s3:ObjectCreated:*"]
    filter_suffix       = ".mov"
  }
}

resource "aws_lambda_permission" "allow_s3" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.vod_trigger.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.vod_input.arn
}

# CloudFront para VOD
resource "aws_cloudfront_distribution" "vod" {
  enabled         = true
  is_ipv6_enabled = true
  comment         = "VOD Streaming Distribution"
  http_version    = "http2and3"
  price_class     = "PriceClass_200"

  aliases = ["vod.${var.domain_name}"]

  origin {
    domain_name              = aws_s3_bucket.vod_output.bucket_regional_domain_name
    origin_id                = "s3-vod"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id

    origin_shield {
      enabled              = true
      origin_shield_region = "sa-east-1"
    }
  }

  # Behavior para manifestos (TTL curto para permitir invalidação)
  ordered_cache_behavior {
    path_pattern     = "*.m3u8"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-vod"
    compress         = true

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 3600    # 1 hora
    max_ttl     = 86400   # 1 dia

    viewer_protocol_policy = "https-only"
  }

  # Behavior para segmentos (cache longo — imutáveis)
  ordered_cache_behavior {
    path_pattern     = "*.ts"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-vod"
    compress         = false  # Vídeo já é comprimido

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 31536000
    default_ttl = 31536000
    max_ttl     = 31536000

    viewer_protocol_policy = "https-only"
  }

  # Behavior para segmentos fMP4
  ordered_cache_behavior {
    path_pattern     = "*.m4s"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-vod"
    compress         = false

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 31536000
    default_ttl = 31536000
    max_ttl     = 31536000

    viewer_protocol_policy = "https-only"
  }

  # Behavior para thumbnails
  ordered_cache_behavior {
    path_pattern     = "*/thumbs/*"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-vod"
    compress         = true

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 2592000
    default_ttl = 2592000
    max_ttl     = 2592000

    viewer_protocol_policy = "https-only"
  }

  # Default behavior
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-vod"
    compress         = true

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 86400
    max_ttl     = 31536000

    viewer_protocol_policy = "https-only"

    # Usar signed cookies para autenticação
    trusted_key_groups = [aws_cloudfront_key_group.streaming.id]
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cloudfront.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"  # Gerenciado via Lambda@Edge por conteúdo
    }
  }
}

# Key Group para Signed URLs/Cookies
resource "aws_cloudfront_public_key" "streaming" {
  comment     = "Public key for VOD signed cookies"
  encoded_key = file("${path.module}/keys/cf-public-key.pem")
  name        = "${var.project_name}-streaming-key"
}

resource "aws_cloudfront_key_group" "streaming" {
  comment = "Key group for VOD streaming"
  items   = [aws_cloudfront_public_key.streaming.id]
  name    = "${var.project_name}-streaming-keys"
}
```

### Validação

```bash
# Testar VOD pipeline
echo "=== Upload vídeo de teste ==="
aws s3 cp test-video.mp4 s3://streaming-vod-input/test/video.mp4

echo "=== Verificar job MediaConvert ==="
aws mediaconvert list-jobs --status PROGRESSING --region sa-east-1

echo "=== Verificar output após processamento ==="
aws s3 ls s3://streaming-vod-output/test/ --recursive

echo "=== Testar playback via CloudFront ==="
curl -sI "https://vod.exemplo.com.br/test/hls/master.m3u8"

echo "=== Verificar signed cookie ==="
# Gerar signed cookie para teste
python3 -c "
import datetime, json, rsa, base64
# ... (script de geração de signed cookie)
print('Set-Cookie: CloudFront-Policy=...')
"

echo "=== Testar com signed cookie ==="
curl -b "CloudFront-Policy=..." \
     -b "CloudFront-Signature=..." \
     -b "CloudFront-Key-Pair-Id=..." \
     "https://vod.exemplo.com.br/test/hls/master.m3u8"
```

### O Que Aprendemos

1. **CMAF** unifica HLS e DASH, reduzindo armazenamento e custo de encoding
2. **Cache TTL diferente** para manifests (curto) vs segments (longo) é essencial
3. **Origin Shield** é especialmente importante para live streaming (muitos edge locations)
4. **Signed cookies** são melhores que signed URLs para streaming (muitos segments)
5. **Token por segmento** adiciona uma camada extra contra pirataria
6. **Geo-restriction por conteúdo** (não por distribuição) atende licenciamento real

### Dica Expert

> **Para live streaming com baixa latência, use LL-HLS (Low-Latency HLS).** O MediaPackage
> suporta LL-HLS nativamente, reduzindo a latência de 15-30s para 2-4s. Configure o
> CloudFront com TTL de 1s para manifestos e use HTTP/3 (QUIC) para reduzir o overhead de
> conexão. Para eventos esportivos onde cada segundo importa, considere WebRTC via Amazon
> IVS (Interactive Video Service) que oferece latência sub-segundo (<500ms).

---

## Desafio 60: Certificação e Próximos Passos (Level Meta)

### Classificação

| Atributo         | Valor                                  |
|------------------|----------------------------------------|
| **Nível**        | Meta (reflexão e consolidação)         |
| **Tipo**         | Revisão, Certificação, Carreira        |
| **Tempo**        | Indefinido                             |
| **Objetivo**     | Consolidar conhecimento e planejar     |

### Recapitulação Completa — Todos os 60 Desafios

| # | Desafio | Módulo | Nível |
|---|---------|--------|-------|
| 1 | O Que é uma CDN e Por Que Usar | 1 | 100 |
| 2 | Sua Primeira Distribuição CloudFront | 1 | 100 |
| 3 | CloudFront + S3 Static Website | 1 | 100 |
| 4 | HTTPS e Certificados com ACM | 1 | 200 |
| 5 | Custom Domain com Route 53 | 1 | 200 |
| 6 | Entendendo o Cache (Hit, Miss, TTL) | 1 | 200 |
| 7 | Invalidação de Cache | 1 | 200 |
| 8 | Headers de Cache e Diagnóstico | 1 | 200 |
| 9 | Origin Access Control (OAC) | 2 | 200 |
| 10 | Custom Origin (ALB/EC2) | 2 | 200 |
| 11 | Multiple Origins e Behaviors | 2 | 200 |
| 12 | Origin Groups (Failover) | 2 | 300 |
| 13 | Origin Shield | 2 | 300 |
| 14 | Custom Headers na Origem | 2 | 300 |
| 15 | Origin Connection (Timeouts/Keepalive) | 2 | 300 |
| 16 | Behaviors Avançados e Prioridade | 2 | 300 |
| 17 | Cache Policies Customizadas | 3 | 200 |
| 18 | Origin Request Policies | 3 | 300 |
| 19 | Response Headers Policies | 3 | 300 |
| 20 | Cache Key e Normalização | 3 | 300 |
| 21 | Managed Policies da AWS | 3 | 200 |
| 22 | Cache por Device Type | 3 | 300 |
| 23 | Cache por Geolocalização | 3 | 300 |
| 24 | Cache Invalidation Strategies | 3 | 300 |
| 25 | Sua Primeira CloudFront Function | 4 | 200 |
| 26 | URL Rewrite e Redirect com Functions | 4 | 300 |
| 27 | A/B Testing com CloudFront Functions | 4 | 300 |
| 28 | Security Headers com Functions | 4 | 300 |
| 29 | Lambda@Edge — Primeiros Passos | 4 | 300 |
| 30 | Lambda@Edge — Manipulação de Origem | 4 | 300 |
| 31 | Lambda@Edge — Autenticação JWT | 4 | 400 |
| 32 | Lambda@Edge — Image Resize on-the-fly | 4 | 400 |
| 33 | AWS WAF v2 com CloudFront | 5 | 300 |
| 34 | Rate Limiting com WAF | 5 | 300 |
| 35 | Bot Management | 5 | 300 |
| 36 | AWS Shield (Standard e Advanced) | 5 | 300 |
| 37 | Signed URLs | 5 | 300 |
| 38 | Signed Cookies | 5 | 300 |
| 39 | Geo-Restriction | 5 | 200 |
| 40 | Field-Level Encryption | 5 | 400 |
| 41 | CloudFront + S3 (OAC Deep Dive) | 6 | 300 |
| 42 | CloudFront + ALB + ECS/EKS | 6 | 300 |
| 43 | CloudFront + API Gateway | 6 | 300 |
| 44 | CloudFront + Lambda Function URLs | 6 | 300 |
| 45 | Standard Logs e Análise com Athena | 6 | 300 |
| 46 | Real-Time Logs com Kinesis | 6 | 400 |
| 47 | CloudFront + CloudWatch Métricas | 6 | 300 |
| 48 | Terraform — Distribuição Completa | 7 | 300 |
| 49 | CI/CD — Deploy Automatizado | 7 | 300 |
| 50 | Multi-Account com AWS Organizations | 7 | 400 |
| 51 | Cost Optimization Strategies | 8 | 300 |
| 52 | Continuous Functions | 8 | 400 |
| 53 | KeyValueStore | 8 | 400 |
| 54 | HTTP/3 e QUIC | 9 | 300 |
| 55 | CloudFront Hosting Toolkit | 9 | 300 |
| 56 | E-commerce Black Friday (Capstone) | 10 | 400 |
| 57 | Multi-Region Active-Active | 10 | 400 |
| 58 | Migração de CDN | 10 | 400 |
| 59 | Video Streaming Platform | 10 | 400 |
| 60 | Certificação e Próximos Passos | 10 | Meta |

### Mapa Visual de Conhecimento

```
                              CloudFront Expert
                                    │
                 ┌──────────────────┼──────────────────────┐
                 │                  │                      │
          FUNDAMENTOS          INTERMEDIÁRIO            AVANÇADO
                 │                  │                      │
         ┌───────┴───────┐   ┌─────┴──────┐    ┌─────────┴──────────┐
         │               │   │            │    │                    │
      Conceitos       Setup  │  Origins   │    │    Segurança       │
      ├─ CDN          ├─ S3  │  ├─ S3     │    │    ├─ WAF          │
      ├─ Edge         ├─ ACM │  ├─ ALB    │    │    ├─ Shield       │
      ├─ PoPs         ├─ R53 │  ├─ EC2    │    │    ├─ Signed URLs  │
      ├─ Cache        └─ OAC │  ├─ APIGW  │    │    ├─ Signed Cook  │
      ├─ TTL               │  └─ Lambda  │    │    ├─ FLE          │
      └─ Invalidation      │            │    │    └─ Geo-Restrict  │
                            │  Behaviors  │    │                    │
                            │  ├─ Path    │    │    Functions       │
                            │  ├─ Priority│    │    ├─ CF Functions  │
                            │  └─ Methods │    │    ├─ URL Rewrite  │
                            │            │    │    ├─ A/B Test     │
                            │  Policies   │    │    ├─ L@E Auth     │
                            │  ├─ Cache   │    │    └─ L@E Image    │
                            │  ├─ Origin  │    │                    │
                            │  └─ Response│    │    Integrações     │
                            └────────────┘    │    ├─ ECS/EKS      │
                                              │    ├─ API Gateway   │
                                              │    ├─ Logs/Athena   │
                                              │    ├─ Real-Time Logs│
                                              │    └─ CloudWatch    │
                                              │                    │
                                              │    Expert          │
                                              │    ├─ IaC (TF)     │
                                              │    ├─ CI/CD        │
                                              │    ├─ Multi-Region │
                                              │    ├─ Streaming    │
                                              │    ├─ Migration    │
                                              │    └─ Production   │
                                              └────────────────────┘
```

### Tópicos de Certificação AWS

Os conhecimentos deste guia se aplicam diretamente a três certificações:

#### AWS Certified Solutions Architect — Professional (SAP-C02)

**Tópicos cobertos:**
- Design de arquiteturas multi-tier com CDN
- Alta disponibilidade e disaster recovery
- Origin failover e multi-region
- Segurança em camadas (WAF + Shield + Signed URLs)
- Otimização de custos com cache strategies
- Migração de workloads para AWS

#### AWS Certified Advanced Networking — Specialty (ANS-C01)

**Tópicos cobertos:**
- CloudFront edge locations e routing
- DNS (Route 53) com latency-based routing
- TLS/SSL certificate management
- Origin access patterns
- HTTP/2, HTTP/3, WebSocket
- Network security (WAF, Shield, NACLs)

#### AWS Certified Security — Specialty (SCS-C02)

**Tópicos cobertos:**
- WAF rules e managed rule groups
- DDoS protection (Shield Standard/Advanced)
- Signed URLs e Signed Cookies
- Field-Level Encryption
- Origin Access Control (OAC)
- Security headers e CSP
- Bot mitigation

### Questões de Exemplo (Estilo Exame)

**Questão 1 (SAP-C02):**

Uma empresa global de e-commerce precisa servir conteúdo estático e dinâmico com latência
inferior a 100ms para usuários em todos os continentes. O conteúdo estático raramente muda,
mas o conteúdo dinâmico é personalizado por usuário. Qual arquitetura MELHOR atende esses
requisitos?

A) Uma distribuição CloudFront com cache de 1 ano para tudo e invalidação quando necessário.

B) Uma distribuição CloudFront com behaviors separados: cache longo para estático, cache
desabilitado para dinâmico, com Lambda@Edge para personalização.

C) Múltiplas distribuições CloudFront, uma por região, com Route 53 geolocation routing.

D) CloudFront com cache habilitado para tudo, usando cookies para variar o cache por usuário.

**Resposta: B**

*Explicação:* A opção B separa corretamente conteúdo estático (cache longo) de dinâmico
(sem cache ou cache curto). Lambda@Edge permite personalização sem comprometer o cache.
A opção A desperdiça invalidações. C é desnecessariamente complexa (CloudFront já é global).
D criaria uma explosão de cache keys (uma versão por usuário por URL), eliminando os
benefícios do cache.

---

**Questão 2 (ANS-C01):**

Um engenheiro de rede está configurando CloudFront para um site que precisa suportar tanto
HTTP/2 quanto HTTP/3. O certificado SSL é gerenciado pelo ACM em us-east-1. Ao testar,
HTTP/3 não funciona. Qual é a causa MAIS PROVÁVEL?

A) O certificado ACM precisa estar em sa-east-1 para suportar HTTP/3.

B) HTTP/3 requer que o viewer certificate use `vip` ao invés de `sni-only`.

C) HTTP/3 não está habilitado na distribuição (o padrão é `http2`).

D) O navegador do usuário não suporta HTTP/3 e precisa ser atualizado.

**Resposta: C**

*Explicação:* Por padrão, CloudFront usa `http2`. Para habilitar HTTP/3, você precisa
explicitamente configurar `http_version = "http2and3"` na distribuição. A opção A está
errada (ACM para CloudFront DEVE estar em us-east-1). B está errada (`vip` é para IPs
dedicados, não para protocolo). D é possível mas improvável.

---

**Questão 3 (SCS-C02):**

Uma empresa precisa proteger seu site contra web scraping automatizado que não é detectado
por rate limiting (os scrapers usam milhares de IPs diferentes). Qual combinação de controles
é MAIS eficaz?

A) WAF rate limiting com threshold mais baixo + IP reputation list.

B) WAF Bot Control (targeted) + CAPTCHA challenge action + CloudFront Functions para
verificar JavaScript execution.

C) Shield Advanced + WAF georestriction para bloquear países com mais scrapers.

D) Lambda@Edge para implementar fingerprinting de browser + blacklist de User-Agents.

**Resposta: B**

*Explicação:* WAF Bot Control com inspeção "targeted" usa machine learning para detectar
bots sofisticados mesmo quando usam IPs distribuídos. O CAPTCHA challenge adiciona uma
verificação que bots não conseguem resolver facilmente. A verificação de JavaScript execution
via CF Functions confirma que o client executa JS (bots simples não executam). A opção A
não funciona contra IPs distribuídos. C bloqueia usuários legítimos. D é parcial e
facilmente contornável.

---

**Questão 4 (SAP-C02):**

Uma equipe está migrando de Akamai para CloudFront. O site tem 100M pageviews/mês e zero
downtime é obrigatório. Qual estratégia de migração é MAIS segura?

A) Configurar CloudFront com todas as features, reduzir DNS TTL para 60s, fazer cutover
instantâneo e monitorar.

B) Configurar CloudFront, usar Route 53 weighted routing para enviar 5% do tráfego para
CloudFront, monitorar por 1 semana, aumentar gradualmente até 100%.

C) Configurar CloudFront como origin do Akamai, validar funcionalidade, depois remover Akamai.

D) Fazer o cutover durante uma janela de manutenção às 3h da manhã quando o tráfego é baixo.

**Resposta: B**

*Explicação:* A migração gradual com weighted routing é a abordagem mais segura porque
permite validar com tráfego real sem arriscar todo o tráfego. Se houver problema, apenas 5%
dos usuários são afetados. A opção A é "big bang" e arriscada. C adiciona latência
desnecessariamente (CDN na frente de CDN). D não elimina o risco, apenas reduz o impacto.

---

**Questão 5 (ANS-C01):**

Uma distribuição CloudFront está configurada com Origin Shield em `sa-east-1` e o cache hit
rate está em 60%. Após análise, você descobre que a cache key inclui o header `Accept-Language`
e os usuários enviam variações como `pt-BR`, `pt-br`, `pt-BR,pt;q=0.9`, `pt-BR, pt; q=0.9`.
Como aumentar o cache hit rate MAIS eficientemente?

A) Remover `Accept-Language` da cache key.

B) Usar uma CloudFront Function no viewer-request para normalizar o header `Accept-Language`
para um formato consistente antes que ele entre na cache key.

C) Aumentar o TTL do cache para compensar a fragmentação.

D) Habilitar Origin Shield em uma região adicional.

**Resposta: B**

*Explicação:* A normalização do header antes da avaliação da cache key garante que variações
do mesmo idioma resultem em cache hits. A opção A eliminaria a capacidade de servir conteúdo
em diferentes idiomas. C não resolve a fragmentação. D não afeta a cache key.

### Recursos para Continuar Aprendendo

#### Talks do re:Invent

- **NET301**: "Accelerate your content with Amazon CloudFront" (anual, overview atualizado)
- **NET401**: "Optimizing performance with CloudFront" (deep dive em performance)
- **SEC301**: "Protecting your web applications with AWS WAF" (segurança em detalhe)
- **NET302**: "CloudFront deep dive — what to expect" (novidades do serviço)
- **SVS404**: "Building serverless applications on the edge" (Lambda@Edge avançado)

#### Blogs Oficiais AWS

- AWS Networking & Content Delivery Blog: https://aws.amazon.com/blogs/networking-and-content-delivery/
- AWS Security Blog: https://aws.amazon.com/blogs/security/
- AWS Architecture Blog: https://aws.amazon.com/blogs/architecture/

#### Workshops e Labs

- AWS CDN Workshop: https://cdnworkshop.com
- AWS WAF Workshop: https://catalog.workshops.aws/waf
- AWS Serverless Workshop: https://serverlessworkshop.io

### Ideias de Projetos para Portfólio

#### Projeto 1: CDN Intelligence Dashboard

**Descrição:** Dashboard que analisa CloudFront real-time logs via Kinesis Data Streams,
processa com Lambda, armazena em DynamoDB e apresenta em um frontend React.

**Skills demonstrados:** Real-time analytics, serverless, data pipeline, CloudFront integration.

**Complexidade:** Media-Alta

#### Projeto 2: Multi-CDN Orchestrator

**Descrição:** Sistema que monitora performance de múltiplas CDNs (CloudFront, Cloudflare)
e roteia tráfego automaticamente para a CDN com melhor performance usando Route 53
health checks e RUM (Real User Monitoring).

**Skills demonstrados:** Multi-CDN strategy, DNS automation, performance monitoring.

**Complexidade:** Alta

#### Projeto 3: Serverless Image Optimization Service

**Descrição:** Serviço completo de otimização de imagens usando CloudFront + Lambda@Edge
que faz resize, format conversion (WebP/AVIF), quality adjustment baseado no device/network
do usuário, com cache inteligente.

**Skills demonstrados:** Lambda@Edge, image processing, device detection, cache strategies.

**Complexidade:** Media

### Comunidades

- **AWS Community Builders** — Programa oficial da AWS para builders ativos
- **AWS re:Post** — Fórum oficial para perguntas e respostas
- **Reddit r/aws** — Comunidade ativa de profissionais AWS
- **AWS User Groups Brasil** — Meetups presenciais e online
- **Telegram: AWS Brasil** — Grupo brasileiro de discussão

---

## Parabéns! Sua Jornada de Especialista CloudFront

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                                                                              ║
║     ██████╗ ██████╗ ███╗   ██╗ ██████╗██╗     ██╗   ██╗██╗██████╗  ██████╗  ║
║    ██╔════╝██╔═══██╗████╗  ██║██╔════╝██║     ██║   ██║██║██╔══██╗██╔═══██╗ ║
║    ██║     ██║   ██║██╔██╗ ██║██║     ██║     ██║   ██║██║██║  ██║██║   ██║ ║
║    ██║     ██║   ██║██║╚██╗██║██║     ██║     ██║   ██║██║██║  ██║██║   ██║ ║
║    ╚██████╗╚██████╔╝██║ ╚████║╚██████╗███████╗╚██████╔╝██║██████╔╝╚██████╔╝ ║
║     ╚═════╝ ╚═════╝ ╚═╝  ╚═══╝ ╚═════╝╚══════╝ ╚═════╝ ╚═╝╚═════╝  ╚═════╝ ║
║                                                                              ║
║                                                                              ║
║              Voce completou TODOS os 60 desafios!                            ║
║                                                                              ║
║              De iniciante a ESPECIALISTA CloudFront.                         ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

### Resumo de Todos os 60 Desafios

```
MODULO 1 — FUNDAMENTOS
  #01  [100] O Que e uma CDN e Por Que Usar
  #02  [100] Sua Primeira Distribuicao CloudFront
  #03  [100] CloudFront + S3 Static Website
  #04  [200] HTTPS e Certificados com ACM
  #05  [200] Custom Domain com Route 53
  #06  [200] Entendendo o Cache (Hit, Miss, TTL)
  #07  [200] Invalidacao de Cache
  #08  [200] Headers de Cache e Diagnostico

MODULO 2 — ORIGINS E BEHAVIORS
  #09  [200] Origin Access Control (OAC)
  #10  [200] Custom Origin (ALB/EC2)
  #11  [200] Multiple Origins e Behaviors
  #12  [300] Origin Groups (Failover)
  #13  [300] Origin Shield
  #14  [300] Custom Headers na Origem
  #15  [300] Origin Connection (Timeouts/Keepalive)
  #16  [300] Behaviors Avancados e Prioridade

MODULO 3 — POLICIES E CACHE
  #17  [200] Cache Policies Customizadas
  #18  [300] Origin Request Policies
  #19  [300] Response Headers Policies
  #20  [300] Cache Key e Normalizacao
  #21  [200] Managed Policies da AWS
  #22  [300] Cache por Device Type
  #23  [300] Cache por Geolocalizacao
  #24  [300] Cache Invalidation Strategies

MODULO 4 — FUNCTIONS E LAMBDA@EDGE
  #25  [200] Sua Primeira CloudFront Function
  #26  [300] URL Rewrite e Redirect com Functions
  #27  [300] A/B Testing com CloudFront Functions
  #28  [300] Security Headers com Functions
  #29  [300] Lambda@Edge — Primeiros Passos
  #30  [300] Lambda@Edge — Manipulacao de Origem
  #31  [400] Lambda@Edge — Autenticacao JWT
  #32  [400] Lambda@Edge — Image Resize on-the-fly

MODULO 5 — SEGURANCA AVANCADA
  #33  [300] AWS WAF v2 com CloudFront
  #34  [300] Rate Limiting com WAF
  #35  [300] Bot Management
  #36  [300] AWS Shield (Standard e Advanced)
  #37  [300] Signed URLs
  #38  [300] Signed Cookies
  #39  [200] Geo-Restriction
  #40  [400] Field-Level Encryption

MODULO 6 — INTEGRACOES AWS
  #41  [300] CloudFront + S3 (OAC Deep Dive)
  #42  [300] CloudFront + ALB + ECS/EKS
  #43  [300] CloudFront + API Gateway
  #44  [300] CloudFront + Lambda Function URLs
  #45  [300] Standard Logs e Analise com Athena
  #46  [400] Real-Time Logs com Kinesis
  #47  [300] CloudFront + CloudWatch Metricas

MODULO 7 — INFRAESTRUTURA COMO CODIGO
  #48  [300] Terraform — Distribuicao Completa
  #49  [300] CI/CD — Deploy Automatizado
  #50  [400] Multi-Account com AWS Organizations

MODULO 8 — FEATURES AVANCADAS
  #51  [300] Cost Optimization Strategies
  #52  [400] Continuous Functions
  #53  [400] KeyValueStore

MODULO 9 — PERFORMANCE E FERRAMENTAS
  #54  [300] HTTP/3 e QUIC
  #55  [300] CloudFront Hosting Toolkit

MODULO 10 — CENARIOS REAIS DE PRODUCAO
  #56  [400] E-commerce Black Friday (Capstone)
  #57  [400] Multi-Region Active-Active
  #58  [400] Migracao de CDN
  #59  [400] Video Streaming Platform
  #60  [META] Certificacao e Proximos Passos
```

### Checklist de Competencias — O Que Voce Domina Agora

```
FUNDAMENTOS
  [x] Entendo como CDNs funcionam e por que sao essenciais
  [x] Sei criar e configurar distribuicoes CloudFront
  [x] Domino HTTPS/TLS com ACM
  [x] Configuro custom domains com Route 53
  [x] Entendo mecanismos de cache (hit, miss, TTL)
  [x] Sei quando e como invalidar cache

ORIGINS E BEHAVIORS
  [x] Configuro OAC para S3 (mais seguro que OAI)
  [x] Configuro custom origins (ALB, EC2, API Gateway)
  [x] Crio behaviors com path patterns otimizados
  [x] Implemento origin failover com Origin Groups
  [x] Uso Origin Shield para reduzir carga na origem
  [x] Protejo origens com custom headers secretos

CACHE E POLICIES
  [x] Crio cache policies customizadas para cada caso
  [x] Configuro origin request policies corretamente
  [x] Implemento response headers de seguranca
  [x] Normalizo cache keys para maximizar hit rate
  [x] Uso managed policies quando apropriado
  [x] Vario cache por device, geo e idioma

FUNCTIONS E EDGE COMPUTING
  [x] Escrevo CloudFront Functions para logica simples
  [x] Implemento URL rewrite/redirect no edge
  [x] Crio testes A/B com distribuicao de trafego
  [x] Desenvolvo Lambda@Edge para logica complexa
  [x] Implemento autenticacao JWT no edge
  [x] Faco manipulacao de origem dinamica

SEGURANCA
  [x] Configuro WAF v2 com managed e custom rules
  [x] Implemento rate limiting inteligente
  [x] Uso Bot Control para proteger contra scrapers
  [x] Entendo Shield Standard vs Advanced
  [x] Gero e valido signed URLs e signed cookies
  [x] Aplico geo-restriction por distribuicao e por conteudo
  [x] Implemento security headers completos (HSTS, CSP, etc)

INTEGRACOES E OBSERVABILIDADE
  [x] Integro CloudFront com todos os servicos AWS relevantes
  [x] Analiso logs com Athena (standard logs)
  [x] Configuro real-time logs com Kinesis
  [x] Crio dashboards e alarmes no CloudWatch
  [x] Monitoro cache hit rate, error rate e latencia

INFRAESTRUTURA E DEVOPS
  [x] Gerencio CloudFront com Terraform (IaC completo)
  [x] Implemento CI/CD para deploy automatizado
  [x] Gerencio multi-account com Organizations

CENARIOS AVANCADOS
  [x] Projeto arquiteturas para alta carga (100k+ req/s)
  [x] Implemento multi-region active-active
  [x] Executo migracoes de CDN com zero downtime
  [x] Construo plataformas de video streaming (live + VOD)
  [x] Otimizo custos em cenarios de producao real
  [x] Planejo e executo load tests com k6
```

### O Que Faz de Voce um Especialista

Voce nao e mais alguem que "sabe usar CloudFront". Voce e alguem que:

1. **Entende os trade-offs** — Sabe quando usar cache longo vs curto, quando usar CloudFront
   Functions vs Lambda@Edge, quando usar signed URLs vs signed cookies.

2. **Pensa em producao** — Considera failover, monitoring, cost, security e escalabilidade
   em todas as decisoes de arquitetura.

3. **Resolve problemas complexos** — Migracoes, multi-region, streaming, e-commerce de alta
   carga. Nenhum cenario e simples demais ou complexo demais.

4. **Automatiza tudo** — Infraestrutura como codigo com Terraform, CI/CD, validacao
   automatizada. Nada manual em producao.

5. **Fala a lingua do negocio** — Traduz requisitos de negocio (latencia, disponibilidade,
   custo) em decisoes tecnicas fundamentadas.

---

### Mensagem Final

Quando voce comecou este guia, talvez CloudFront fosse apenas "aquele servico de CDN da AWS".
Agora voce sabe que e muito mais. E uma plataforma de edge computing completa, capaz de
executar logica, autenticar usuarios, otimizar conteudo, proteger aplicacoes e escalar para
milhoes de usuarios em milissegundos.

O conhecimento que voce adquiriu aqui nao e apenas tecnico. E a capacidade de olhar para um
problema complexo de distribuicao de conteudo e enxergar a solucao completa — da arquitetura
ao codigo, do Terraform ao monitoring, do primeiro commit ao go-live da Black Friday.

Continue construindo. Continue aprendendo. Continue compartilhando.

A nuvem evolui rapido, mas os fundamentos que voce domina agora — cache, edge computing,
seguranca em camadas, observabilidade, infraestrutura como codigo — sao atemporais.

Voce esta pronto para qualquer desafio.

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║   "The best way to predict the future is to build it."      ║
║                                              — Alan Kay      ║
║                                                              ║
║   Agora va construir coisas incriveis.                       ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

> **Guia CloudFront Especialista** — Modulo 10 de 10
> Todos os 60 desafios completos. De Level 100 a Level 400.
> Do basico ao expert. Da teoria a producao.
