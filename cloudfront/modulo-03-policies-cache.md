# Módulo 03 — Policies e Cache

> **Nível:** Intermediário
> **Objetivo do Módulo:** Dominar os três tipos de policies do CloudFront (Cache, Origin Request, Response Headers), entender profundamente como o cache funciona, e otimizar a cache hit ratio para máxima performance.

---

## Conceitos-Chave Antes de Começar

### Os Três Tipos de Policy

```
Viewer (Usuário) ──→ CloudFront Edge ──→ Origin (Servidor)
                          │
              ┌───────────┼───────────┐
              │           │           │
        Cache Policy  Origin Req   Response Headers
        (como cachear) Policy      Policy
                      (o que       (o que adicionar
                       enviar       na resposta)
                       à origin)
```

| Policy | O Que Faz | Direção |
|--------|-----------|---------|
| **Cache Policy** | Define a cache key e TTLs | Edge ↔ Cache |
| **Origin Request Policy** | Define o que enviar para a origin | Edge → Origin |
| **Response Headers Policy** | Define headers adicionados na resposta | Edge → Viewer |

### Cache Key — O Conceito Mais Importante

A **Cache Key** é o identificador único de um objeto no cache. Dois requests com a mesma cache key retornam o mesmo objeto cacheado.

```
Cache Key = URL + Headers incluídos + Cookies incluídos + Query Strings incluídas
```

**Quanto mais elementos na cache key, MENOR a cache hit ratio** (mais variações = mais cache misses).

Exemplo:
```
# Cache key simples (alta hit ratio)
/images/logo.png

# Cache key com query string (hit ratio menor)
/images/logo.png?size=large
/images/logo.png?size=small   ← objetos DIFERENTES no cache

# Cache key com header (hit ratio ainda menor)
/images/logo.png + Accept-Language: pt-BR
/images/logo.png + Accept-Language: en-US   ← objetos DIFERENTES no cache
```

---

## Desafio 11: Managed Cache Policies — Entendendo Cada Uma

### Objetivo
Conhecer profundamente cada Managed Cache Policy da AWS, quando usar cada uma, e como escolher a certa para cada cenário.

### Contexto
A AWS oferece políticas pré-configuradas que cobrem 90% dos casos de uso. Saber qual usar evita criar policies customizadas desnecessárias.

### Managed Cache Policies — Referência Completa

#### 1. CachingOptimized (`658327ea-f89d-4fab-a63d-7e88639e58f6`)

| Campo | Valor |
|-------|-------|
| **Min TTL** | 1 segundo |
| **Max TTL** | 31.536.000 (1 ano) |
| **Default TTL** | 86.400 (24 horas) |
| **Headers** | Nenhum na cache key |
| **Cookies** | Nenhum na cache key |
| **Query Strings** | Nenhuma na cache key |
| **Gzip** | Habilitado |
| **Brotli** | Habilitado |
| **Quando usar** | Conteúdo estático (S3, assets, imagens) |

```
Comportamento do TTL:
1. Se a origin envia Cache-Control: max-age=X → usa X (respeitando min/max)
2. Se a origin NÃO envia Cache-Control → usa Default TTL (24h)
3. Se a origin envia Cache-Control: no-cache → cacheia por Min TTL (1s)
4. Se a origin envia Cache-Control: no-store → NÃO cacheia
```

#### 2. CachingOptimizedForUncompressedObjects (`b2884449-e4de-46a7-ac36-70bc7f1ddd6d`)

| Campo | Valor |
|-------|-------|
| **Igual ao CachingOptimized, mas:** | |
| **Gzip** | Desabilitado |
| **Brotli** | Desabilitado |
| **Quando usar** | Vídeos, imagens comprimidas, binários |

#### 3. CachingDisabled (`4135ea2d-6df8-44a3-9df3-4b5a84be39ad`)

| Campo | Valor |
|-------|-------|
| **Min TTL** | 0 |
| **Max TTL** | 0 |
| **Default TTL** | 0 |
| **Headers** | Nenhum |
| **Cookies** | Nenhum |
| **Query Strings** | Nenhuma |
| **Quando usar** | APIs dinâmicas, conteúdo personalizado |

> **Importante:** CachingDisabled NÃO envia headers/cookies/query strings para a origin. Para isso, você PRECISA de uma **Origin Request Policy** junto.

#### 4. Elemental-MediaPackage (`08627262-05a9-4f76-9ded-b50ca2e3a84f`)

| Campo | Valor |
|-------|-------|
| **Min TTL** | 0 |
| **Max TTL** | 31.536.000 |
| **Default TTL** | 86.400 |
| **Headers** | `Origin` |
| **Query Strings** | Todas |
| **Quando usar** | Streaming com AWS Elemental MediaPackage |

#### 5. Amplify (`2e54312d-136d-493c-8eb9-b001f22f67d2`)

Para aplicações AWS Amplify. Configuração otimizada para SPAs.

### Passo a Passo — Comparando na Prática

