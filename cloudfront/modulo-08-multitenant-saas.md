# Modulo 08 -- Multi-tenant, SaaS e Features Avancadas

> **Nivel:** 300-400 (Advanced/Expert)
> **Tempo Total Estimado:** 12-16 horas de labs
> **Objetivo do Modulo:** Dominar arquiteturas multi-tenant com CloudFront Distribution Tenants, padroes SaaS de isolamento e roteamento, Continuous Deployment (blue/green e canary), Static IPs, WebSocket e protocolos HTTP/2 e HTTP/3 (QUIC).

---

## Mapa do Modulo

```
                    ┌─────────────────────────────────┐
                    │   Modulo 08 — Multi-tenant/SaaS  │
                    └───────────────┬─────────────────┘
                                    │
        ┌───────────┬───────────┬───┴───┬───────────┬───────────┐
        │           │           │       │           │           │
   ┌────┴────┐ ┌────┴────┐ ┌───┴───┐ ┌─┴─────┐ ┌──┴───┐ ┌────┴────┐
   │Multi-   │ │SaaS     │ │Cont.  │ │Static │ │Web   │ │HTTP/2   │
   │tenant   │ │Archit.  │ │Deploy │ │IPs    │ │Socket│ │HTTP/3   │
   │Distrib. │ │CF+L@E   │ │B/G    │ │       │ │      │ │QUIC     │
   └─────────┘ └─────────┘ └───────┘ └───────┘ └──────┘ └─────────┘
     D.45        D.46        D.47      D.48      D.49      D.50
```

---

## Desafio 45: Multi-tenant Distributions (Distribution Tenants)

> **Level:** 400 | **Tempo:** 90 min | **Custo:** ~$1-3

### Objetivo

Entender e implementar o conceito de **Distribution Tenants** do CloudFront, onde uma unica distribution serve multiplos tenants, cada um com seu proprio dominio, certificado TLS, WAF e geo-restrictions.

### Contexto

Antes de Distribution Tenants, cada cliente SaaS precisava de sua propria CloudFront Distribution. Com 500 clientes, voce gerenciava 500 distributions separadas. Isso causava:

- **Explosao de recursos** — cada distribution com sua propria config
- **Limites de conta** — soft limit de 200 distributions por conta
- **Inconsistencia** — dificil manter todas sincronizadas
- **Custo operacional** — deploy de mudancas em todas as distributions

Distribution Tenants resolvem isso: **uma distribution pai** com ate **5.000 tenants**, cada um com configuracoes individualizadas.

### Arquitetura

```
                          Tenant A                Tenant B                Tenant C
                     app-a.example.com       app-b.client.com        custom.empresa.br
                     Cert: *.example.com     Cert: client.com        Cert: empresa.br
                     WAF: WebACL-A           WAF: WebACL-B           WAF: (nenhum)
                     Geo: Allow ALL          Geo: Block RU,CN        Geo: Allow BR only
                            │                       │                       │
                            └───────────┬───────────┘───────────────────────┘
                                        │
                                        ▼
                              ┌─────────────────────┐
                              │  CloudFront          │
                              │  Distribution (Pai)  │
                              │                      │
                              │  Behaviors:          │
                              │   /* → S3 Origin     │
                              │   /api/* → ALB       │
                              │                      │
                              │  Cache Policy:       │
                              │   CachingOptimized   │
                              └──────────┬──────────┘
                                         │
                              ┌──────────┴──────────┐
                              │                     │
                         ┌────┴────┐          ┌─────┴────┐
                         │   S3    │          │   ALB    │
                         │ Bucket  │          │  Origin  │
                         └─────────┘          └──────────┘
```

### Conceitos Fundamentais

| Conceito | Descricao |
|----------|-----------|
| **Distribution (Pai)** | A distribution principal que define behaviors, origins, cache policies |
| **Distribution Tenant** | Entidade filha que herda config da distribution pai, mas customiza dominio, cert, WAF, geo |
| **Connection Group** | Grupo que define como tenants se conectam (TLS, HTTP version) |
| **Tenant Customizations** | Overrides que cada tenant pode fazer: dominio, certificado, WAF, geo-restrictions |
| **Limite** | Ate 5.000 tenants por distribution |

### O Que Cada Tenant Pode Customizar

```
┌─────────────────────────────────────────────────────────────┐
│                    DISTRIBUTION PAI                          │
│  (define: origins, behaviors, cache policies, functions)    │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              TENANT CUSTOMIZATIONS                    │   │
│  │                                                       │   │
│  │  ✓ Dominios alternativos (CNAMEs)                    │   │
│  │  ✓ Certificado SSL/TLS (ACM)                         │   │
│  │  ✓ Web ACL (WAF)                                     │   │
│  │  ✓ Geo-restrictions (whitelist/blacklist)            │   │
│  │  ✓ Logging configuration                             │   │
│  │  ✓ Tags                                              │   │
│  │                                                       │   │
│  │  ✗ Behaviors (herda do pai)                          │   │
│  │  ✗ Origins (herda do pai)                            │   │
│  │  ✗ Cache Policies (herda do pai)                     │   │
│  │  ✗ Functions/Lambda@Edge (herda do pai)              │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Passo a Passo

#### Passo 1: Criar a Distribution Pai

```bash
# Criar bucket S3 para conteudo
aws s3 mb s3://multitenant-saas-content-$(aws sts get-caller-identity --query Account --output text)

# Upload de conteudo de exemplo
echo '<html><body><h1>Welcome, Tenant!</h1></body></html>' > index.html
aws s3 cp index.html s3://multitenant-saas-content-$(aws sts get-caller-identity --query Account --output text)/index.html

# Criar OAC
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "multitenant-oac",
    "Description": "OAC para multi-tenant distribution",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'
```

```bash
# Criar a distribution pai
# Salve o ID retornado — voce usara para criar tenants
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_DOMAIN="multitenant-saas-content-${ACCOUNT_ID}.s3.us-east-1.amazonaws.com"
OAC_ID=$(aws cloudfront list-origin-access-controls --query "OriginAccessControlList.Items[?Name=='multitenant-oac'].Id" --output text)

aws cloudfront create-distribution \
  --distribution-config "{
    \"CallerReference\": \"multitenant-$(date +%s)\",
    \"Comment\": \"Multi-tenant SaaS Distribution\",
    \"Enabled\": true,
    \"Origins\": {
      \"Quantity\": 1,
      \"Items\": [{
        \"Id\": \"S3-multitenant\",
        \"DomainName\": \"${BUCKET_DOMAIN}\",
        \"S3OriginConfig\": { \"OriginAccessIdentity\": \"\" },
        \"OriginAccessControlId\": \"${OAC_ID}\"
      }]
    },
    \"DefaultCacheBehavior\": {
      \"TargetOriginId\": \"S3-multitenant\",
      \"ViewerProtocolPolicy\": \"redirect-to-https\",
      \"AllowedMethods\": {
        \"Quantity\": 2,
        \"Items\": [\"GET\", \"HEAD\"]
      },
      \"CachePolicyId\": \"658327ea-f89d-4fab-a63d-7e88639e58f6\",
      \"Compress\": true
    },
    \"PriceClass\": \"PriceClass_100\",
    \"HttpVersion\": \"http2and3\",
    \"DefaultRootObject\": \"index.html\"
  }"

# Guardar o ID da distribution
DIST_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Comment=='Multi-tenant SaaS Distribution'].Id" \
  --output text)
echo "Distribution ID: $DIST_ID"
```

#### Passo 2: Criar Certificados ACM para os Tenants

```bash
# Certificado wildcard para tenants no dominio principal
aws acm request-certificate \
  --domain-name "*.saas-platform.example.com" \
  --validation-method DNS \
  --region us-east-1

# Certificado para dominio customizado de um cliente
aws acm request-certificate \
  --domain-name "cdn.clienteA.com.br" \
  --validation-method DNS \
  --region us-east-1

# Aguardar validacao DNS...
# (Crie os registros CNAME de validacao no DNS)
```

#### Passo 3: Criar Distribution Tenants

```bash
# Obter ARN do certificado wildcard
WILDCARD_CERT_ARN=$(aws acm list-certificates \
  --query "CertificateSummaryList[?DomainName=='*.saas-platform.example.com'].CertificateArn" \
  --output text --region us-east-1)

# Criar Tenant A — usa dominio do platform
aws cloudfront create-distribution-tenant \
  --distribution-id $DIST_ID \
  --name "tenant-clienteA" \
  --domains '{
    "Quantity": 1,
    "Items": [{
      "Domain": "clienteA.saas-platform.example.com"
    }]
  }' \
  --customizations '{
    "Certificate": {
      "Arn": "'"${WILDCARD_CERT_ARN}"'"
    },
    "GeoRestrictions": {
      "RestrictionType": "none",
      "Quantity": 0
    }
  }' \
  --tags '{
    "Items": [
      { "Key": "Tenant", "Value": "clienteA" },
      { "Key": "Plan", "Value": "enterprise" }
    ]
  }'
```

```bash
# Criar Tenant B — com geo-restriction
aws cloudfront create-distribution-tenant \
  --distribution-id $DIST_ID \
  --name "tenant-clienteB" \
  --domains '{
    "Quantity": 1,
    "Items": [{
      "Domain": "clienteB.saas-platform.example.com"
    }]
  }' \
  --customizations '{
    "Certificate": {
      "Arn": "'"${WILDCARD_CERT_ARN}"'"
    },
    "GeoRestrictions": {
      "RestrictionType": "blacklist",
      "Quantity": 2,
      "Items": ["RU", "CN"]
    }
  }' \
  --tags '{
    "Items": [
      { "Key": "Tenant", "Value": "clienteB" },
      { "Key": "Plan", "Value": "pro" }
    ]
  }'
```

```bash
# Criar Tenant C — com dominio customizado e WAF
CUSTOM_CERT_ARN=$(aws acm list-certificates \
  --query "CertificateSummaryList[?DomainName=='cdn.clienteA.com.br'].CertificateArn" \
  --output text --region us-east-1)

WAF_ACL_ARN="arn:aws:wafv2:us-east-1:${ACCOUNT_ID}:global/webacl/tenant-c-waf/example-id"

aws cloudfront create-distribution-tenant \
  --distribution-id $DIST_ID \
  --name "tenant-clienteC" \
  --domains '{
    "Quantity": 1,
    "Items": [{
      "Domain": "cdn.clienteA.com.br"
    }]
  }' \
  --customizations '{
    "Certificate": {
      "Arn": "'"${CUSTOM_CERT_ARN}"'"
    },
    "WebAcl": {
      "Arn": "'"${WAF_ACL_ARN}"'"
    },
    "GeoRestrictions": {
      "RestrictionType": "whitelist",
      "Quantity": 1,
      "Items": ["BR"]
    }
  }' \
  --tags '{
    "Items": [
      { "Key": "Tenant", "Value": "clienteC" },
      { "Key": "Plan", "Value": "enterprise" }
    ]
  }'
```

#### Passo 4: Listar e Gerenciar Tenants

```bash
# Listar todos os tenants de uma distribution
aws cloudfront list-distribution-tenants-by-distribution \
  --distribution-id $DIST_ID

# Obter detalhes de um tenant especifico
TENANT_ID="<tenant-id-retornado>"
aws cloudfront get-distribution-tenant --id $TENANT_ID

# Atualizar um tenant (ex: adicionar WAF)
aws cloudfront update-distribution-tenant \
  --id $TENANT_ID \
  --if-match $(aws cloudfront get-distribution-tenant --id $TENANT_ID --query "ETag" --output text) \
  --customizations '{
    "WebAcl": {
      "Arn": "arn:aws:wafv2:us-east-1:123456789012:global/webacl/new-waf/id"
    }
  }'

# Deletar um tenant
aws cloudfront delete-distribution-tenant \
  --id $TENANT_ID \
  --if-match $(aws cloudfront get-distribution-tenant --id $TENANT_ID --query "ETag" --output text)
```

#### Passo 5: Tenant Lifecycle Management

```bash
# Fluxo completo de onboarding de um novo cliente SaaS:

# 1. Solicitar certificado
NEW_CERT_ARN=$(aws acm request-certificate \
  --domain-name "cdn.novocliente.com" \
  --validation-method DNS \
  --region us-east-1 \
  --query "CertificateArn" --output text)

echo "Certificado solicitado: $NEW_CERT_ARN"
echo "Aguardando validacao DNS..."

# 2. Aguardar validacao
aws acm wait certificate-validated \
  --certificate-arn $NEW_CERT_ARN \
  --region us-east-1