```bash
# 1. Listar todas as managed cache policies
aws cloudfront list-cache-policies --type managed \
    --query "CachePolicyList.Items[*].CachePolicy.{Name:CachePolicyConfig.Name,Id:Id,DefaultTTL:CachePolicyConfig.DefaultTTL,MaxTTL:CachePolicyConfig.MaxTTL}" \
    --output table

# 2. Ver detalhes de uma policy específica
aws cloudfront get-cache-policy \
    --id "658327ea-f89d-4fab-a63d-7e88639e58f6" \
    | jq '.CachePolicy.CachePolicyConfig'

# 3. Comparar headers/cookies/query strings de cada policy
for POLICY_ID in \
    "658327ea-f89d-4fab-a63d-7e88639e58f6" \
    "4135ea2d-6df8-44a3-9df3-4b5a84be39ad" \
    "b2884449-e4de-46a7-ac36-70bc7f1ddd6d"; do
    echo "=== Policy: $POLICY_ID ==="
    aws cloudfront get-cache-policy --id $POLICY_ID \
        | jq '{
            Name: .CachePolicy.CachePolicyConfig.Name,
            DefaultTTL: .CachePolicy.CachePolicyConfig.DefaultTTL,
            Headers: .CachePolicy.CachePolicyConfig.ParametersInCacheKeyAndForwardedToOrigin.HeadersConfig,
            Cookies: .CachePolicy.CachePolicyConfig.ParametersInCacheKeyAndForwardedToOrigin.CookiesConfig,
            QueryStrings: .CachePolicy.CachePolicyConfig.ParametersInCacheKeyAndForwardedToOrigin.QueryStringsConfig
        }'
    echo ""
done
```

### Como Testar

```bash
# Testar CachingOptimized (conteúdo estático)
curl -I "https://$DOMAIN/index.html"
# Primeira: x-cache: Miss from cloudfront
# Segunda: x-cache: Hit from cloudfront, age: X

# Testar CachingDisabled (API)
curl -I "https://$DOMAIN/api/health"
# SEMPRE: x-cache: Miss from cloudfront (nunca cacheia)

# Testar com query strings — CachingOptimized IGNORA query strings
curl -I "https://$DOMAIN/index.html?v=1"
curl -I "https://$DOMAIN/index.html?v=2"
# AMBAS retornam o MESMO objeto! (query string não está na cache key)
```

### O Que Aprendemos

| Conceito | Descrição |
|----------|-----------|
| **Managed Policies** | Policies pré-configuradas pela AWS |
| **CachingOptimized** | O "padrão" para conteúdo estático |
| **CachingDisabled** | Zero cache, mas NÃO envia headers à origin sozinha |
| **TTL Hierarchy** | Origin Cache-Control > Default TTL > Min TTL |
| **Compressão** | Gzip/Brotli são features da cache policy |

### Árvore de Decisão

```
Preciso cachear?
├── NÃO → CachingDisabled + Origin Request Policy (AllViewer ou AllViewerExceptHostHeader)
│
└── SIM
    ├── É conteúdo estático (JS, CSS, HTML, imagens)?
    │   └── CachingOptimized
    │
    ├── É vídeo/binário já comprimido?
    │   └── CachingOptimizedForUncompressedObjects
    │
    ├── É streaming (MediaPackage)?
    │   └── Elemental-MediaPackage
    │
    └── Preciso de cache key customizada?
        └── Criar Custom Cache Policy (Desafio 12)
```

### Dica Expert
> **Erro clássico:** Usar `CachingDisabled` para API e esquecer de adicionar uma **Origin Request Policy**. Sem ela, headers como `Authorization`, cookies e query strings NÃO são enviados à origin, e sua API recebe requests "vazias". Sempre combine `CachingDisabled` com `AllViewer` ou `AllViewerExceptHostHeader`.

---

## Desafio 12: Custom Cache Policies — Controle Total do Cache

### Objetivo
Criar cache policies customizadas para cenários que as managed policies não cobrem, otimizando a cache key para máxima hit ratio.

### Contexto
Uma API que retorna dados diferentes por idioma precisa de `Accept-Language` na cache key. Uma loja que personaliza por região precisa de `CloudFront-Viewer-Country`. Managed policies não cobrem esses cenários.

### Cenário 1: E-commerce com Personalização por País

```
Mesmo produto, preço diferente por país:
  /produto/123 + Brasil     → R$ 99,90
  /produto/123 + EUA        → $19.99
  /produto/123 + Alemanha   → €17,99

Cache key: URL + CloudFront-Viewer-Country
```

#### Criar a Policy

```bash
aws cloudfront create-cache-policy \
    --cache-policy-config '{
        "Name": "Ecommerce-ByCountry",
        "Comment": "Cache por URL + país do viewer",
        "DefaultTTL": 3600,
        "MaxTTL": 86400,
        "MinTTL": 0,
        "ParametersInCacheKeyAndForwardedToOrigin": {
            "EnableAcceptEncodingGzip": true,
            "EnableAcceptEncodingBrotli": true,
            "HeadersConfig": {
                "HeaderBehavior": "whitelist",
                "Headers": {
                    "Quantity": 1,
                    "Items": ["CloudFront-Viewer-Country"]
                }
            },
            "CookiesConfig": {
                "CookieBehavior": "whitelist",
                "Cookies": {
                    "Quantity": 1,
                    "Items": ["currency"]
                }
            },
            "QueryStringsConfig": {
                "QueryStringBehavior": "whitelist",
                "QueryStrings": {
                    "Quantity": 2,
                    "Items": ["category", "sort"]
                }
            }
        }
    }'
```

#### Terraform