# 3. Criar o tenant
aws cloudfront create-distribution-tenant \
  --distribution-id $DIST_ID \
  --name "tenant-novocliente" \
  --domains '{
    "Quantity": 1,
    "Items": [{"Domain": "cdn.novocliente.com"}]
  }' \
  --customizations '{
    "Certificate": {"Arn": "'"${NEW_CERT_ARN}"'"}
  }'

# 4. Informar o CNAME para o cliente apontar
echo "Cliente deve criar CNAME:"
echo "  cdn.novocliente.com → <distribution-domain>.cloudfront.net"

# 5. Verificar propagacao
dig cdn.novocliente.com CNAME
curl -I https://cdn.novocliente.com/
```

### Terraform

```hcl
# ============================================================
# Multi-tenant Distribution com Terraform
# ============================================================

variable "tenants" {
  description = "Mapa de tenants SaaS"
  type = map(object({
    domain           = string
    certificate_arn  = string
    waf_acl_arn      = optional(string)
    geo_restriction  = optional(string, "none")
    geo_countries    = optional(list(string), [])
    plan             = string
  }))
  default = {
    clienteA = {
      domain          = "clienteA.saas-platform.example.com"
      certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/abc-123"
      plan            = "enterprise"
    }
    clienteB = {
      domain           = "clienteB.saas-platform.example.com"
      certificate_arn  = "arn:aws:acm:us-east-1:123456789012:certificate/abc-123"
      geo_restriction  = "blacklist"
      geo_countries    = ["RU", "CN"]
      plan             = "pro"
    }
    clienteC = {
      domain           = "cdn.clienteC.com.br"
      certificate_arn  = "arn:aws:acm:us-east-1:123456789012:certificate/def-456"
      waf_acl_arn      = "arn:aws:wafv2:us-east-1:123456789012:global/webacl/tenant-c/id"
      geo_restriction  = "whitelist"
      geo_countries    = ["BR"]
      plan             = "enterprise"
    }
  }
}

# Distribution Pai
resource "aws_cloudfront_distribution" "multitenant" {
  comment             = "Multi-tenant SaaS Distribution"
  enabled             = true
  default_root_object = "index.html"
  http_version        = "http2and3"
  price_class         = "PriceClass_100"

  origin {
    domain_name              = aws_s3_bucket.content.bucket_regional_domain_name
    origin_id                = "S3-multitenant"
    origin_access_control_id = aws_cloudfront_origin_access_control.main.id
  }

  default_cache_behavior {
    target_origin_id       = "S3-multitenant"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6"
    compress               = true
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }

  tags = {
    Environment = "production"
    Type        = "multi-tenant"
  }
}

# Distribution Tenants
resource "aws_cloudfront_distribution_tenant" "tenants" {
  for_each = var.tenants

  distribution_id = aws_cloudfront_distribution.multitenant.id
  name            = "tenant-${each.key}"

  domains {
    domain = each.value.domain
  }

  customizations {
    certificate {
      arn = each.value.certificate_arn
    }

    dynamic "web_acl" {
      for_each = each.value.waf_acl_arn != null ? [1] : []
      content {
        arn = each.value.waf_acl_arn
      }
    }

    geo_restrictions {
      restriction_type = each.value.geo_restriction
      locations        = each.value.geo_countries
    }
  }

  tags = {
    Tenant = each.key
    Plan   = each.value.plan
  }
}

# Outputs
output "tenant_domains" {
  description = "Dominios dos tenants criados"
  value = {
    for k, v in aws_cloudfront_distribution_tenant.tenants :
    k => {
      domain      = v.domains[0].domain
      tenant_id   = v.id
      domain_name = v.domain_name
    }
  }
}
```

### Validacao

```bash
# 1. Verificar que a distribution pai esta ativa
aws cloudfront get-distribution --id $DIST_ID \
  --query "Distribution.Status"
# Esperado: "Deployed"

# 2. Listar tenants
aws cloudfront list-distribution-tenants-by-distribution \
  --distribution-id $DIST_ID \
  --query "DistributionTenantList[].{Name:Name,Id:Id,Status:Status}"

# 3. Testar acesso via dominio do tenant
curl -I https://clienteA.saas-platform.example.com/
# Esperado: HTTP/2 200, x-cache: Hit from cloudfront

# 4. Testar geo-restriction do Tenant B (de um IP bloqueado)
curl -I https://clienteB.saas-platform.example.com/
# Esperado: 403 Forbidden (se acessado de RU/CN)

# 5. Verificar WAF no Tenant C
curl -I https://cdn.clienteA.com.br/
# Headers WAF devem estar presentes

# 6. Verificar limites
aws service-quotas get-service-quota \
  --service-code cloudfront \
  --quota-code L-XXXXXXXX \
  --query "Quota.Value"
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| **Distribution Tenant** | Entidade filha que herda behaviors/origins da distribution pai |
| **Limite de tenants** | Ate 5.000 tenants por distribution (soft limit) |
| **Customizacoes** | Cada tenant pode ter: dominio, cert, WAF, geo-restriction proprios |
| **Heranca** | Behaviors, origins, cache policies sao herdados e nao customizaveis por tenant |
| **Lifecycle** | Tenants podem ser criados, atualizados e deletados independentemente |
| **Connection Group** | Agrupa tenants com mesma configuracao de conexao (TLS, HTTP version) |
| **Vantagem vs N distributions** | Uma unica distribution para gerenciar, deploy atomico de mudancas |

### Expert Tip

> **Multi-tenant vs Multi-distribution:** Use Distribution Tenants quando todos os tenants compartilham a mesma logica de behaviors/origins mas precisam de dominios e seguranca individualizados. Se tenants precisam de behaviors completamente diferentes (ex: um serve SPA, outro serve API), considere distributions separadas. O modelo hibrido tambem funciona: agrupe tenants por "tipo" em distributions pai diferentes.

---

## Desafio 46: Arquitetura SaaS com CloudFront

> **Level:** 400 | **Tempo:** 120 min | **Custo:** ~$2-5

### Objetivo

Projetar e implementar uma arquitetura SaaS completa com CloudFront, incluindo tenant resolution via CloudFront Functions, isolamento de cache, roteamento avancado com Lambda@Edge, e suporte a dominios customizados.

### Contexto

Voce esta construindo uma plataforma SaaS que serve multiplos clientes. Cada cliente pode acessar via:

1. **Subdominio da plataforma:** `client1.platform.com`, `client2.platform.com`
2. **Dominio customizado:** `app.meucliente.com.br`

A arquitetura precisa:
- Resolver o tenant a partir do request
- Isolar cache entre tenants
- Rotear para o backend correto
- Servir assets especificos do tenant (logo, tema)

### Arquitetura

```
                     client1.platform.com     app.meucliente.com.br
                              │                        │
                              └────────────┬───────────┘
                                           │
                                           ▼
                              ┌────────────────────────┐
                              │      CloudFront         │
                              │                         │
                              │  ┌──────────────────┐   │
                              │  │ CF Function       │   │
                              │  │ (Viewer Request)  │   │
                              │  │                   │   │
                              │  │ Host header →     │   │
                              │  │ X-Tenant-Id       │   │
                              │  │ X-Tenant-Tier     │   │
                              │  └────────┬─────────┘   │
                              │           │              │
                              │  ┌────────┴─────────┐   │
                              │  │ Lambda@Edge       │   │
                              │  │ (Origin Request)  │   │
                              │  │                   │   │
                              │  │ Dynamic origin    │   │
                              │  │ selection based   │   │
                              │  │ on tenant tier    │   │
                              │  └────────┬─────────┘   │
                              └───────────┼─────────────┘
                                          │
                        ┌─────────────────┼─────────────────┐
                        │                 │                  │
                   ┌────┴────┐     ┌──────┴──────┐    ┌─────┴─────┐
                   │   S3    │     │    ALB      │    │   ALB     │
                   │ Assets  │     │  Shared     │    │ Dedicated │
                   │ /tenant │     │  Backend    │    │ (Enter-   │
                   │ /logos  │     │  (Free/Pro) │    │  prise)   │
                   └─────────┘     └─────────────┘    └───────────┘
```

### Passo a Passo

#### Passo 1: CloudFront Function para Tenant Resolution

```javascript
// cf-function-tenant-resolver.js
// Resolve o tenant a partir do Host header e injeta headers

// Mapa de dominios customizados para tenant IDs
// Em producao, isso viria de um DynamoDB/Parameter Store via Lambda@Edge
var CUSTOM_DOMAIN_MAP = {
  'app.meucliente.com.br': { tenantId: 'cliente-abc', tier: 'enterprise' },
  'cdn.outrocliente.com':  { tenantId: 'cliente-xyz', tier: 'pro' }
};

var PLATFORM_DOMAIN = '.platform.example.com';

function handler(event) {
  var request = event.request;
  var host = request.headers.host.value.toLowerCase();
  var tenantId = 'unknown';
  var tenantTier = 'free';

  // Caso 1: Subdominio da plataforma
  // Ex: client1.platform.example.com → tenantId = client1
  if (host.endsWith(PLATFORM_DOMAIN)) {
    tenantId = host.replace(PLATFORM_DOMAIN, '');
    // Tier default para subdominios da plataforma
    tenantTier = 'pro';
  }
  // Caso 2: Dominio customizado
  else if (CUSTOM_DOMAIN_MAP[host]) {
    tenantId = CUSTOM_DOMAIN_MAP[host].tenantId;
    tenantTier = CUSTOM_DOMAIN_MAP[host].tier;
  }

  // Injetar headers para downstream
  request.headers['x-tenant-id'] = { value: tenantId };
  request.headers['x-tenant-tier'] = { value: tenantTier };

  // Adicionar tenant ao cache key via query string
  // (alternativa: usar cache policy com header no cache key)
  // request.querystring['_tenant'] = { value: tenantId };

  return request;
}
```

```bash
# Criar a CloudFront Function
aws cloudfront create-function \
  --name tenant-resolver \
  --function-config '{
    "Comment": "Resolve tenant from Host header",
    "Runtime": "cloudfront-js-2.0"
  }' \
  --function-code fileb://cf-function-tenant-resolver.js

# Publicar
FUNCTION_ETAG=$(aws cloudfront describe-function \
  --name tenant-resolver \
  --query "ETag" --output text)

aws cloudfront publish-function \
  --name tenant-resolver \
  --if-match $FUNCTION_ETAG
```

#### Passo 2: Lambda@Edge para Roteamento Avancado

```javascript
// lambda-edge-tenant-router.js
// Origin Request — redireciona para backend correto baseado no tier

'use strict';

const BACKENDS = {
  free: {
    domainName: 'shared-backend.alb.us-east-1.amazonaws.com',
    port: 443,
    protocol: 'https',
    path: ''
  },
  pro: {
    domainName: 'shared-backend.alb.us-east-1.amazonaws.com',
    port: 443,
    protocol: 'https',
    path: ''
  },
  enterprise: {
    domainName: 'dedicated-backend.alb.us-east-1.amazonaws.com',
    port: 443,
    protocol: 'https',
    path: ''
  }
};

exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const tenantId = request.headers['x-tenant-id']
    ? request.headers['x-tenant-id'][0].value
    : 'unknown';
  const tenantTier = request.headers['x-tenant-tier']
    ? request.headers['x-tenant-tier'][0].value
    : 'free';

  // Selecionar backend baseado no tier
  const backend = BACKENDS[tenantTier] || BACKENDS.free;

  // Atualizar origin
  request.origin = {
    custom: {
      domainName: backend.domainName,
      port: backend.port,
      protocol: backend.protocol,
      path: backend.path,
      sslProtocols: ['TLSv1.2'],
      readTimeout: 30,
      keepaliveTimeout: 5,
      customHeaders: {
        'x-tenant-id': [{
          key: 'X-Tenant-Id',
          value: tenantId
        }],
        'x-tenant-tier': [{
          key: 'X-Tenant-Tier',
          value: tenantTier
        }]
      }
    }
  };

  // Atualizar Host header para o ALB
  request.headers['host'] = [{ key: 'Host', value: backend.domainName }];

  // Para assets do tenant, reescrever path
  // /assets/logo.png → /tenants/{tenantId}/assets/logo.png
  if (request.uri.startsWith('/assets/')) {
    request.uri = `/tenants/${tenantId}${request.uri}`;
  }

  return request;
};
```

```bash
# Criar a funcao Lambda@Edge
cd /tmp && mkdir -p tenant-router && cd tenant-router
cp /path/to/lambda-edge-tenant-router.js index.js
zip function.zip index.js

aws lambda create-function \
  --function-name cloudfront-tenant-router \
  --runtime nodejs20.x \
  --handler index.handler \
  --role arn:aws:iam::${ACCOUNT_ID}:role/lambda-edge-role \
  --zip-file fileb://function.zip \
  --region us-east-1

# Publicar versao (obrigatorio para Lambda@Edge)
LAMBDA_VERSION=$(aws lambda publish-version \
  --function-name cloudfront-tenant-router \
  --region us-east-1 \
  --query "Version" --output text)

LAMBDA_ARN="arn:aws:lambda:us-east-1:${ACCOUNT_ID}:function:cloudfront-tenant-router:${LAMBDA_VERSION}"
echo "Lambda ARN: $LAMBDA_ARN"
```

#### Passo 3: Cache Policy com Isolamento por Tenant

```bash
# Cache policy que inclui Host header no cache key
# Isso garante isolamento de cache entre tenants
aws cloudfront create-cache-policy \
  --cache-policy-config '{
    "Name": "SaaS-TenantIsolated-Cache",
    "Comment": "Cache policy com isolamento por tenant via Host header",
    "DefaultTTL": 86400,
    "MaxTTL": 31536000,
    "MinTTL": 0,
    "ParametersInCacheKeyAndForwardedToOrigin": {
      "EnableAcceptEncodingGzip": true,
      "EnableAcceptEncodingBrotli": true,
      "HeadersConfig": {
        "HeaderBehavior": "whitelist",
        "Headers": {
          "Quantity": 1,
          "Items": ["Host"]
        }
      },
      "CookiesConfig": {
        "CookieBehavior": "none"
      },
      "QueryStringsConfig": {
        "QueryStringBehavior": "none"
      }
    }
  }'
```

**Por que Host no cache key?**

```
Sem Host no cache key:
  client1.platform.com/index.html  →  Cache Key: /index.html  ← PROBLEMA!
  client2.platform.com/index.html  →  Cache Key: /index.html  ← Mesmo cache!

Com Host no cache key:
  client1.platform.com/index.html  →  Cache Key: client1.platform.com/index.html  ✓
  client2.platform.com/index.html  →  Cache Key: client2.platform.com/index.html  ✓
```

#### Passo 4: Origin Request Policy

```bash
# Origin request policy que envia headers do tenant para o origin
aws cloudfront create-origin-request-policy \
  --origin-request-policy-config '{
    "Name": "SaaS-TenantHeaders-ORP",
    "Comment": "Envia headers de tenant para o origin",
    "HeadersConfig": {
      "HeaderBehavior": "whitelist",
      "Headers": {
        "Quantity": 3,
        "Items": ["X-Tenant-Id", "X-Tenant-Tier", "Host"]
      }
    },
    "CookiesConfig": {
      "CookieBehavior": "all"
    },
    "QueryStringsConfig": {
      "QueryStringBehavior": "all"
    }
  }'
```

#### Passo 5: Wildcard Certificate e DNS

```bash
# Certificado wildcard para a plataforma
aws acm request-certificate \
  --domain-name "*.platform.example.com" \
  --subject-alternative-names "platform.example.com" \
  --validation-method DNS \
  --region us-east-1

# Para cada cliente com dominio customizado:
# O cliente cria um CNAME apontando para a distribution
# Ex: app.meucliente.com.br CNAME d1234567.cloudfront.net
```

### Terraform Completo

```hcl
# ============================================================
# Arquitetura SaaS Completa com CloudFront
# ============================================================

terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

locals {
  platform_domain = "platform.example.com"

  tenants = {
    client1 = {
      subdomain = "client1"
      tier      = "pro"
    }
    client2 = {
      subdomain = "client2"
      tier      = "enterprise"
    }
  }
}

# ---- S3: Assets por tenant ----
resource "aws_s3_bucket" "tenant_assets" {
  bucket = "saas-tenant-assets-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_policy" "tenant_assets" {
  bucket = aws_s3_bucket.tenant_assets.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowCloudFrontOAC"
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.tenant_assets.arn}/*"
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.saas.arn
        }
      }
    }]
  })
}

# ---- OAC ----
resource "aws_cloudfront_origin_access_control" "s3" {
  name                              = "saas-s3-oac"
  description                       = "OAC para assets de tenants"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# ---- CloudFront Function: Tenant Resolver ----
resource "aws_cloudfront_function" "tenant_resolver" {
  name    = "tenant-resolver"
  runtime = "cloudfront-js-2.0"
  comment = "Resolve tenant a partir do Host header"
  publish = true
  code    = file("${path.module}/functions/tenant-resolver.js")
}

# ---- Lambda@Edge: Tenant Router ----
data "aws_iam_policy_document" "lambda_edge_assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com", "edgelambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda_edge" {
  name               = "saas-lambda-edge-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_edge_assume.json
}

resource "aws_iam_role_policy_attachment" "lambda_edge_basic" {
  role       = aws_iam_role.lambda_edge.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

data "archive_file" "tenant_router" {
  type        = "zip"
  source_file = "${path.module}/functions/tenant-router.js"
  output_path = "${path.module}/dist/tenant-router.zip"
}

resource "aws_lambda_function" "tenant_router" {
  function_name    = "cloudfront-tenant-router"
  role             = aws_iam_role.lambda_edge.arn
  handler          = "tenant-router.handler"
  runtime          = "nodejs20.x"
  filename         = data.archive_file.tenant_router.output_path
  source_code_hash = data.archive_file.tenant_router.output_base64sha256
  publish          = true  # Lambda@Edge precisa de versao publicada

  provider = aws.us_east_1  # Lambda@Edge deve estar em us-east-1
}

# ---- Cache Policy: Tenant Isolated ----
resource "aws_cloudfront_cache_policy" "tenant_isolated" {
  name        = "SaaS-TenantIsolated"
  comment     = "Cache isolado por tenant via Host header"
  default_ttl = 86400
  max_ttl     = 31536000
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    enable_accept_encoding_gzip   = true
    enable_accept_encoding_brotli = true

    headers_config {
      header_behavior = "whitelist"
      headers {
        items = ["Host"]
      }
    }

    cookies_config {
      cookie_behavior = "none"
    }

    query_strings_config {
      query_string_behavior = "none"
    }
  }
}

# ---- Origin Request Policy ----
resource "aws_cloudfront_origin_request_policy" "tenant_headers" {
  name    = "SaaS-TenantHeaders"
  comment = "Forwarda headers de tenant para origin"

  headers_config {
    header_behavior = "whitelist"
    headers {
      items = ["X-Tenant-Id", "X-Tenant-Tier", "Host"]
    }
  }

  cookies_config {
    cookie_behavior = "all"
  }

  query_strings_config {
    query_string_behavior = "all"
  }
}

# ---- ACM: Wildcard Certificate ----
resource "aws_acm_certificate" "platform" {
  domain_name               = "*.${local.platform_domain}"
  subject_alternative_names = [local.platform_domain]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

# ---- CloudFront Distribution ----
resource "aws_cloudfront_distribution" "saas" {
  comment             = "SaaS Platform Distribution"
  enabled             = true
  default_root_object = "index.html"
  http_version        = "http2and3"
  price_class         = "PriceClass_100"
  is_ipv6_enabled     = true

  aliases = concat(
    [local.platform_domain],
    [for k, v in local.tenants : "${v.subdomain}.${local.platform_domain}"]
  )

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.platform.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # Origin 1: S3 para assets estaticos
  origin {
    domain_name              = aws_s3_bucket.tenant_assets.bucket_regional_domain_name
    origin_id                = "S3-Assets"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  # Origin 2: ALB Shared (Free/Pro)
  origin {
    domain_name = "shared-backend.internal.example.com"
    origin_id   = "ALB-Shared"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Origin 3: ALB Dedicated (Enterprise)
  origin {
    domain_name = "dedicated-backend.internal.example.com"
    origin_id   = "ALB-Dedicated"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Behavior: Assets estaticos (/* .css, .js, .png, etc.)
  ordered_cache_behavior {
    path_pattern             = "/assets/*"
    target_origin_id         = "S3-Assets"
    viewer_protocol_policy   = "redirect-to-https"
    allowed_methods          = ["GET", "HEAD"]
    cached_methods           = ["GET", "HEAD"]
    cache_policy_id          = aws_cloudfront_cache_policy.tenant_isolated.id
    compress                 = true

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.tenant_resolver.arn
    }

    lambda_function_association {
      event_type   = "origin-request"
      lambda_arn   = aws_lambda_function.tenant_router.qualified_arn
      include_body = false
    }
  }

  # Behavior: API
  ordered_cache_behavior {
    path_pattern             = "/api/*"
    target_origin_id         = "ALB-Shared"
    viewer_protocol_policy   = "https-only"
    allowed_methods          = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods           = ["GET", "HEAD"]
    cache_policy_id          = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"  # CachingDisabled
    origin_request_policy_id = aws_cloudfront_origin_request_policy.tenant_headers.id
    compress                 = true

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.tenant_resolver.arn
    }

    lambda_function_association {
      event_type   = "origin-request"
      lambda_arn   = aws_lambda_function.tenant_router.qualified_arn
      include_body = true
    }
  }

  # Default behavior
  default_cache_behavior {
    target_origin_id         = "ALB-Shared"
    viewer_protocol_policy   = "redirect-to-https"
    allowed_methods          = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods           = ["GET", "HEAD"]
    cache_policy_id          = aws_cloudfront_cache_policy.tenant_isolated.id
    origin_request_policy_id = aws_cloudfront_origin_request_policy.tenant_headers.id
    compress                 = true

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.tenant_resolver.arn
    }

    lambda_function_association {
      event_type   = "origin-request"
      lambda_arn   = aws_lambda_function.tenant_router.qualified_arn
      include_body = false
    }
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = {
    Environment = "production"
    Service     = "saas-platform"
  }
}

# ---- Data Sources ----
data "aws_caller_identity" "current" {}

# ---- Outputs ----
output "distribution_domain" {
  value = aws_cloudfront_distribution.saas.domain_name
}

output "distribution_id" {
  value = aws_cloudfront_distribution.saas.id
}

output "tenant_urls" {
  value = {
    for k, v in local.tenants :
    k => "https://${v.subdomain}.${local.platform_domain}"
  }
}
```

### Validacao

```bash
# 1. Testar resolucao de tenant via subdominio
curl -v https://client1.platform.example.com/api/whoami
# Response deve conter: {"tenantId": "client1", "tier": "pro"}

# 2. Testar dominio customizado
curl -v https://app.meucliente.com.br/api/whoami
# Response deve conter: {"tenantId": "cliente-abc", "tier": "enterprise"}

# 3. Verificar isolamento de cache
# Acessar mesmo path por tenants diferentes
curl -s https://client1.platform.example.com/assets/config.json | jq .tenant
# "client1"
curl -s https://client2.platform.example.com/assets/config.json | jq .tenant
# "client2"

# 4. Verificar headers de resposta
curl -sI https://client1.platform.example.com/ | grep -i x-cache
# X-Cache: Miss from cloudfront (primeira vez)
# X-Cache: Hit from cloudfront (segunda vez)

# 5. Testar que tenants diferentes tem cache keys diferentes
curl -sI https://client1.platform.example.com/page.html | grep x-amz-cf-id
curl -sI https://client2.platform.example.com/page.html | grep x-amz-cf-id
# IDs devem ser diferentes (cache entries separadas)
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| **Tenant Resolution** | CloudFront Function extrai tenant do Host header e injeta X-Tenant-Id |
| **Cache Isolation** | Incluir Host no cache key garante que tenants nao compartilhem cache |
| **Dynamic Origin** | Lambda@Edge seleciona origin baseado no tier do tenant |
| **Wildcard Certs** | Um certificado *.platform.com serve todos os subdominios |
| **Custom Domains** | Clientes enterprise podem usar seu proprio dominio |
| **Origin Request Policy** | Envia headers de tenant para o backend processar |
| **CF Function vs L@E** | CF Function para logica simples (viewer), L@E para logica complexa (origin) |

### Expert Tip

> **Performance da resolucao de tenant:** CloudFront Functions executam em menos de 1ms e sao ideais para tenant resolution. Evite chamar DynamoDB ou qualquer servico externo em CF Functions (nao e permitido). Se precisar de lookup dinamico de dominios customizados, use Lambda@Edge no evento origin-request (com cache local em memoria para evitar chamadas repetidas ao DynamoDB). Uma Lambda@Edge com 128MB e lookup em DynamoDB adiciona ~5-15ms, e o resultado fica cacheado no CloudFront.

---

## Desafio 47: Continuous Deployment -- Blue/Green e Canary

> **Level:** 400 | **Tempo:** 90 min | **Custo:** ~$1-2

### Objetivo

Implementar Continuous Deployment no CloudFront usando staging distributions, traffic shifting (weight-based e header-based), session stickiness, e workflows de promote/rollback.

### Contexto

Mudancas em CloudFront distributions sao globais e afetam todos os edge locations. Um erro de configuracao pode derrubar todo o trafego. Continuous Deployment permite:

1. Criar uma **staging distribution** (copia da producao)
2. Enviar uma **porcentagem do trafego** para staging
3. Validar com **headers especificos** ou **peso**
4. **Promover** staging para producao ou **rollback**

### Arquitetura

```
                          Usuarios (100%)
                               │
                               ▼
                    ┌──────────────────────┐
                    │   CloudFront Edge    │
                    │                      │
                    │  Continuous Deploy   │
                    │  Policy:            │
                    │  ┌────────────────┐  │
                    │  │ Weight: 90/10  │  │
                    │  │  ou            │  │
                    │  │ Header-based   │  │
                    │  └───────┬────────┘  │
                    └──────────┼───────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
              ┌─────┴──────┐       ┌──────┴─────┐
              │  Primary   │       │  Staging    │
              │  Distrib.  │       │  Distrib.   │
              │            │       │             │
              │  Config v1 │       │  Config v2  │
              │  (90%)     │       │  (10%)      │
              └─────┬──────┘       └──────┬──────┘
                    │                     │
                    └──────────┬──────────┘
                               │
                          ┌────┴────┐
                          │ Origin  │
                          │ (mesmo) │
                          └─────────┘