```hcl
resource "aws_cloudfront_cache_policy" "ecommerce_by_country" {
  name        = "Ecommerce-ByCountry"
  comment     = "Cache por URL + país do viewer"
  default_ttl = 3600   # 1 hora
  max_ttl     = 86400  # 1 dia
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    enable_accept_encoding_gzip   = true
    enable_accept_encoding_brotli = true

    headers_config {
      header_behavior = "whitelist"
      headers {
        items = [
          "CloudFront-Viewer-Country",
          # Outros headers CloudFront disponíveis:
          # "CloudFront-Viewer-Country-Region"
          # "CloudFront-Viewer-City"
          # "CloudFront-Viewer-Latitude"
          # "CloudFront-Viewer-Longitude"
          # "CloudFront-Is-Mobile-Viewer"
          # "CloudFront-Is-Desktop-Viewer"
          # "CloudFront-Is-Tablet-Viewer"
          # "CloudFront-Is-SmartTV-Viewer"
          # "CloudFront-Viewer-TLS"
        ]
      }
    }

    cookies_config {
      cookie_behavior = "whitelist"
      cookies {
        items = ["currency"]
      }
    }

    query_strings_config {
      query_string_behavior = "whitelist"
      query_strings {
        items = ["category", "sort"]
      }
    }
  }
}
```

### Cenário 2: API com Autenticação e Paginação

```
Mesmo endpoint, resultados diferentes por:
  - Página (query string)
  - Token de autenticação (header)

/api/users?page=1&limit=20  (user A) → dados do user A
/api/users?page=1&limit=20  (user B) → dados do user B
```

```hcl
resource "aws_cloudfront_cache_policy" "api_authenticated" {
  name        = "API-Authenticated"
  comment     = "Cache por URL + auth + paginação"
  default_ttl = 60     # 1 minuto
  max_ttl     = 300    # 5 minutos
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    enable_accept_encoding_gzip   = true
    enable_accept_encoding_brotli = true

    headers_config {
      header_behavior = "whitelist"
      headers {
        items = ["Authorization"]
        # ATENÇÃO: Authorization na cache key = cada user tem seu próprio cache
        # Isso REDUZ drasticamente a hit ratio mas garante isolamento
      }
    }

    cookies_config {
      cookie_behavior = "none"
    }

    query_strings_config {
      query_string_behavior = "whitelist"
      query_strings {
        items = ["page", "limit", "sort", "filter"]
      }
    }
  }
}
```

### Cenário 3: Site Multi-idioma

```hcl
resource "aws_cloudfront_cache_policy" "multi_language" {
  name        = "Multi-Language"
  comment     = "Cache por idioma"
  default_ttl = 3600
  max_ttl     = 86400
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    enable_accept_encoding_gzip   = true
    enable_accept_encoding_brotli = true

    headers_config {
      header_behavior = "whitelist"
      headers {
        items = ["Accept-Language"]
        # CUIDADO: Accept-Language pode ter milhares de variações
        # "pt-BR", "pt-BR,pt;q=0.9", "pt-BR,pt;q=0.9,en-US;q=0.8"
        # Melhor usar CloudFront Function para normalizar antes
      }
    }

    cookies_config {
      cookie_behavior = "whitelist"
      cookies {
        items = ["lang"]  # Cookie normalizado é melhor que Accept-Language
      }
    }

    query_strings_config {
      query_string_behavior = "none"
    }
  }
}
```

### Headers CloudFront Disponíveis para Cache Key

| Header | Valor Exemplo | Uso |
|--------|---------------|-----|
| `CloudFront-Viewer-Country` | `BR`, `US`, `DE` | Geo-personalização |
| `CloudFront-Viewer-Country-Region` | `SP`, `CA`, `BY` | Estado/região |
| `CloudFront-Viewer-City` | `Sao Paulo`, `New York` | Cidade |
| `CloudFront-Is-Mobile-Viewer` | `true`/`false` | Layout responsivo |
| `CloudFront-Is-Desktop-Viewer` | `true`/`false` | Layout responsivo |
| `CloudFront-Is-Tablet-Viewer` | `true`/`false` | Layout responsivo |
| `CloudFront-Is-SmartTV-Viewer` | `true`/`false` | Smart TV |
| `CloudFront-Viewer-TLS` | `TLSv1.3:...` | Versão TLS |
| `CloudFront-Viewer-Address` | `1.2.3.4:12345` | IP do viewer |
| `CloudFront-Viewer-ASN` | `16509` | ASN do viewer |

### Como Testar

```bash
# 1. Testar cache por país (usar header do CloudFront)
curl -H "CloudFront-Viewer-Country: BR" -I "https://$DOMAIN/produto/123"
curl -H "CloudFront-Viewer-Country: US" -I "https://$DOMAIN/produto/123"
# São cache entries DIFERENTES (mesmo URL, países diferentes)

# 2. Testar cache por query string
curl -I "https://$DOMAIN/api/users?page=1&limit=20"
curl -I "https://$DOMAIN/api/users?page=2&limit=20"
# Diferentes (page diferente)

curl -I "https://$DOMAIN/api/users?page=1&limit=20&debug=true"
# debug não está no whitelist, então é IGUAL ao page=1&limit=20
# (query strings fora do whitelist são ignoradas)

# 3. Verificar cache key efetiva
# O header x-amz-cf-id pode ajudar a debugar (identificador único da request)

# 4. Listar custom policies
aws cloudfront list-cache-policies --type custom \
    --query "CachePolicyList.Items[*].CachePolicy.{Name:CachePolicyConfig.Name,Id:Id}" \
    --output table
```

### O Que Aprendemos