Fluxo Temporal:
  ┌─────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
  │ Copy    │ →  │ Create   │ →  │ Shift     │ →  │ Promote  │
  │ Distrib │    │ CD Policy│    │ Traffic   │    │ or       │
  │         │    │          │    │ 5%→10%→50%│    │ Rollback │
  └─────────┘    └──────────┘    └───────────┘    └──────────┘
```

### Modos de Traffic Shifting

```
┌─────────────────────────────────────────────────────────────┐
│                 WEIGHT-BASED (Canary)                        │
│                                                             │
│  X% do trafego vai para staging                             │
│  Usuarios sao atribuidos aleatoriamente                     │
│  Session stickiness via cookie AWS-CF-CD-*                  │
│                                                             │
│  Exemplo: 5% → staging, 95% → primary                      │
│           Gradual: 5% → 10% → 25% → 50% → promote          │
├─────────────────────────────────────────────────────────────┤
│                 HEADER-BASED (Blue/Green)                    │
│                                                             │
│  Apenas requests com header especifico vao para staging     │
│  Ideal para testes internos antes de abrir para usuarios    │
│                                                             │
│  Exemplo: Header "aws-cf-cd-staging: true"                  │
│           Apenas testers com esse header veem staging        │
└─────────────────────────────────────────────────────────────┘
```

### Passo a Passo

#### Passo 1: Identificar a Distribution de Producao

```bash
# Vamos usar uma distribution existente como "producao"
PROD_DIST_ID="E1ABCDEF123456"  # Substitua pelo seu ID

# Verificar configuracao atual
aws cloudfront get-distribution-config --id $PROD_DIST_ID \
  --query "DistributionConfig.Comment"
# "Production SaaS Distribution"

# Salvar ETag (necessario para operacoes de update)
PROD_ETAG=$(aws cloudfront get-distribution --id $PROD_DIST_ID \
  --query "ETag" --output text)
echo "Production ETag: $PROD_ETAG"
```

#### Passo 2: Criar a Staging Distribution (Copy)

```bash
# copy-distribution cria uma copia exata da distribution de producao
# A staging distribution tera seu proprio ID mas compartilha o mesmo dominio
aws cloudfront copy-distribution \
  --primary-distribution-id $PROD_DIST_ID \
  --caller-reference "staging-$(date +%s)" \
  --if-match $PROD_ETAG \
  --enabled

# Salvar o ID da staging distribution
STAGING_DIST_ID=$(aws cloudfront copy-distribution \
  --primary-distribution-id $PROD_DIST_ID \
  --caller-reference "staging-$(date +%s)" \
  --if-match $PROD_ETAG \
  --enabled \
  --query "Distribution.Id" --output text)

echo "Staging Distribution ID: $STAGING_DIST_ID"

# Aguardar deploy da staging
aws cloudfront wait distribution-deployed --id $STAGING_DIST_ID
echo "Staging distribution deployed!"
```

#### Passo 3: Modificar a Staging Distribution

```bash
# Agora faca as mudancas desejadas na staging
# Exemplo: mudar cache policy, adicionar header, etc.

STAGING_CONFIG=$(aws cloudfront get-distribution-config --id $STAGING_DIST_ID)
STAGING_ETAG=$(echo $STAGING_CONFIG | jq -r '.ETag')

# Extrair e modificar a config
echo $STAGING_CONFIG | jq '.DistributionConfig' > /tmp/staging-config.json

# Editar /tmp/staging-config.json com as mudancas desejadas
# Exemplo: trocar o Comment para identificar a versao
jq '.Comment = "Staging - v2.0 - New cache policy"' /tmp/staging-config.json > /tmp/staging-config-v2.json

# Aplicar mudancas
aws cloudfront update-distribution \
  --id $STAGING_DIST_ID \
  --if-match $STAGING_ETAG \
  --distribution-config file:///tmp/staging-config-v2.json

# Aguardar deploy
aws cloudfront wait distribution-deployed --id $STAGING_DIST_ID
```

#### Passo 4: Criar Continuous Deployment Policy (Header-Based)

```bash
# Primeiro, testar com header-based (blue/green)
# Apenas requests com o header especifico vao para staging

aws cloudfront create-continuous-deployment-policy \
  --continuous-deployment-policy-config '{
    "StagingDistributionDnsNames": {
      "Quantity": 1,
      "Items": ["'$STAGING_DIST_ID'.cloudfront.net"]
    },
    "Enabled": true,
    "TrafficConfig": {
      "Type": "SingleHeader",
      "SingleHeaderConfig": {
        "Header": "aws-cf-cd-staging",
        "Value": "true"
      }
    }
  }'

CD_POLICY_ID=$(aws cloudfront list-continuous-deployment-policies \
  --query "ContinuousDeploymentPolicyList.Items[0].ContinuousDeploymentPolicy.Id" \
  --output text)
echo "CD Policy ID: $CD_POLICY_ID"
```

```bash
# Associar a policy a distribution de producao
PROD_CONFIG=$(aws cloudfront get-distribution-config --id $PROD_DIST_ID)
PROD_ETAG=$(echo $PROD_CONFIG | jq -r '.ETag')

echo $PROD_CONFIG | jq ".DistributionConfig.ContinuousDeploymentPolicyId = \"$CD_POLICY_ID\"" \
  | jq '.DistributionConfig' > /tmp/prod-config-cd.json

aws cloudfront update-distribution \
  --id $PROD_DIST_ID \
  --if-match $PROD_ETAG \
  --distribution-config file:///tmp/prod-config-cd.json

aws cloudfront wait distribution-deployed --id $PROD_DIST_ID
```

#### Passo 5: Testar com Header (Blue/Green)

```bash
# Sem header → vai para producao (config v1)
curl -sI https://meusite.example.com/ | grep -E "x-cache|server|x-amz"

# Com header → vai para staging (config v2)
curl -sI -H "aws-cf-cd-staging: true" https://meusite.example.com/ | grep -E "x-cache|server|x-amz"

# Comparar respostas
echo "=== Producao ==="
curl -s https://meusite.example.com/api/version
echo ""
echo "=== Staging ==="
curl -s -H "aws-cf-cd-staging: true" https://meusite.example.com/api/version
```

#### Passo 6: Mudar para Weight-Based (Canary)

```bash
# Depois de validar com headers, mudar para weight-based
CD_ETAG=$(aws cloudfront get-continuous-deployment-policy \
  --id $CD_POLICY_ID --query "ETag" --output text)

aws cloudfront update-continuous-deployment-policy \
  --id $CD_POLICY_ID \
  --if-match $CD_ETAG \
  --continuous-deployment-policy-config '{
    "StagingDistributionDnsNames": {
      "Quantity": 1,
      "Items": ["'$STAGING_DIST_ID'.cloudfront.net"]
    },
    "Enabled": true,
    "TrafficConfig": {
      "Type": "SingleWeight",
      "SingleWeightConfig": {
        "Weight": 0.05,
        "SessionStickinessConfig": {
          "IdleTTL": 300,
          "MaximumTTL": 600
        }
      }
    }
  }'

echo "5% do trafego agora vai para staging"
```

```bash
# Aumentar gradualmente
# 5% → 15% → 50%

# Esperar e validar metricas...
# Se tudo OK, aumentar para 15%
CD_ETAG=$(aws cloudfront get-continuous-deployment-policy \
  --id $CD_POLICY_ID --query "ETag" --output text)

aws cloudfront update-continuous-deployment-policy \
  --id $CD_POLICY_ID \
  --if-match $CD_ETAG \
  --continuous-deployment-policy-config '{
    "StagingDistributionDnsNames": {
      "Quantity": 1,
      "Items": ["'$STAGING_DIST_ID'.cloudfront.net"]
    },
    "Enabled": true,
    "TrafficConfig": {
      "Type": "SingleWeight",
      "SingleWeightConfig": {
        "Weight": 0.15,
        "SessionStickinessConfig": {
          "IdleTTL": 300,
          "MaximumTTL": 600
        }
      }
    }
  }'

echo "15% do trafego agora vai para staging"
```

#### Passo 7: Promover Staging para Producao

```bash
# Quando satisfeito, promover staging para producao
# Isso copia a configuracao do staging para o primary

PROD_ETAG=$(aws cloudfront get-distribution --id $PROD_DIST_ID \
  --query "ETag" --output text)
STAGING_ETAG=$(aws cloudfront get-distribution --id $STAGING_DIST_ID \
  --query "ETag" --output text)

aws cloudfront update-distribution-with-staging-config \
  --id $PROD_DIST_ID \
  --staging-distribution-id $STAGING_DIST_ID \
  --if-match "$PROD_ETAG,$STAGING_ETAG"

echo "Staging promovido para producao!"

# Aguardar deploy
aws cloudfront wait distribution-deployed --id $PROD_DIST_ID
```

#### Passo 8: Rollback (Se Necessario)

```bash
# OPCAO 1: Desabilitar a CD policy (volta 100% para producao)
CD_ETAG=$(aws cloudfront get-continuous-deployment-policy \
  --id $CD_POLICY_ID --query "ETag" --output text)

aws cloudfront update-continuous-deployment-policy \
  --id $CD_POLICY_ID \
  --if-match $CD_ETAG \
  --continuous-deployment-policy-config '{
    "StagingDistributionDnsNames": {
      "Quantity": 1,
      "Items": ["'$STAGING_DIST_ID'.cloudfront.net"]
    },
    "Enabled": false,
    "TrafficConfig": {
      "Type": "SingleWeight",
      "SingleWeightConfig": {
        "Weight": 0.0
      }
    }
  }'

echo "Rollback completo! 100% do trafego de volta para producao"

# OPCAO 2: Apenas reduzir peso para 0
# (mantem a policy ativa mas sem trafego para staging)
```

#### Passo 9: Cleanup

```bash
# Depois de promover, limpar recursos

# 1. Remover CD policy da distribution
PROD_CONFIG=$(aws cloudfront get-distribution-config --id $PROD_DIST_ID)
PROD_ETAG=$(echo $PROD_CONFIG | jq -r '.ETag')
echo $PROD_CONFIG | jq '.DistributionConfig.ContinuousDeploymentPolicyId = ""' \
  | jq '.DistributionConfig' > /tmp/prod-no-cd.json

aws cloudfront update-distribution \
  --id $PROD_DIST_ID \
  --if-match $PROD_ETAG \
  --distribution-config file:///tmp/prod-no-cd.json

# 2. Deletar CD policy
CD_ETAG=$(aws cloudfront get-continuous-deployment-policy \
  --id $CD_POLICY_ID --query "ETag" --output text)
aws cloudfront delete-continuous-deployment-policy \
  --id $CD_POLICY_ID \
  --if-match $CD_ETAG

# 3. Desabilitar e deletar staging distribution
STAGING_CONFIG=$(aws cloudfront get-distribution-config --id $STAGING_DIST_ID)
STAGING_ETAG=$(echo $STAGING_CONFIG | jq -r '.ETag')
echo $STAGING_CONFIG | jq '.DistributionConfig.Enabled = false' \
  | jq '.DistributionConfig' > /tmp/staging-disable.json

aws cloudfront update-distribution \
  --id $STAGING_DIST_ID \
  --if-match $STAGING_ETAG \
  --distribution-config file:///tmp/staging-disable.json

aws cloudfront wait distribution-deployed --id $STAGING_DIST_ID

STAGING_ETAG=$(aws cloudfront get-distribution --id $STAGING_DIST_ID \
  --query "ETag" --output text)
aws cloudfront delete-distribution --id $STAGING_DIST_ID --if-match $STAGING_ETAG
```

### Terraform

```hcl
# ============================================================
# Continuous Deployment com Terraform
# ============================================================

# Distribution de Producao (ja existente)
resource "aws_cloudfront_distribution" "production" {
  comment             = "Production Distribution"
  enabled             = true
  # ... (configuracao completa)

  # Associar CD policy
  continuous_deployment_policy_id = aws_cloudfront_continuous_deployment_policy.canary.id
}

# Staging Distribution (copia da producao com mudancas)
resource "aws_cloudfront_distribution" "staging" {
  comment             = "Staging Distribution - v2.0"
  enabled             = true
  staging             = true  # Marca como staging

  # Mesma configuracao da producao, mas com as mudancas
  # Exemplo: nova cache policy
  default_cache_behavior {
    # ... config atualizada
    cache_policy_id = aws_cloudfront_cache_policy.new_policy.id
  }

  # ... resto da config
}

# Continuous Deployment Policy
resource "aws_cloudfront_continuous_deployment_policy" "canary" {
  enabled = true

  staging_distribution_dns_names {
    items    = [aws_cloudfront_distribution.staging.domain_name]
    quantity = 1
  }

  traffic_config {
    type = "SingleWeight"

    single_weight_config {
      weight = 0.10  # 10% para staging

      session_stickiness_config {
        idle_ttl    = 300
        maximum_ttl = 600
      }
    }
  }
}

# Para header-based:
# resource "aws_cloudfront_continuous_deployment_policy" "bluegreen" {
#   enabled = true
#
#   staging_distribution_dns_names {
#     items    = [aws_cloudfront_distribution.staging.domain_name]
#     quantity = 1
#   }
#
#   traffic_config {
#     type = "SingleHeader"
#
#     single_header_config {
#       header = "aws-cf-cd-staging"
#       value  = "true"
#     }
#   }
# }
```

### Validacao

```bash
# 1. Verificar CD policy esta ativa
aws cloudfront get-continuous-deployment-policy --id $CD_POLICY_ID \
  --query "ContinuousDeploymentPolicy.ContinuousDeploymentPolicyConfig.Enabled"
# true

# 2. Verificar session stickiness
# Primeira request — recebe cookie
curl -sI https://meusite.example.com/ | grep -i set-cookie
# Set-Cookie: AWS-CF-CD-<hash>=...

# 3. Monitorar metricas (em outro terminal)
watch -n 5 'aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name 5xxErrorRate \
  --dimensions Name=DistributionId,Value='$STAGING_DIST_ID' \
  --start-time $(date -u -d "5 minutes ago" +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average'

# 4. Testar que peso esta funcionando
# Fazer 100 requests e contar quantas vao para staging
for i in $(seq 1 100); do
  curl -s -o /dev/null -w "%{http_code}" https://meusite.example.com/
done | sort | uniq -c
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| **Staging Distribution** | Copia da producao onde testamos mudancas |
| **copy-distribution** | CLI para criar staging a partir de producao |
| **CD Policy** | Define como o trafego e dividido entre primary e staging |
| **Weight-based** | Porcentagem do trafego vai para staging (canary) |
| **Header-based** | Apenas requests com header especifico vao para staging (blue/green) |
| **Session Stickiness** | Cookie garante que usuario veja versao consistente |
| **Promote** | update-distribution-with-staging-config copia staging para producao |
| **Rollback** | Desabilitar CD policy retorna 100% para producao |

### Expert Tip

> **Estrategia de canary progressivo:** Comece com header-based para testar internamente (QA, engenharia). Depois mude para weight-based com 1-2%, monitore error rate e latencia por 15 min. Se OK, suba para 5%, 15%, 50%. So promova para 100% quando todas as metricas estiverem estaveis. Crie um CloudWatch Alarm que automaticamente desabilita a CD policy (via Lambda) se o error rate da staging distribution ultrapassar um threshold. Isso cria um rollback automatico.

---

## Desafio 48: CloudFront Static IPs

> **Level:** 300 | **Tempo:** 45 min | **Custo:** ~$0-1

### Objetivo

Entender e configurar CloudFront Static IPs para cenarios que exigem IPs fixos e previsiveis, como whitelisting em firewalls corporativos e compliance.

### Contexto

Por padrao, CloudFront usa IPs anycast dinamicos que mudam frequentemente. Isso e um problema quando:

- Clientes corporativos precisam whitelistar IPs em firewalls
- Regulamentacoes exigem IPs fixos para auditoria
- Integracoes com parceiros requerem IPs conhecidos

CloudFront Static IPs fornece um conjunto fixo de enderecos IP anycast que nao mudam, mesmo quando a infraestrutura subjacente e atualizada.

### Arquitetura

```
                         Usuarios
                            │
                            ▼
                ┌───────────────────────┐
                │   DNS Resolution      │
                │   meusite.com →       │
                │   Static IPs:         │
                │   198.51.100.1        │
                │   198.51.100.2        │
                │   203.0.113.1         │
                │   203.0.113.2         │
                │   (IPs fixos anycast) │
                └───────────┬───────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │   CloudFront Edge     │
                │   (mesmo edge com     │
                │    anycast routing)   │
                └───────────┬───────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │   Origin Server       │
                │   (Firewall do        │
                │    origin nao muda)   │
                └───────────────────────┘


  IPs Dinamicos (padrao):            Static IPs:
  ┌─────────────────────┐           ┌─────────────────────┐
  │ IPs mudam a cada    │           │ IPs fixos, mesmo    │
  │ deploy/manutencao   │           │ com mudancas na     │
  │ da infra AWS        │           │ infraestrutura      │
  │                     │           │                     │
  │ Ranges publicados   │           │ Conjunto pequeno    │
  │ em ip-ranges.json   │           │ e previsivel        │
  │ (milhares de IPs)   │           │                     │
  └─────────────────────┘           └─────────────────────┘
```

### Como Funciona

| Aspecto | Detalhe |
|---------|---------|
| **O que sao** | Conjunto fixo de IPs anycast atribuidos a sua distribution |
| **Quantos IPs** | Conjunto pequeno (varia, tipicamente 4-8 IPs por distribution) |
| **Anycast** | Mesmos IPs sao anunciados de todos os edge locations |
| **Custo** | Custo adicional por GB transferido (verificar pricing atual) |
| **Disponibilidade** | Sujeito a alocacao — pode haver fila de espera |
| **Habilitacao** | Via console, CLI ou Terraform |

### Passo a Passo

#### Passo 1: Verificar Elegibilidade

```bash
# Verificar se sua conta tem acesso ao feature
# Static IPs pode requerer enrollment previo
aws cloudfront get-distribution --id $DIST_ID \
  --query "Distribution.DistributionConfig"

# Verificar limites da conta
aws service-quotas list-service-quotas \
  --service-code cloudfront \
  --query "Quotas[?QuotaName=='Static IP addresses per distribution']"
```

#### Passo 2: Habilitar Static IPs

```bash
# Obter config atual
DIST_CONFIG=$(aws cloudfront get-distribution-config --id $DIST_ID)
ETAG=$(echo $DIST_CONFIG | jq -r '.ETag')

# Modificar para habilitar Static IPs
echo $DIST_CONFIG | jq '.DistributionConfig.AnycastIpListId = "ANYCAST_IP_LIST_ID"' \
  | jq '.DistributionConfig' > /tmp/dist-static-ip.json

# Criar Anycast IP List primeiro
aws cloudfront create-anycast-ip-list \
  --name "my-static-ips" \
  --ip-count 4

# Obter o ID da lista
ANYCAST_LIST_ID=$(aws cloudfront list-anycast-ip-lists \
  --query "AnycastIpLists.Items[?Name=='my-static-ips'].Id" \
  --output text)

# Associar a distribution
# Atualizar a distribution config com o AnycastIpListId
aws cloudfront update-distribution \
  --id $DIST_ID \
  --if-match $ETAG \
  --distribution-config file:///tmp/dist-static-ip.json

# Aguardar deploy
aws cloudfront wait distribution-deployed --id $DIST_ID
```

#### Passo 3: Obter os IPs Estaticos

```bash
# Listar os IPs atribuidos
aws cloudfront get-anycast-ip-list --id $ANYCAST_LIST_ID

# Verificar resolucao DNS
dig +short meusite.example.com
# Deve retornar os IPs estaticos

# Comparar com resolucao padrao (sem static IPs)
# IPs estaticos serao consistentes independente de onde voce resolver
dig +short meusite.example.com @8.8.8.8
dig +short meusite.example.com @1.1.1.1
# Mesmos IPs em todos os resolvers
```

#### Passo 4: Configurar Firewall do Cliente

```bash
# Exemplo: Security Group do origin que so aceita trafego dos Static IPs
STATIC_IPS=("198.51.100.1/32" "198.51.100.2/32" "203.0.113.1/32" "203.0.113.2/32")

for IP in "${STATIC_IPS[@]}"; do
  aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxxxx \
    --protocol tcp \
    --port 443 \
    --cidr $IP
done

echo "Firewall configurado com ${#STATIC_IPS[@]} IPs estaticos"
```

### Comparacao: CloudFront Static IPs vs Global Accelerator

| Aspecto | CloudFront Static IPs | Global Accelerator Static IPs |
|---------|----------------------|------------------------------|
| **Tipo** | Anycast IPs para CDN | Anycast IPs para acelerador de rede |
| **Proposito** | Servir conteudo via CDN com IPs fixos | Melhorar latencia de rede para qualquer endpoint |
| **IPs** | Conjunto dedicado por distribution | 2 IPs estaticos por accelerator |
| **Cache** | Sim (CDN completo) | Nao (pass-through de rede) |
| **Protocolos** | HTTP/HTTPS | TCP/UDP (qualquer protocolo) |
| **Edge Functions** | Sim (CF Functions, Lambda@Edge) | Nao |
| **Use Case** | CDN com IPs fixos para firewall | Acelerador de rede com IPs fixos |
| **Custo Base** | Por GB transferido (premium) | $0.025/hora por accelerator + DT |
| **Quando Usar** | Precisa de CDN + IPs fixos | Precisa de IPs fixos sem CDN |
| **Failover** | Via origin failover do CloudFront | Via endpoint groups e health checks |
| **WAF** | Sim (AWS WAF integrado) | Nao (apenas shield) |

### Terraform

```hcl
# ============================================================
# CloudFront com Static IPs
# ============================================================

resource "aws_cloudfront_anycast_ip_list" "static" {
  name     = "my-static-ips"
  ip_count = 4
}

resource "aws_cloudfront_distribution" "with_static_ips" {
  comment              = "Distribution com Static IPs"
  enabled              = true
  http_version         = "http2and3"
  anycast_ip_list_id   = aws_cloudfront_anycast_ip_list.static.id

  origin {
    domain_name              = aws_s3_bucket.content.bucket_regional_domain_name
    origin_id                = "S3"
    origin_access_control_id = aws_cloudfront_origin_access_control.main.id
  }

  default_cache_behavior {
    target_origin_id       = "S3"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6"
    compress               = true
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  tags = {
    Feature = "static-ips"
  }
}

output "static_ips" {
  description = "IPs estaticos atribuidos a distribution"
  value       = aws_cloudfront_anycast_ip_list.static.anycast_ips
}

output "distribution_domain" {
  value = aws_cloudfront_distribution.with_static_ips.domain_name
}
```

### Validacao