| Conceito | Descrição |
|----------|-----------|
| **Cache Key** | URL + headers + cookies + query strings que formam o identificador único |
| **Whitelist** | Incluir apenas itens específicos na cache key |
| **Hit Ratio** | % de requests servidas do cache (mais items = menor ratio) |
| **Headers CloudFront** | Headers geo/device adicionados automaticamente pelo CloudFront |
| **Normalização** | Padronizar valores antes da cache key (ex: Accept-Language) |
| **Trade-off** | Personalização vs Performance (mais personalização = menos cache) |

### Dica Expert
> **A regra de ouro:** inclua o MÍNIMO necessário na cache key. Cada elemento adicionado multiplica as variações do cache. Se você tem 200 países e 3 tipos de device, são 600 variações para CADA URL. Use **CloudFront Functions** para normalizar valores (ex: mapear Accept-Language para 5 idiomas suportados) antes de incluir na cache key.

---

## Desafio 13: Origin Request Policies — Controlando o que a Origin Recebe

### Objetivo
Dominar Origin Request Policies para controlar exatamente quais headers, cookies e query strings são enviados para a origin.

### Contexto
A Cache Policy controla a cache key. Mas e se você precisa enviar um header para a origin que NÃO deve fazer parte da cache key? É para isso que serve a Origin Request Policy. É a separação entre "o que identifica o cache" e "o que a origin precisa para processar".

### Diferença Crítica: Cache Policy vs Origin Request Policy

```
Viewer envia:
  Authorization: Bearer token123
  Accept-Language: pt-BR
  Cookie: session=abc; tracking=xyz
  ?page=1&limit=20&utm_source=google

Cache Policy (cache key):
  Headers na key: Accept-Language
  Cookies na key: session
  Query strings na key: page, limit

Origin Request Policy (enviado à origin):
  Headers: Authorization, Accept-Language, X-Forwarded-For
  Cookies: session
  Query strings: page, limit

Resultado:
  Cache key: /api/users + pt-BR + session=abc + page=1&limit=20
  Origin recebe: tudo acima + Authorization + X-Forwarded-For

  Authorization NÃO está na cache key, mas É enviado à origin
  utm_source NÃO está na cache key e NÃO é enviado à origin
  tracking cookie NÃO está na cache key e NÃO é enviado à origin
```

### Managed Origin Request Policies

#### 1. AllViewer (`216adef6-5c7f-47e4-b989-5492eafa07d3`)

| Campo | Valor |
|-------|-------|
| **Headers** | Todos os headers do viewer |
| **Cookies** | Todos |
| **Query Strings** | Todas |
| **Quando usar** | APIs que precisam de tudo (auth, cookies, query strings) |

#### 2. AllViewerExceptHostHeader (`b689b0a8-53d0-40ab-baf2-68738e2966ac`)

| Campo | Valor |
|-------|-------|
| **Headers** | Todos EXCETO Host |
| **Cookies** | Todos |
| **Query Strings** | Todas |
| **Quando usar** | Origins que não entendem o Host header do CloudFront |

#### 3. AllViewerAndCloudFrontHeaders-2022-06 (`33f36d7e-f396-46d9-90e0-52428a34d9dc`)

| Campo | Valor |
|-------|-------|
| **Headers** | Todos do viewer + TODOS os CloudFront-* headers |
| **Cookies** | Todos |
| **Query Strings** | Todas |
| **Quando usar** | Origin precisa de geolocalização, device type, etc |

#### 4. CORS-S3Origin (`88a5eaf4-2fd4-4709-b370-b4c650ea3fcf`)

| Campo | Valor |
|-------|-------|
| **Headers** | Origin, Access-Control-Request-Headers, Access-Control-Request-Method |
| **Cookies** | Nenhum |
| **Query Strings** | Nenhuma |
| **Quando usar** | S3 com CORS habilitado |

#### 5. UserAgentRefererHeaders (`acba4595-bd28-49b8-b9fe-13317c0390fa`)

| Campo | Valor |
|-------|-------|
| **Headers** | User-Agent, Referer |
| **Quando usar** | Origins que fazem analytics ou bot detection |

### Criar Custom Origin Request Policy

```hcl
# Cenário: API que precisa de auth + geo, mas geo NÃO deve afetar cache
resource "aws_cloudfront_origin_request_policy" "api_with_geo" {
  name    = "API-WithGeo"
  comment = "Envia auth e geo para a origin"

  headers_config {
    header_behavior = "whitelist"
    headers {
      items = [
        "Authorization",
        "Content-Type",
        "Accept",
        "CloudFront-Viewer-Country",
        "CloudFront-Viewer-Country-Region",
        "CloudFront-Viewer-City",
        "CloudFront-Is-Mobile-Viewer",
        "CloudFront-Viewer-Address",
        "X-Forwarded-For",
        "Referer",
        "Origin"  # Necessário para CORS
      ]
    }
  }

  cookies_config {
    cookie_behavior = "whitelist"
    cookies {
      items = ["session", "csrf_token"]
    }
  }

  query_strings_config {
    query_string_behavior = "all"  # Todas as query strings
  }
}
```