```bash
# 1. Verificar que Static IPs estao ativos
aws cloudfront get-anycast-ip-list --id $ANYCAST_LIST_ID \
  --query "AnycastIpList.AnycastIps"

# 2. Resolver DNS da distribution
dig +short $DIST_DOMAIN
# Deve retornar os IPs estaticos

# 3. Testar conectividade via IPs estaticos
curl --resolve "meusite.example.com:443:198.51.100.1" \
  https://meusite.example.com/

# 4. Verificar consistencia de diferentes locais
# Use ferramentas como dig de diferentes DNS resolvers
for DNS in 8.8.8.8 1.1.1.1 208.67.222.222; do
  echo "Resolver: $DNS"
  dig +short meusite.example.com @$DNS
  echo "---"
done

# 5. Verificar que os IPs nao mudam ao longo do tempo
# Execute periodicamente e compare
dig +short meusite.example.com > /tmp/ips-$(date +%Y%m%d).txt
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| **Static IPs** | IPs anycast fixos que nao mudam com atualizacoes de infra |
| **Anycast** | Mesmo IP anunciado de multiplos edge locations |
| **Anycast IP List** | Recurso que gerencia o conjunto de IPs estaticos |
| **Use case principal** | Whitelisting em firewalls corporativos |
| **Custo** | Premium sobre transfer out padrao |
| **vs Global Accelerator** | GA e para rede/protocolo, CF Static IPs e para CDN |
| **Limitacao** | Numero limitado de IPs, sujeito a alocacao |

### Expert Tip

> **Quando NAO usar Static IPs:** Se o requisito e apenas "saber quais IPs o CloudFront pode usar", voce nao precisa de Static IPs. A AWS publica os ranges de IP em `https://ip-ranges.amazonaws.com/ip-ranges.json` (filtrar por service=CLOUDFRONT). Static IPs sao para quando o cliente precisa de um conjunto PEQUENO e FIXO de IPs. Para a maioria dos casos, orientar o cliente a whitelistar os ranges publicados e suficiente e mais barato.

---

## Desafio 49: CloudFront + WebSocket

> **Level:** 300 | **Tempo:** 60 min | **Custo:** ~$1-2

### Objetivo

Configurar CloudFront para servir conexoes WebSocket, entendendo os requisitos de protocolo, timeouts, e implementando um cenario de chat em tempo real.

### Contexto

WebSocket e um protocolo full-duplex sobre TCP que permite comunicacao bidirecional entre cliente e servidor. O handshake comeca como HTTP e faz upgrade para WebSocket:

```
Cliente                         CloudFront                      Origin
  │                                │                               │
  │  GET / HTTP/1.1                │                               │
  │  Upgrade: websocket            │                               │
  │  Connection: Upgrade           │                               │
  │  Sec-WebSocket-Key: dGhl...    │  (forwarda o request)         │
  │  ─────────────────────────────►│  ────────────────────────────►│
  │                                │                               │
  │                                │  HTTP/1.1 101 Switching       │
  │                                │  Upgrade: websocket           │
  │  HTTP/1.1 101 Switching        │  Connection: Upgrade          │
  │  ◄─────────────────────────────│  ◄────────────────────────────│
  │                                │                               │
  │  ═══════ WebSocket Channel ════╪═══════════════════════════════│
  │  (full-duplex, binary/text)    │                               │
  │  ◄════════════════════════════►│  ◄═══════════════════════════►│
  │                                │                               │
```

### Requisitos do CloudFront para WebSocket

```
┌─────────────────────────────────────────────────────────────────┐
│              REQUISITOS para WebSocket no CloudFront             │
│                                                                 │
│  1. Viewer Protocol: HTTPS only (ou HTTP+HTTPS)                │
│     WebSocket usa wss:// (WebSocket Secure)                    │
│                                                                 │
│  2. Origin Protocol: HTTPS ou Match Viewer                     │
│     Origin deve suportar wss://                                │
│                                                                 │
│  3. Allowed Methods: GET, HEAD (minimo)                        │
│     WebSocket handshake e um GET                               │
│                                                                 │
│  4. Cache Policy: CachingDisabled                              │
│     WebSocket NAO pode ser cacheado                            │
│                                                                 │
│  5. Origin Request Policy: AllViewer                           │
│     TODOS os headers devem ser forwardados                     │
│     (Upgrade, Connection, Sec-WebSocket-*)                     │
│                                                                 │
│  6. HTTP Version: HTTP/1.1 para o upgrade                      │
│     (CloudFront faz downgrade automatico)                      │
│                                                                 │
│  7. TTL: 0 (sem cache)                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Timeouts WebSocket

| Parametro | Valor Padrao | Maximo | Descricao |
|-----------|-------------|--------|-----------|
| **Idle Timeout** | 10 segundos | 180 segundos | Tempo sem dados antes de fechar |
| **Connection Timeout** | 30 segundos | 60 segundos | Tempo para estabelecer conexao com origin |
| **Keep-Alive** | Depende do origin | N/A | Origin deve enviar pings/pongs |

**Importante:** O idle timeout de 10s e muito curto para a maioria das aplicacoes. Voce DEVE implementar keep-alive (ping/pong) no nivel da aplicacao para manter a conexao viva.

```
Timeline de uma conexao WebSocket via CloudFront:

0s      ── Handshake (GET + Upgrade) ──────────────────── OK (101)
1s      ── Mensagem do cliente ────────────────────────── Recebida
2s      ── Mensagem do servidor ───────────────────────── Recebida
...
10s     ── (sem atividade) ─── IDLE TIMEOUT! ─── Conexao fechada!

COM keep-alive (ping a cada 8s):
0s      ── Handshake ─────────────────────────────────── OK (101)
1s      ── Mensagem ──────────────────────────────────── OK
8s      ── Ping ──────────────────────────────────────── Pong (reset timer)
16s     ── Ping ──────────────────────────────────────── Pong (reset timer)
...     (conexao permanece aberta indefinidamente)
```

### Passo a Passo

#### Passo 1: Backend WebSocket (Node.js + ws)

```javascript
// server.js — Backend WebSocket simples
const { WebSocketServer } = require('ws');
const http = require('http');

const server = http.createServer((req, res) => {
  // Health check endpoint
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'healthy', connections: wss.clients.size }));
    return;
  }
  res.writeHead(404);
  res.end();
});

const wss = new WebSocketServer({ server, path: '/ws' });

wss.on('connection', (ws, req) => {
  const clientIp = req.headers['x-forwarded-for'] || req.socket.remoteAddress;
  console.log(`Nova conexao de ${clientIp}`);

  // Enviar mensagem de boas-vindas
  ws.send(JSON.stringify({
    type: 'welcome',
    message: 'Conectado ao chat!',
    timestamp: new Date().toISOString()
  }));

  // Keep-alive: ping a cada 8 segundos
  const pingInterval = setInterval(() => {
    if (ws.readyState === ws.OPEN) {
      ws.ping();
    }
  }, 8000);

  ws.on('pong', () => {
    // Cliente respondeu ao ping — conexao esta viva
  });

  ws.on('message', (data) => {
    const message = JSON.parse(data);
    console.log(`Mensagem de ${clientIp}: ${message.text}`);

    // Broadcast para todos os clientes conectados
    wss.clients.forEach((client) => {
      if (client.readyState === client.OPEN) {
        client.send(JSON.stringify({
          type: 'message',
          from: clientIp,
          text: message.text,
          timestamp: new Date().toISOString()
        }));
      }
    });
  });

  ws.on('close', () => {
    clearInterval(pingInterval);
    console.log(`Conexao fechada: ${clientIp}`);
  });

  ws.on('error', (err) => {
    clearInterval(pingInterval);
    console.error(`Erro: ${err.message}`);
  });
});

const PORT = process.env.PORT || 8080;
server.listen(PORT, () => {
  console.log(`WebSocket server rodando na porta ${PORT}`);
});
```

```bash
# Deploy no EC2/ECS/Fargate
# Exemplo rapido com EC2:
ssh ec2-user@<ip-da-instancia>
npm init -y && npm install ws
# Copiar server.js
node server.js &

# Verificar que esta rodando
curl http://localhost:8080/health
# {"status":"healthy","connections":0}
```

#### Passo 2: Configurar ALB para WebSocket

```bash
# ALB com target group para WebSocket
# O ALB suporta WebSocket nativamente — nao precisa de config especial
# Apenas certifique-se de que o stickiness esta habilitado
# para manter a conexao no mesmo target

aws elbv2 create-target-group \
  --name websocket-targets \
  --protocol HTTPS \
  --port 8080 \
  --vpc-id vpc-xxxxxxxx \
  --target-type instance \
  --health-check-path /health \
  --health-check-protocol HTTPS

# Habilitar stickiness (importante para WebSocket!)
aws elbv2 modify-target-group-attributes \
  --target-group-arn $TG_ARN \
  --attributes Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=86400

# Registrar instancias
aws elbv2 register-targets \
  --target-group-arn $TG_ARN \
  --targets Id=i-xxxxxxxx,Port=8080
```

#### Passo 3: Configurar CloudFront para WebSocket

```bash
# Cache policy: CachingDisabled (OBRIGATORIO para WebSocket)
CACHING_DISABLED="4135ea2d-6df8-44a3-9df3-4b5a84be39ad"

# Origin Request Policy: AllViewer (forwarda TODOS os headers)
ALL_VIEWER="216adef6-5c7f-47e4-b989-5492eafa07d3"

# Criar/atualizar distribution com behavior WebSocket
DIST_CONFIG=$(aws cloudfront get-distribution-config --id $DIST_ID)
ETAG=$(echo $DIST_CONFIG | jq -r '.ETag')

# Adicionar behavior para /ws*
# O behavior deve:
# 1. Usar CachingDisabled
# 2. Usar AllViewer origin request policy
# 3. Permitir GET/HEAD (WebSocket handshake = GET)
# 4. viewer-protocol-policy: https-only

cat > /tmp/ws-behavior.json << 'EOF'
{
  "PathPattern": "/ws*",
  "TargetOriginId": "ALB-WebSocket",
  "ViewerProtocolPolicy": "https-only",
  "AllowedMethods": {
    "Quantity": 3,
    "Items": ["GET", "HEAD", "OPTIONS"],
    "CachedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"]
    }
  },
  "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
  "OriginRequestPolicyId": "216adef6-5c7f-47e4-b989-5492eafa07d3",
  "Compress": false
}
EOF

echo "Behavior WebSocket configurado"

# Atualizar a distribution (adicionar o behavior acima na lista de CacheBehaviors)
# Use jq para inserir o behavior no array existente
echo $DIST_CONFIG | jq '.DistributionConfig' | \
  jq '.CacheBehaviors.Items += ['"$(cat /tmp/ws-behavior.json)"']' | \
  jq '.CacheBehaviors.Quantity += 1' > /tmp/dist-ws.json

aws cloudfront update-distribution \
  --id $DIST_ID \
  --if-match $ETAG \
  --distribution-config file:///tmp/dist-ws.json

aws cloudfront wait distribution-deployed --id $DIST_ID
echo "Distribution atualizada com WebSocket support!"
```

#### Passo 4: Ajustar Timeouts

```bash
# Aumentar idle timeout do origin (padrao 10s, max 180s)
# Isso e configurado no origin, nao no behavior

DIST_CONFIG=$(aws cloudfront get-distribution-config --id $DIST_ID)
ETAG=$(echo $DIST_CONFIG | jq -r '.ETag')

# Modificar o origin para ter read timeout maior
echo $DIST_CONFIG | jq '.DistributionConfig.Origins.Items[] |
  select(.Id == "ALB-WebSocket") |
  .CustomOriginConfig.OriginReadTimeout = 180' > /tmp/origin-timeout.json

# Nota: OriginReadTimeout max = 180s para custom origins
# Para manter conexao alem de 180s, DEVE usar keep-alive (ping/pong)
```

#### Passo 5: Testar com wscat

```bash
# Instalar wscat (ferramenta CLI para WebSocket)
npm install -g wscat

# Conectar via CloudFront
wscat -c wss://meusite.example.com/ws

# Enviar mensagem
> {"text": "Hello from CloudFront!"}

# Receber resposta
< {"type":"message","from":"203.0.113.50","text":"Hello from CloudFront!","timestamp":"2026-03-27T10:30:00Z"}

# Testar keep-alive (esperar mais de 10s)
# Se o ping/pong esta funcionando, a conexao permanece aberta
# Sem ping/pong, CloudFront fecha a conexao apos 10s idle

# Testar com multiplas conexoes simultaneas
for i in $(seq 1 5); do
  wscat -c wss://meusite.example.com/ws -x '{"text":"Client '$i' says hello"}' &
done
wait
```

### Terraform

```hcl
# ============================================================
# CloudFront + WebSocket
# ============================================================

# ALB Origin para WebSocket
resource "aws_lb" "websocket" {
  name               = "websocket-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnets
}

resource "aws_lb_target_group" "websocket" {
  name     = "websocket-tg"
  port     = 8080
  protocol = "HTTPS"
  vpc_id   = var.vpc_id

  health_check {
    path     = "/health"
    protocol = "HTTPS"
  }

  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
    enabled         = true
  }
}