```bash
# Via CLI
aws cloudfront create-origin-request-policy \
    --origin-request-policy-config '{
        "Name": "API-WithGeo",
        "Comment": "Envia auth e geo para a origin",
        "HeadersConfig": {
            "HeaderBehavior": "allViewerAndWhitelistCloudFront",
            "Headers": {
                "Quantity": 4,
                "Items": [
                    "CloudFront-Viewer-Country",
                    "CloudFront-Viewer-Country-Region",
                    "CloudFront-Is-Mobile-Viewer",
                    "CloudFront-Viewer-City"
                ]
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

### Combinação Correta: Cache Policy + Origin Request Policy

| Cenário | Cache Policy | Origin Request Policy |
|---------|-------------|----------------------|
| **S3 estático** | CachingOptimized | Nenhuma (não precisa) |
| **S3 com CORS** | CachingOptimized | CORS-S3Origin |
| **API sem cache** | CachingDisabled | AllViewer |
| **API com cache por país** | Custom (com Country na key) | AllViewerAndCloudFrontHeaders |
| **API auth + cache por página** | Custom (page na key) | Custom (Authorization + geo) |
| **Proxy reverso** | CachingDisabled | AllViewerExceptHostHeader |

### Como Testar

```bash
# 1. Listar managed origin request policies
aws cloudfront list-origin-request-policies --type managed \
    --query "OriginRequestPolicyList.Items[*].OriginRequestPolicy.{Name:OriginRequestPolicyConfig.Name,Id:Id}" \
    --output table

# 2. Ver detalhes
aws cloudfront get-origin-request-policy \
    --id "216adef6-5c7f-47e4-b989-5492eafa07d3" \
    | jq '.OriginRequestPolicy.OriginRequestPolicyConfig'

# 3. Testar na prática — verificar o que a origin recebe
# No seu backend, logue todos os headers recebidos
# Exemplo em Node.js:
# app.get('/debug', (req, res) => {
#     res.json({
#         headers: req.headers,
#         query: req.query,
#         cookies: req.cookies
#     });
# });

curl "https://$DOMAIN/api/debug?test=1" \
    -H "Authorization: Bearer token123" \
    -H "Accept-Language: pt-BR" \
    -b "session=abc123; tracking=xyz"

# 4. Verificar quais headers chegaram na origin (via endpoint de debug)
```

### O Que Aprendemos

| Conceito | Descrição |
|----------|-----------|
| **Origin Request Policy** | Controla o que é enviado para a origin |
| **Separação de concerns** | Cache key ≠ dados enviados à origin |
| **allViewerAndWhitelistCloudFront** | Todos os headers do viewer + CloudFront headers selecionados |
| **CORS** | Precisa enviar `Origin` header para origins com CORS |
| **Host header** | Geralmente NÃO enviar para origins customizadas (usar AllViewerExceptHostHeader) |

### Dica Expert
> **Nunca coloque `Authorization` na cache key** a menos que realmente precise de cache por usuário. Em vez disso, use `CachingDisabled` + `AllViewer`. Se precisa cachear API autenticada, faça a autenticação em Lambda@Edge e cache a resposta sem o token. Isso mantém alta hit ratio sem comprometer segurança.

---

## Desafio 14: Response Headers Policies — Segurança e CORS

### Objetivo
Configurar Response Headers Policies para adicionar headers de segurança (CSP, HSTS, X-Frame-Options), CORS, e headers customizados em todas as respostas.

### Contexto
Headers de segurança são obrigatórios em produção. Em vez de configurá-los em cada origin (S3, ALB, EC2), configure uma vez no CloudFront e eles são adicionados em TODAS as respostas — centralizado e consistente.

### Managed Response Headers Policies

#### 1. SecurityHeadersPolicy (`67f7725c-6f97-4210-82d7-5512b31e9d03`)

Headers incluídos:
```
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=63072000
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: (não incluso — muito específico por aplicação)
```

#### 2. CORS-with-preflight (`5cc3b908-e619-4b99-88e5-2cf7f45965bd`)

Headers incluídos:
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
Access-Control-Allow-Headers: *
Access-Control-Max-Age: 86400
```

#### 3. CORS-With-Preflight-And-SecurityHeadersPolicy (`eaab4381-ed33-4a86-88ca-d9558dc6cd63`)

Combinação dos dois acima.

### Criar Custom Response Headers Policy

```hcl
resource "aws_cloudfront_response_headers_policy" "custom_security" {
  name    = "Custom-Security-And-CORS"
  comment = "Headers de segurança + CORS customizado"

  # ============================================================
  # SECURITY HEADERS
  # ============================================================
  security_headers_config {

    # HSTS - Force HTTPS por 2 anos, incluindo subdomínios
    strict_transport_security {
      access_control_max_age_sec = 63072000  # 2 anos
      include_subdomains         = true
      preload                    = true
      override                   = true
    }

    # Prevent clickjacking
    frame_options {
      frame_option = "DENY"  # ou "SAMEORIGIN"
      override     = true
    }

    # Prevent MIME type sniffing
    content_type_options {
      override = true
      # Automaticamente adiciona: X-Content-Type-Options: nosniff
    }

    # XSS Protection (legacy, mas ainda útil)
    xss_protection {
      mode_block = true
      protection = true
      override   = true
    }

    # Referrer Policy
    referrer_policy {
      referrer_policy = "strict-origin-when-cross-origin"
      override        = true
    }

    # Content Security Policy
    content_security_policy {
      content_security_policy = "default-src 'self'; script-src 'self' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' https://fonts.gstatic.com; connect-src 'self' https://api.meusite.com; frame-ancestors 'none'"
      override = true
    }
  }

  # ============================================================
  # CORS HEADERS
  # ============================================================
  cors_config {
    # Origins permitidas
    access_control_allow_origins {
      items = [
        "https://meusite.com",
        "https://www.meusite.com",
        "https://app.meusite.com"
      ]
    }

    # Methods permitidos
    access_control_allow_methods {
      items = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "DELETE"]
    }

    # Headers permitidos
    access_control_allow_headers {
      items = ["Authorization", "Content-Type", "X-Requested-With", "Accept"]
    }

    # Headers expostos ao JavaScript
    access_control_expose_headers {
      items = ["X-Request-Id", "X-RateLimit-Remaining"]
    }

    # Tempo de cache do preflight
    access_control_max_age_sec = 86400  # 24 horas

    # Permitir credentials (cookies)
    access_control_allow_credentials = true

    # Aplicar mesmo quando origin não envia Origin header
    origin_override = true
  }

  # ============================================================
  # CUSTOM HEADERS
  # ============================================================
  custom_headers_config {
    items {
      header   = "X-Powered-By"
      value    = "CloudFront"
      override = true
    }

    items {
      header   = "X-Environment"
      value    = "production"
      override = false  # NÃO sobrescrever se a origin já enviou
    }

    items {
      header   = "Permissions-Policy"
      value    = "camera=(), microphone=(), geolocation=(self)"
      override = true
    }

    # Remove server header (boa prática de segurança)
    items {
      header   = "Server"
      value    = ""
      override = true
    }
  }

  # ============================================================
  # REMOVE HEADERS (remover headers da origin)
  # ============================================================
  remove_headers_config {
    items {
      header = "X-Powered-By"  # Remover header padrão de frameworks
    }
    items {
      header = "X-AspNet-Version"
    }
  }
}
```

```bash
# Via CLI
aws cloudfront create-response-headers-policy \
    --response-headers-policy-config '{
        "Name": "Custom-Security",
        "Comment": "Headers de segurança customizados",
        "SecurityHeadersConfig": {
            "StrictTransportSecurity": {
                "Override": true,
                "AccessControlMaxAgeSec": 63072000,
                "IncludeSubdomains": true,
                "Preload": true
            },
            "FrameOptions": {
                "Override": true,
                "FrameOption": "DENY"
            },
            "ContentTypeOptions": {
                "Override": true
            },
            "XSSProtection": {
                "Override": true,
                "Protection": true,
                "ModeBlock": true
            },
            "ReferrerPolicy": {
                "Override": true,
                "ReferrerPolicy": "strict-origin-when-cross-origin"
            }
        }
    }'
```

### Como Testar

```bash
# 1. Verificar headers de segurança
curl -I "https://$DOMAIN/"
# Deve mostrar:
# strict-transport-security: max-age=63072000; includeSubDomains; preload
# x-frame-options: DENY
# x-content-type-options: nosniff
# x-xss-protection: 1; mode=block
# referrer-policy: strict-origin-when-cross-origin
# content-security-policy: default-src 'self'; ...

# 2. Testar CORS preflight
curl -X OPTIONS "https://$DOMAIN/api/users" \
    -H "Origin: https://meusite.com" \
    -H "Access-Control-Request-Method: POST" \
    -H "Access-Control-Request-Headers: Authorization" \
    -I
# Deve mostrar:
# access-control-allow-origin: https://meusite.com
# access-control-allow-methods: GET, HEAD, OPTIONS, PUT, POST, DELETE
# access-control-allow-headers: Authorization, Content-Type, ...
# access-control-max-age: 86400

# 3. Testar CORS com origin não permitida
curl -X OPTIONS "https://$DOMAIN/api/users" \
    -H "Origin: https://hacker.com" \
    -I
# NÃO deve incluir access-control-allow-origin

# 4. Usar securityheaders.com para score
echo "Teste seu domínio em: https://securityheaders.com/?q=https://$DOMAIN"

# 5. Verificar CSP com browser DevTools
# Abra o site no browser → F12 → Console
# Erros de CSP aparecerão aqui
```

### Headers de Segurança — Referência

| Header | Protege Contra | Valor Recomendado |
|--------|---------------|-------------------|
| `Strict-Transport-Security` | Downgrade attacks, SSL stripping | `max-age=63072000; includeSubDomains; preload` |
| `X-Frame-Options` | Clickjacking | `DENY` ou `SAMEORIGIN` |
| `X-Content-Type-Options` | MIME type sniffing | `nosniff` |
| `Content-Security-Policy` | XSS, injection | Específico por aplicação |
| `Referrer-Policy` | Information leakage via Referer | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Acesso a features do browser | `camera=(), microphone=()` |
| `X-XSS-Protection` | XSS (legacy browsers) | `1; mode=block` |

### O Que Aprendemos

| Conceito | Descrição |
|----------|-----------|
| **Response Headers Policy** | Headers adicionados em TODAS as respostas |
| **override** | Se true, sobrescreve header da origin; se false, mantém o da origin |
| **Security headers** | Proteção contra ataques web comuns |
| **CORS** | Cross-Origin Resource Sharing — controle de acesso cross-domain |
| **Preflight** | Request OPTIONS que o browser faz antes de requests cross-origin |
| **CSP** | Content Security Policy — controla de onde recursos podem ser carregados |
| **HSTS preload** | Registrar no HSTS preload list dos browsers |
| **Remove headers** | Esconder informações sensíveis (Server, X-Powered-By) |

### Dica Expert
> **Content-Security-Policy é o header mais difícil de configurar** porque é específico por aplicação. Comece com `Content-Security-Policy-Report-Only` para monitorar sem bloquear. Use o endpoint de report (`report-uri`) para coletar violações. Só mude para enforcement (`Content-Security-Policy`) quando tiver confiança de que não vai quebrar nada.

---

## Desafio 15: Otimizando Cache Hit Ratio — O Caminho para 95%+