resource "aws_lb_listener" "websocket" {
  load_balancer_arn = aws_lb.websocket.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.websocket.arn
  }
}

# CloudFront Distribution com WebSocket
resource "aws_cloudfront_distribution" "with_websocket" {
  comment             = "Distribution com WebSocket support"
  enabled             = true
  default_root_object = "index.html"
  http_version        = "http2and3"
  price_class         = "PriceClass_100"

  # Origin: S3 para conteudo estatico
  origin {
    domain_name              = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id                = "S3-Static"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  # Origin: ALB para WebSocket
  origin {
    domain_name = aws_lb.websocket.dns_name
    origin_id   = "ALB-WebSocket"

    custom_origin_config {
      http_port                = 80
      https_port               = 443
      origin_protocol_policy   = "https-only"
      origin_ssl_protocols     = ["TLSv1.2"]
      origin_read_timeout      = 180  # MAX para WebSocket idle
      origin_keepalive_timeout = 60
    }
  }

  # Behavior: WebSocket (/ws*)
  ordered_cache_behavior {
    path_pattern     = "/ws*"
    target_origin_id = "ALB-WebSocket"

    # OBRIGATORIO: HTTPS only para WebSocket (wss://)
    viewer_protocol_policy = "https-only"

    allowed_methods = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
    cached_methods  = ["GET", "HEAD"]

    # OBRIGATORIO: Sem cache para WebSocket
    cache_policy_id = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"  # CachingDisabled

    # OBRIGATORIO: Forwardar TODOS os headers (Upgrade, Connection, etc.)
    origin_request_policy_id = "216adef6-5c7f-47e4-b989-5492eafa07d3"  # AllViewer

    compress = false  # Nao comprimir WebSocket frames
  }

  # Default: Conteudo estatico
  default_cache_behavior {
    target_origin_id       = "S3-Static"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6"
    compress               = true
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  tags = {
    Feature = "websocket"
  }
}
```

### Validacao

```bash
# 1. Verificar que a distribution esta deployed
aws cloudfront get-distribution --id $DIST_ID \
  --query "Distribution.Status"
# "Deployed"

# 2. Testar WebSocket handshake
curl -sI \
  -H "Upgrade: websocket" \
  -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://meusite.example.com/ws
# Esperado: HTTP/1.1 101 Switching Protocols

# 3. Testar com wscat
wscat -c wss://meusite.example.com/ws
# Deve conectar e receber mensagem de boas-vindas

# 4. Testar keep-alive (manter conexao por > 30s)
# Conectar e esperar — se tiver ping/pong, conexao fica aberta

# 5. Testar latencia do WebSocket
wscat -c wss://meusite.example.com/ws --execute '{"text":"ping"}' \
  2>&1 | head -5

# 6. Verificar metricas
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name Requests \
  --dimensions Name=DistributionId,Value=$DIST_ID \
  --start-time $(date -u -d "1 hour ago" +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| **WebSocket no CloudFront** | Suportado nativamente, handshake via GET + Upgrade |
| **Cache Policy** | DEVE ser CachingDisabled — WebSocket nao e cacheavel |
| **Origin Request Policy** | DEVE ser AllViewer — headers Upgrade/Connection precisam ser forwardados |
| **Viewer Protocol** | HTTPS only (wss://) — ws:// nao e suportado via CloudFront |
| **Idle Timeout** | Padrao 10s, max 180s — DEVE implementar ping/pong |
| **Keep-Alive** | Ping/pong a cada 8-9s para manter conexao viva |
| **ALB Stickiness** | Importante para WebSocket — mesma conexao deve ir para o mesmo target |
| **HTTP Version** | CloudFront faz downgrade para HTTP/1.1 para o upgrade handshake |

### Expert Tip

> **Dimensionamento de WebSocket via CloudFront:** CloudFront nao limita o numero de conexoes WebSocket simultaneas, mas cada edge location tem limites de conexoes concorrentes para o origin. Se voce tem muitos clientes WebSocket, considere usar multiplos origins (origin group) ou um ALB com auto-scaling agressivo. O idle timeout de 180s e um HARD LIMIT — mesmo com keep-alive, CloudFront pode fechar conexoes por outras razoes (deploy, edge rotation). Sua aplicacao DEVE implementar reconexao automatica (exponential backoff).

---

## Desafio 50: HTTP/2, HTTP/3 (QUIC) e Protocolos

> **Level:** 300 | **Tempo:** 60 min | **Custo:** ~$0-1

### Objetivo

Entender a evolucao dos protocolos HTTP, configurar HTTP/2 e HTTP/3 (QUIC) no CloudFront, medir ganhos de performance, e entender gRPC over CloudFront.

### Contexto

A evolucao dos protocolos HTTP no CloudFront:

```
        HTTP/1.1 (1997)          HTTP/2 (2015)           HTTP/3 (2022)
  ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
  │                   │   │                   │   │                   │
  │  TCP              │   │  TCP              │   │  QUIC (UDP)       │
  │  Text-based       │   │  Binary framing   │   │  Binary framing   │
  │  1 request/conn   │   │  Multiplexing     │   │  Multiplexing     │
  │  No compression   │   │  HPACK compress.  │   │  QPACK compress.  │
  │  No prioritization│   │  Stream priority  │   │  No HOL blocking  │
  │  HOL blocking     │   │  HOL blocking(TCP)│   │  0-RTT connection │
  │  No server push   │   │  Server Push      │   │  Connection migr. │
  │                   │   │                   │   │                   │
  └───────────────────┘   └───────────────────┘   └───────────────────┘
         │                       │                       │
         │   ┌───────────────────┘                       │
         │   │   ┌───────────────────────────────────────┘
         ▼   ▼   ▼
  CloudFront suporta os tres! (http_version = "http2and3")
```

### HTTP/2: O Que Mudou

```
HTTP/1.1 — 6 conexoes TCP separadas:
  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │ TCP  │    │ TCP  │    │ TCP  │    │ TCP  │    │ TCP  │    │ TCP  │
  │ conn │    │ conn │    │ conn │    │ conn │    │ conn │    │ conn │
  │  1   │    │  2   │    │  3   │    │  4   │    │  5   │    │  6   │
  │      │    │      │    │      │    │      │    │      │    │      │
  │ GET  │    │ GET  │    │ GET  │    │ GET  │    │ GET  │    │ GET  │
  │ /css │    │ /js  │    │ /img1│    │ /img2│    │ /img3│    │ /font│
  └──────┘    └──────┘    └──────┘    └──────┘    └──────┘    └──────┘

HTTP/2 — 1 conexao TCP, multiplos streams:
  ┌────────────────────────────────────────────────────────────────┐
  │                    1 conexao TCP                               │
  │                                                                │
  │  Stream 1: GET /css  ════════════════════════►                │
  │  Stream 3: GET /js   ═══════════════════►                     │
  │  Stream 5: GET /img1 ═════════════════════════════►           │
  │  Stream 7: GET /img2 ════════════════►                        │
  │  Stream 9: GET /img3 ══════════════════════►                  │
  │  Stream 11: GET /font ═════════════════════════►              │
  │                                                                │
  │  Todos multiplexados na mesma conexao!                        │
  └────────────────────────────────────────────────────────────────┘
```

**Features do HTTP/2:**

| Feature | Descricao | Impacto |
|---------|-----------|---------|
| **Multiplexing** | Multiplos requests/responses na mesma conexao TCP | Elimina HOL blocking no nivel HTTP |
| **HPACK** | Compressao de headers | Reduz overhead (headers HTTP sao repetitivos) |
| **Binary Framing** | Protocolo binario (nao texto) | Parsing mais eficiente |
| **Stream Prioritization** | Priorizar recursos criticos (CSS antes de imagens) | Melhora percepao de velocidade |
| **Server Push** | Servidor envia recursos antes do cliente pedir | Elimina round-trips (DEPRECATED na pratica) |

### HTTP/3 (QUIC): A Revolucao

```
Problema do HTTP/2: HOL Blocking no TCP

  HTTP/2 sobre TCP:
  ┌────────────────────────────────────┐
  │  TCP Stream (ordenado)             │
  │                                    │
  │  [Stream1][Stream3][Stream5]       │
  │                    ▲               │
  │                    │               │
  │              Pacote perdido!       │
  │              TCP retransmite       │
  │              TUDO para apos        │
  │              esse ponto            │
  │              Stream3 e Stream5     │
  │              ficam bloqueados      │
  │              esperando Stream1     │
  └────────────────────────────────────┘

  HTTP/3 sobre QUIC (UDP):
  ┌────────────────────────────────────┐
  │  QUIC Streams (independentes)      │
  │                                    │
  │  Stream1: [pkt1][pkt2][pkt3]      │
  │  Stream3: [pkt1][pkt2]            │
  │  Stream5: [pkt1][pkt2][pkt3]      │
  │                    ▲               │
  │                    │               │
  │              Pacote perdido        │
  │              em Stream1?           │
  │              So Stream1 espera!    │
  │              Stream3 e Stream5     │
  │              continuam normal      │
  └────────────────────────────────────┘
```

**Features exclusivas do HTTP/3 (QUIC):**

| Feature | Descricao | Impacto |
|---------|-----------|---------|
| **Baseado em UDP** | Nao usa TCP, usa QUIC sobre UDP | Sem HOL blocking no transporte |
| **0-RTT Connection** | Conexoes subsequentes sem handshake | Primeiros bytes 30-50% mais rapido |
| **No HOL Blocking** | Streams independentes no transporte | Melhor em redes com packet loss |
| **Connection Migration** | Conexao sobrevive mudanca de IP/rede | Wi-Fi → 4G sem reconexao |
| **QPACK** | Evolucao do HPACK para QUIC | Compressao de headers sem HOL |
| **Built-in Encryption** | TLS 1.3 integrado no protocolo | Menos round-trips no handshake |

### Comparacao de Performance

```
Cenario: Pagina com 50 recursos, rede com 2% packet loss

  HTTP/1.1:  ████████████████████████████████████████  3.2s
  HTTP/2:    ██████████████████████████               2.1s  (-34%)
  HTTP/3:    ████████████████████                     1.6s  (-50%)

Cenario: Rede estavel (0% loss), baixa latencia

  HTTP/1.1:  ████████████████████████████████████████  2.0s
  HTTP/2:    ██████████████████████                    1.4s  (-30%)
  HTTP/3:    █████████████████████                     1.3s  (-35%)

Cenario: Rede movel (5% loss, variavel)

  HTTP/1.1:  ████████████████████████████████████████████████  5.8s
  HTTP/2:    ████████████████████████████████████████         4.2s  (-28%)
  HTTP/3:    ████████████████████████████                     2.9s  (-50%)
```

| Metrica | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| **TTFB (Rede estavel)** | ~150ms | ~120ms | ~80ms |
| **TTFB (0-RTT reconnect)** | N/A | ~120ms | ~40ms |
| **Conexoes TCP** | 6 por dominio | 1 por dominio | 0 (UDP) |
| **HOL Blocking** | Sim (HTTP + TCP) | Parcial (TCP) | Nao |
| **Handshake RTTs** | 2-3 (TCP+TLS) | 2-3 (TCP+TLS) | 1 (QUIC+TLS) |
| **Network Migration** | Reconexao | Reconexao | Transparente |
| **Packet Loss Impact** | Alto | Medio | Baixo |

### Passo a Passo

#### Passo 1: Verificar Configuracao Atual

```bash
# Verificar versao HTTP atual da distribution
aws cloudfront get-distribution --id $DIST_ID \
  --query "Distribution.DistributionConfig.HttpVersion"
# Possiveis valores: "http1.1", "http2", "http2and3"
```

#### Passo 2: Habilitar HTTP/2 e HTTP/3

```bash
# Obter config atual
DIST_CONFIG=$(aws cloudfront get-distribution-config --id $DIST_ID)
ETAG=$(echo $DIST_CONFIG | jq -r '.ETag')

# Modificar para http2and3
echo $DIST_CONFIG | jq '.DistributionConfig.HttpVersion = "http2and3"' \
  | jq '.DistributionConfig' > /tmp/dist-http3.json

# Aplicar
aws cloudfront update-distribution \
  --id $DIST_ID \
  --if-match $ETAG \
  --distribution-config file:///tmp/dist-http3.json

aws cloudfront wait distribution-deployed --id $DIST_ID
echo "HTTP/2 e HTTP/3 habilitados!"
```

#### Passo 3: Verificar HTTP/3 (QUIC) via Headers

```bash
# O CloudFront anuncia HTTP/3 via header alt-svc
curl -sI https://meusite.example.com/ | grep -i alt-svc
# alt-svc: h3=":443"; ma=86400

# Explicacao:
# h3=":443"   → HTTP/3 disponivel na porta 443
# ma=86400    → Maximo de 86400 segundos (24h) para o cliente lembrar

# Fluxo:
# 1. Primeiro acesso: cliente usa HTTP/2 (TCP)
# 2. Resposta inclui alt-svc header
# 3. Segundo acesso: cliente tenta HTTP/3 (QUIC/UDP)
# 4. Se QUIC falhar: fallback automatico para HTTP/2
```

#### Passo 4: Testar HTTP/2

```bash
# Testar com curl (HTTP/2)
curl -sI --http2 https://meusite.example.com/ | head -1
# HTTP/2 200

# Verificar multiplexing (multiplos requests na mesma conexao)
curl -w "HTTP Version: %{http_version}\nTime: %{time_total}\n" \
  --http2 -s -o /dev/null \
  https://meusite.example.com/

# Testar com nghttp (ferramenta especializada HTTP/2)
# Instalar: apt install nghttp2-client
nghttp -nv https://meusite.example.com/
# Mostra frames HTTP/2 detalhados: SETTINGS, HEADERS, DATA, etc.
```

#### Passo 5: Testar HTTP/3 (QUIC)

```bash
# Testar com curl compilado com suporte HTTP/3
# Nota: curl padrao pode nao ter suporte QUIC. Verificar:
curl --version | grep HTTP3

# Se tiver suporte:
curl -sI --http3 https://meusite.example.com/ | head -5
# HTTP/3 200
# ...

# Se nao tiver, instalar curl com quiche ou ngtcp2:
# (Exemplo com Docker)
docker run --rm ymuski/curl-http3 \
  curl -sI --http3 https://meusite.example.com/

# Alternativa: usar o Chrome DevTools
# 1. Abrir Chrome → F12 → Network tab
# 2. Acessar o site
# 3. Coluna "Protocol" deve mostrar "h3" apos o primeiro acesso
# 4. Primeiro acesso mostra "h2", segundo mostra "h3"

# Verificar via CloudWatch metrics
aws cloudwatch get-metric-data --cli-input-json '{
  "MetricDataQueries": [{
    "Id": "http3requests",
    "MetricStat": {
      "Metric": {
        "Namespace": "AWS/CloudFront",
        "MetricName": "Requests",
        "Dimensions": [
          {"Name": "DistributionId", "Value": "'$DIST_ID'"},
          {"Name": "Protocol", "Value": "HTTP/3"}
        ]
      },
      "Period": 3600,
      "Stat": "Sum"
    }
  }],
  "StartTime": "'$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S)'",
  "EndTime": "'$(date -u +%Y-%m-%dT%H:%M:%S)'"
}'
```

#### Passo 6: Auto Fallback

```
Como funciona o fallback HTTP/3 → HTTP/2:

  Cliente (Navegador)                    CloudFront Edge
        │                                      │
        │  1. GET / (HTTP/2 via TCP)           │
        │  ───────────────────────────────────►│
        │                                      │
        │  2. Response + alt-svc: h3=":443"    │
        │  ◄───────────────────────────────────│
        │                                      │
        │  3. Tenta QUIC (UDP:443)             │
        │  ═══════════════════════════════════►│
        │                                      │
        │  4a. Se QUIC funciona: usa HTTP/3    │
        │  ◄═══════════════════════════════════│
        │                                      │
        │  4b. Se QUIC bloqueado (firewall):   │
        │  X═════════════════════════════X     │
        │                                      │
        │  5. Fallback: continua HTTP/2 (TCP)  │
        │  ───────────────────────────────────►│
        │                                      │

  Nota: Alguns firewalls corporativos bloqueiam UDP:443
  O fallback e automatico e transparente para o usuario
```

```bash
# Verificar se QUIC/UDP esta bloqueado na sua rede
# Testar conectividade UDP na porta 443
nc -zuv meusite.example.com 443
# Se timeout: QUIC esta bloqueado, HTTP/2 sera usado como fallback

# Chrome: chrome://net-internals/#quic
# Mostra conexoes QUIC ativas e erros
```

### gRPC over CloudFront

CloudFront suporta gRPC quando HTTP/2 esta habilitado, pois gRPC usa HTTP/2 como transporte.

```
  gRPC Client                CloudFront              gRPC Server
       │                         │                        │
       │  HTTP/2 POST            │                        │
       │  Content-Type:          │                        │
       │   application/grpc      │                        │
       │  ──────────────────────►│  ─────────────────────►│
       │                         │                        │
       │  HTTP/2 200             │                        │
       │  Content-Type:          │                        │
       │   application/grpc      │                        │
       │  ◄──────────────────────│  ◄─────────────────────│
       │                         │                        │
```

```bash
# Configuracao para gRPC:
# 1. HTTP Version: http2 ou http2and3
# 2. Origin Protocol: HTTPS (gRPC requer TLS)
# 3. Cache: CachingDisabled (gRPC nao e cacheavel)
# 4. Allowed Methods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
# 5. Origin Request Policy: AllViewer (forwarda headers gRPC)

# Exemplo de behavior gRPC no cache behavior:
cat > /tmp/grpc-behavior.json << 'EOF'
{
  "PathPattern": "/grpc/*",
  "TargetOriginId": "ALB-gRPC",
  "ViewerProtocolPolicy": "https-only",
  "AllowedMethods": {
    "Quantity": 7,
    "Items": ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
  },
  "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
  "OriginRequestPolicyId": "216adef6-5c7f-47e4-b989-5492eafa07d3",
  "GrpcConfig": {
    "Enabled": true
  },
  "Compress": false
}
EOF

# Testar com grpcurl
grpcurl -plaintext meusite.example.com:443 \
  my.service.v1.MyService/GetResource
```

### Terraform

```hcl
# ============================================================
# CloudFront com HTTP/2, HTTP/3 e gRPC
# ============================================================

resource "aws_cloudfront_distribution" "modern_protocols" {
  comment             = "Distribution com HTTP/2, HTTP/3 e gRPC"
  enabled             = true
  default_root_object = "index.html"
  is_ipv6_enabled     = true
  price_class         = "PriceClass_100"

  # CHAVE: habilitar HTTP/2 e HTTP/3 (QUIC)
  http_version = "http2and3"

  # Origin: S3 para conteudo estatico
  origin {
    domain_name              = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id                = "S3-Static"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  # Origin: ALB para gRPC
  origin {
    domain_name = aws_lb.grpc.dns_name
    origin_id   = "ALB-gRPC"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Behavior: gRPC
  ordered_cache_behavior {
    path_pattern     = "/grpc/*"
    target_origin_id = "ALB-gRPC"

    viewer_protocol_policy = "https-only"
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]

    cache_policy_id          = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"  # CachingDisabled
    origin_request_policy_id = "216adef6-5c7f-47e4-b989-5492eafa07d3"  # AllViewer

    grpc_config {
      enabled = true
    }

    compress = false
  }

  # Default: Conteudo estatico (beneficia-se de HTTP/2 multiplexing)
  default_cache_behavior {
    target_origin_id       = "S3-Static"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6"
    compress               = true
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  tags = {
    Protocols = "http2-http3-grpc"
  }
}

output "distribution_domain" {
  value = aws_cloudfront_distribution.modern_protocols.domain_name
}

output "http_version" {
  value = aws_cloudfront_distribution.modern_protocols.http_version
}
```

### Validacao

```bash
# 1. Verificar HTTP version da distribution
aws cloudfront get-distribution --id $DIST_ID \
  --query "Distribution.DistributionConfig.HttpVersion"
# "http2and3"

# 2. Testar HTTP/2
curl -sI --http2 https://meusite.example.com/ | head -1
# HTTP/2 200

# 3. Verificar alt-svc header (anuncia HTTP/3)
curl -sI https://meusite.example.com/ | grep -i alt-svc
# alt-svc: h3=":443"; ma=86400

# 4. Testar HTTP/3 (se curl suporta)
curl -sI --http3 https://meusite.example.com/ | head -1
# HTTP/3 200

# 5. Comparar performance HTTP/1.1 vs HTTP/2 vs HTTP/3
echo "=== HTTP/1.1 ==="
curl -w "TTFB: %{time_starttransfer}s Total: %{time_total}s\n" \
  --http1.1 -s -o /dev/null https://meusite.example.com/

echo "=== HTTP/2 ==="
curl -w "TTFB: %{time_starttransfer}s Total: %{time_total}s\n" \
  --http2 -s -o /dev/null https://meusite.example.com/

echo "=== HTTP/3 ==="
curl -w "TTFB: %{time_starttransfer}s Total: %{time_total}s\n" \
  --http3 -s -o /dev/null https://meusite.example.com/

# 6. Testar gRPC
grpcurl -d '{"id": "123"}' \
  meusite.example.com:443 \
  my.service.v1.MyService/GetResource

# 7. Verificar no Chrome DevTools
# Abrir site → F12 → Network → coluna "Protocol"
# h2 = HTTP/2, h3 = HTTP/3
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| **HTTP/2** | Multiplexing, HPACK, binary framing — padrao desde 2015 |
| **HTTP/3 (QUIC)** | Baseado em UDP, 0-RTT, sem HOL blocking — melhor em redes instaveis |
| **http2and3** | Habilita ambos — CloudFront negocia o melhor com cada cliente |
| **alt-svc** | Header que anuncia disponibilidade de HTTP/3 |
| **Auto Fallback** | Se QUIC bloqueado, fallback transparente para HTTP/2 |
| **0-RTT** | Conexoes subsequentes HTTP/3 sem handshake completo |
| **Connection Migration** | HTTP/3 sobrevive mudanca Wi-Fi → 4G |
| **gRPC** | Suportado via HTTP/2 com grpc_config enabled |

### Expert Tip

> **Quando HTTP/3 faz diferenca real:** HTTP/3 brilha em redes moveis com packet loss alto (3-5%). Em redes estables (datacenter, fibra), a diferenca entre HTTP/2 e HTTP/3 e marginal. Para sites com audiencia global e muitos usuarios moveis, habilitar HTTP/3 e uma vitoria facil sem downside — o fallback e automatico. Para gRPC, prefira HTTP/2 por enquanto: gRPC sobre HTTP/3 ainda e experimental na maioria dos frameworks. Monitore o header `alt-svc` nos seus testes para confirmar que HTTP/3 esta sendo anunciado corretamente.

---

## Resumo do Modulo 08

```
┌─────────────────────────────────────────────────────────────────┐
│                    MODULO 08 — COMPLETO                          │
│                                                                 │
│  D.45  Distribution Tenants     ✓  Multi-tenant, 5000 tenants  │
│  D.46  Arquitetura SaaS         ✓  Tenant resolution, cache    │
│  D.47  Continuous Deployment    ✓  Blue/green, canary, promote  │
│  D.48  Static IPs               ✓  Anycast fixo, firewalls     │
│  D.49  WebSocket                ✓  wss://, keep-alive, chat    │
│  D.50  HTTP/2, HTTP/3, gRPC    ✓  QUIC, 0-RTT, multiplexing   │
│                                                                 │
│  Habilidades adquiridas:                                        │
│  • Projetar arquiteturas SaaS multi-tenant com CloudFront      │
│  • Implementar deploys seguros com canary e blue/green         │
│  • Configurar protocolos modernos para performance otima       │
│  • Integrar WebSocket e gRPC via CloudFront                    │
│  • Gerenciar Static IPs para compliance                        │
└─────────────────────────────────────────────────────────────────┘
```

### Tabela de Referencia Rapida

| Desafio | Feature | Quando Usar |
|---------|---------|-------------|
| **D.45** | Distribution Tenants | SaaS com muitos clientes, mesmo behavior, dominios diferentes |
| **D.46** | SaaS Architecture | Plataforma completa com tenant isolation e routing |
| **D.47** | Continuous Deployment | Deploy seguro com rollback rapido |
| **D.48** | Static IPs | Firewall whitelisting, compliance |
| **D.49** | WebSocket | Real-time: chat, notifications, live updates |
| **D.50** | HTTP/2 + HTTP/3 | Performance, especialmente em redes moveis |

---

> **Proximo:** [Modulo 09 — Observabilidade, Troubleshooting e FinOps](modulo-09-observabilidade-finops.md) — Logging avancado, Real-time Logs, metricas custom, troubleshooting de 5xx/4xx, e otimizacao de custos.