### Objetivo
Diagnosticar e otimizar a cache hit ratio da sua distribuição CloudFront, aplicando técnicas avançadas para alcançar 95%+ de hit ratio.

### Contexto
Cache hit ratio é A métrica mais importante do CloudFront. Quanto maior, menos requests vão para a origin (menor custo, menor latência, maior resiliência). Uma hit ratio de 60% significa que 40% das requests vão para a origin — tem muito espaço para otimizar.

### Diagnóstico: Medindo a Hit Ratio Atual

```bash
# 1. Via CloudWatch (últimas 24h)
aws cloudwatch get-metric-statistics \
    --namespace AWS/CloudFront \
    --metric-name CacheHitRate \
    --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
    --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 3600 \
    --statistics Average \
    --output table

# 2. Via logs (análise manual)
# Se Standard Logging estiver habilitado, analise os logs S3:
# Campo sc-status + x-edge-result-type
# Hit, Miss, RefreshHit, LimitExceeded, Error

# 3. Via Real-time Logs (mais detalhado)
# Envia para Kinesis Data Streams em tempo real
```

### Checklist de Otimização — 15 Técnicas

#### Técnica 1: Normalizar Query Strings

**Problema:** Mesma página com query strings em ordem diferente = cache entries diferentes.

```
/products?color=red&size=M    ← Cache entry 1
/products?size=M&color=red    ← Cache entry 2 (DIFERENTE!)
```

**Solução:** CloudFront Function para ordenar query strings.

```javascript
// CloudFront Function para normalizar query strings
function handler(event) {
    var request = event.request;
    var qs = request.querystring;

    // Ordenar query strings alfabeticamente
    var sortedQs = Object.keys(qs)
        .sort()
        .map(key => `${key}=${qs[key].value}`)
        .join('&');

    // Remover query strings de tracking (não afetam conteúdo)
    var cleanQs = {};
    for (var key in qs) {
        if (!key.startsWith('utm_') && key !== 'fbclid' && key !== 'gclid') {
            cleanQs[key] = qs[key];
        }
    }

    request.querystring = cleanQs;
    return request;
}
```

#### Técnica 2: Normalizar Accept-Language

**Problema:** Milhares de variações do header Accept-Language.

```
Accept-Language: pt-BR,pt;q=0.9,en-US;q=0.8,en;q=0.7
Accept-Language: pt-BR
Accept-Language: pt
→ Todos significam "português" mas são cache entries DIFERENTES
```

**Solução:**

```javascript
function handler(event) {
    var request = event.request;
    var acceptLanguage = request.headers['accept-language']
        ? request.headers['accept-language'].value
        : '';

    var supportedLanguages = ['pt', 'en', 'es', 'fr', 'de'];
    var defaultLang = 'en';
    var lang = defaultLang;

    for (var i = 0; i < supportedLanguages.length; i++) {
        if (acceptLanguage.toLowerCase().includes(supportedLanguages[i])) {
            lang = supportedLanguages[i];
            break;
        }
    }

    // Substituir o header por versão normalizada
    request.headers['accept-language'] = { value: lang };
    return request;
}
```

#### Técnica 3: Remover Headers Desnecessários da Cache Key

```
# RUIM: incluir User-Agent na cache key
User-Agent tem milhares de variações → hit ratio ~0%

# BOM: usar CloudFront device headers
CloudFront-Is-Mobile-Viewer: true/false → apenas 2 variações
```

#### Técnica 4: Habilitar Origin Shield

```hcl
origin {
  domain_name = aws_s3_bucket.media.bucket_regional_domain_name
  origin_id   = "S3-Media"

  origin_shield {
    enabled              = true
    origin_shield_region = "us-east-1"  # Região mais próxima da origin
  }
}
```

```
Sem Origin Shield:
  Edge SP → Origin
  Edge RJ → Origin
  Edge POA → Origin
  = 3 requests à origin para o mesmo objeto

Com Origin Shield:
  Edge SP → Origin Shield (us-east-1) → Origin
  Edge RJ → Origin Shield (us-east-1) → Hit! (já buscou)
  Edge POA → Origin Shield (us-east-1) → Hit! (já buscou)
  = 1 request à origin, 2 hits no Origin Shield
```

#### Técnica 5: Otimizar TTLs

```
Tipo de Conteúdo      TTL Recomendado     Motivo
─────────────────────────────────────────────────────
index.html            5-60 min            Muda frequentemente
*.js (com hash)       1 ano               Imutável
*.css (com hash)      1 ano               Imutável
Imagens               30 dias             Raramente muda
API /config           5 min               Muda pouco
API dinâmica          0 (sem cache)       Sempre diferente
Fontes                1 ano               Nunca muda
Vídeos                30 dias             Raramente muda
```

#### Técnica 6: Habilitar Compressão (Gzip + Brotli)

```hcl
# Na cache policy
parameters_in_cache_key_and_forwarded_to_origin {
  enable_accept_encoding_gzip   = true
  enable_accept_encoding_brotli = true
}
```

> CloudFront normaliza automaticamente `Accept-Encoding` para `gzip` ou `br`, evitando variações na cache key.

#### Técnica 7: Usar Cache Busting em vez de Invalidations

```javascript
// webpack.config.js
output: {
    filename: '[name].[contenthash:8].js',
    // Gera: app.a1b2c3d4.js
}
```

#### Técnica 8: Minimizar Cookies na Cache Key

```
# RUIM
Cookie: session=abc; tracking=xyz; preferences=dark; cart=123; _ga=GA1.2...

# BOM: whitelist apenas o necessário
Cookie: session=abc
```

#### Técnica 9: Wildcard CNAME vs Specific CNAME

```
# Cada CNAME diferente pode ter cache separado
cdn1.site.com/logo.png  → Cache entry 1
cdn2.site.com/logo.png  → Cache entry 2

# Use um único CNAME para maximizar hit ratio
cdn.site.com/logo.png   → Uma única cache entry
```

#### Técnica 10: Usar HTTP/2 Server Push (se aplicável)

```
# ALB pode enviar Link header para prefetch
Link: </css/style.css>; rel=preload; as=style
```

#### Técnica 11: Configurar Error Caching

```hcl
custom_error_response {
  error_code            = 404
  error_caching_min_ttl = 300  # Cache 404 por 5 min
  response_code         = 404
  response_page_path    = "/error.html"
}
```

> Sem isso, cada 404 vai até a origin. Com sites que têm muitos 404 (bots, scanners), isso pode ser significativo.

#### Técnica 12: Monitorar e Iterar

```bash
# Dashboard CloudWatch para cache performance
# Métricas essenciais:
# - CacheHitRate (target: >90%)
# - OriginLatency (quanto menor, melhor)
# - Requests (volume total)
# - BytesDownloaded vs BytesUploaded
# - 4xxErrorRate e 5xxErrorRate
```

### Tabela de Impacto

| Técnica | Impacto na Hit Ratio | Complexidade |
|---------|---------------------|--------------|
| Remover headers desnecessários da key | ⬆⬆⬆ Alto | Baixa |
| Normalizar query strings | ⬆⬆ Médio | Média |
| Normalizar Accept-Language | ⬆⬆ Médio | Média |
| Origin Shield | ⬆⬆ Médio | Baixa |
| TTLs otimizados | ⬆⬆⬆ Alto | Baixa |
| Cache busting (hash no nome) | ⬆⬆⬆ Alto | Média |
| Minimizar cookies na key | ⬆⬆ Médio | Baixa |
| Error caching | ⬆ Baixo | Baixa |
| Habilitar compressão | ⬆ Baixo (melhora tamanho) | Baixa |

### Como Testar

```bash
# 1. Verificar hit ratio antes das otimizações
# (anotar o baseline)

# 2. Aplicar técnicas uma por uma

# 3. Aguardar 24h para dados significativos

# 4. Comparar hit ratio via CloudWatch
aws cloudwatch get-metric-statistics \
    --namespace AWS/CloudFront \
    --metric-name CacheHitRate \
    --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
    --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 86400 \
    --statistics Average \
    --output table

# 5. Analisar resultados por behavior
# CloudFront > Reports & Analytics > Cache Statistics
# Filtrar por behavior para ver qual tem pior hit ratio
```

### O Que Aprendemos

| Conceito | Descrição |
|----------|-----------|
| **Cache Hit Ratio** | % de requests servidas do cache (alvo: >90%) |
| **Cache Key** | Quanto menor, melhor a hit ratio |
| **Normalização** | Padronizar valores para reduzir variações |
| **Origin Shield** | Cache centralizado que reduz requests à origin |
| **TTL** | Quanto maior (quando possível), melhor a hit ratio |
| **Cache Busting** | Evita invalidations usando hash no nome do arquivo |
| **Error Caching** | Cachear erros para proteger a origin |

### Dica Expert
> **A melhor cache hit ratio que vi em produção foi 98.7%** em um site de e-commerce grande. A receita: assets estáticos com hash (TTL 1 ano), HTML com TTL de 5 min, API sem cache, Origin Shield habilitado, e CloudFront Function para normalizar query strings e remover tracking params. O segredo é medir constantemente e otimizar iterativamente. Cada 1% de hit ratio a mais = menos custo e melhor experiência do usuário.

---

## Resumo do Módulo 03

Ao completar este módulo, você domina:

- [x] Todas as Managed Cache Policies e quando usar cada uma
- [x] Criar Custom Cache Policies com cache key otimizada
- [x] Origin Request Policies — separar cache key dos dados enviados à origin
- [x] Response Headers Policies — segurança (HSTS, CSP, etc) e CORS
- [x] 12+ técnicas para otimizar cache hit ratio para 95%+

### Mapa Mental de Policies

```
Request do Viewer
       │
       ▼
┌─────────────────┐
│ Cache Policy     │ → Define a CACHE KEY (URL + headers + cookies + qs)
│                  │ → Define TTLs (min, max, default)
│                  │ → Define compressão (Gzip/Brotli)
└────────┬────────┘
         │
    Cache HIT? ──→ SIM ──→ Retorna do cache + Response Headers Policy
         │
        NÃO
         │
         ▼
┌─────────────────┐
│ Origin Request   │ → Define o que ENVIAR para a origin
│ Policy           │ → Headers, cookies, query strings
│                  │ → Pode enviar mais do que está na cache key
└────────┬────────┘
         │
         ▼
      Origin
         │
         ▼
┌─────────────────┐
│ Response Headers │ → Adiciona/remove headers na RESPOSTA
│ Policy           │ → Security headers (HSTS, CSP, etc)
│                  │ → CORS headers
│                  │ → Custom headers
└─────────────────┘
         │
         ▼
    Viewer (Usuário)
```

**Próximo:** [Módulo 04 — CloudFront Functions e Lambda@Edge →](modulo-04-functions-lambda-edge.md)
