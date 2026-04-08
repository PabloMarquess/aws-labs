# Módulo 05 — Segurança Avançada

> **Nível:** 300-400 (Advanced/Expert)
> **Tempo Total Estimado:** 8-12 horas de labs
> **Custo Estimado:** ~$5-15 (maioria no free tier, WAF tem custo)
> **Objetivo do Módulo:** Dominar todas as camadas de segurança do CloudFront — desde acesso restrito com signed URLs/cookies até WAF, Shield, mTLS e field-level encryption.

---

## Mapa de Segurança do CloudFront

```
                              CAMADAS DE SEGURANÇA
                              ═══════════════════

Layer 7 (Aplicação)
├── WAF (Web Application Firewall) ───────── Desafio 25
│   ├── Managed Rules (SQLi, XSS, Bad Bots)
│   ├── Rate Limiting
│   ├── IP Sets (whitelist/blacklist)
│   ├── Geo Blocking
│   └── Custom Rules (regex, headers)
│
├── Signed URLs / Signed Cookies ─────────── Desafios 23, 24
│   ├── Canned Policy (simples)
│   └── Custom Policy (IP, date range)
│
├── Field-Level Encryption ───────────────── Desafio 28
│   └── Criptografia de campos específicos no edge
│
├── mTLS (Mutual TLS) ───────────────────── Desafio 30
│   └── Certificado do cliente obrigatório
│
└── Response Headers (CSP, HSTS, etc) ──── (Módulo 03)

Layer 3/4 (Rede/Transporte)
├── AWS Shield Standard (automático) ─────── Desafio 26
├── AWS Shield Advanced (premium) ────────── Desafio 26
├── TLS 1.2/1.3 ──────────────────────────── (Módulo 01)
└── Geo Restriction ──────────────────────── Desafio 27

Acesso à Origin
├── OAC (Origin Access Control) ──────────── Desafio 29
├── OAI (Origin Access Identity) legacy ──── Desafio 29
├── VPC Origins (PrivateLink) ────────────── (Módulo 02)
├── Custom Headers secretos ──────────────── (Módulo 02)
└── Trust Stores ─────────────────────────── Desafio 30
```

---

## Desafio 23: Signed URLs — Conteúdo Restrito com URLs Temporárias

> **Level:** 300 | **Tempo:** 45 min | **Custo:** Free tier

### Objetivo
Criar URLs assinadas que expiram após um período, restringindo acesso a conteúdo premium (vídeos, downloads, documentos).

### Contexto Real
Plataformas de cursos online (Udemy, Coursera), sites de streaming, e portais de documentos confidenciais usam signed URLs para:
- Limitar acesso temporário (link expira em 1h)
- Restringir por IP (só o comprador pode assistir)
- Impedir compartilhamento de links (link único por sessão)

### Arquitetura

```
┌──────────┐    1. Login/Pagamento     ┌──────────┐
│  Viewer  │ ────────────────────────→ │ Backend  │
│ (Browser)│                           │ (API)    │
└────┬─────┘                           └────┬─────┘
     │                                      │
     │  2. Gera Signed URL                  │
     │     com private key                  │
     │  ←───────────────────────────────────┘
     │
     │  3. Acessa: https://cdn.site.com/video.mp4
     │     ?Expires=1234567890
     │     &Signature=abc...
     │     &Key-Pair-Id=K1ABC
     │
     ▼
┌──────────────┐   4. Valida assinatura    ┌──────────┐
│  CloudFront  │ ──────────────────────→   │    S3    │
│  (Edge)      │   5. Se válida, busca     │ (Origin) │
│              │ ←─────────────────────    │          │
└──────────────┘   6. Retorna conteúdo     └──────────┘
```

### Pré-requisitos
- Distribution CloudFront configurada (Módulo 01)
- OpenSSL instalado

### Passo a Passo

#### 1. Gerar RSA Key Pair

```bash
# Gerar private key (2048 bits)
openssl genrsa -out private_key.pem 2048

# Extrair public key
openssl rsa -pubout -in private_key.pem -out public_key.pem

# Verificar as chaves
openssl rsa -in private_key.pem -check
echo "---"
cat public_key.pem
```

#### 2. Criar Public Key no CloudFront

```bash
# Upload da public key para CloudFront
PUBLIC_KEY_ID=$(aws cloudfront create-public-key \
    --public-key-config '{
        "CallerReference": "lab-signed-urls-'$(date +%s)'",
        "Name": "SignedURL-Key-Lab",
        "Comment": "Chave para signed URLs - Lab Desafio 23",
        "EncodedKey": "'"$(cat public_key.pem)"'"
    }' \
    --query "PublicKey.Id" --output text)

echo "Public Key ID: $PUBLIC_KEY_ID"
```

#### 3. Criar Key Group

```bash
# Key Group agrupa public keys que podem assinar URLs
KEY_GROUP_ID=$(aws cloudfront create-key-group \
    --key-group-config '{
        "Name": "SignedURL-KeyGroup-Lab",
        "Comment": "Key Group para lab de signed URLs",
        "Items": ["'$PUBLIC_KEY_ID'"]
    }' \
    --query "KeyGroup.Id" --output text)

echo "Key Group ID: $KEY_GROUP_ID"
```

#### 4. Configurar Behavior para Exigir Signed URLs

```bash
# Obter config atual da distribution
ETAG=$(aws cloudfront get-distribution-config --id $DIST_ID --query "ETag" --output text)
aws cloudfront get-distribution-config --id $DIST_ID --query "DistributionConfig" > dist-config.json

# Editar dist-config.json:
# No DefaultCacheBehavior ou em um ordered behavior específico, adicionar:
# "TrustedKeyGroups": {
#     "Enabled": true,
#     "Quantity": 1,
#     "Items": ["KEY_GROUP_ID"]
# }

# Exemplo com jq para behavior /premium/*:
cat dist-config.json | jq '
    .CacheBehaviors.Items += [{
        "PathPattern": "/premium/*",
        "TargetOriginId": "S3-meu-site",
        "ViewerProtocolPolicy": "https-only",
        "AllowedMethods": {
            "Quantity": 2,
            "Items": ["GET", "HEAD"]
        },
        "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
        "Compress": true,
        "TrustedKeyGroups": {
            "Enabled": true,
            "Quantity": 1,
            "Items": ["'$KEY_GROUP_ID'"]
        }
    }] | .CacheBehaviors.Quantity += 1
' > dist-config-updated.json

aws cloudfront update-distribution \
    --id $DIST_ID \
    --if-match $ETAG \
    --distribution-config file://dist-config-updated.json
```

#### 5. Gerar Signed URL — Canned Policy (Node.js)

```javascript
// generate-signed-url.js
const crypto = require('crypto');
const fs = require('fs');

// Configuração
const DISTRIBUTION_DOMAIN = 'd1234abcdef.cloudfront.net';
const PRIVATE_KEY_PATH = './private_key.pem';
const KEY_PAIR_ID = 'K1ABCDEFGHIJKL'; // Public Key ID do CloudFront

function generateSignedUrl(resourceUrl, expiresInSeconds) {
    const privateKey = fs.readFileSync(PRIVATE_KEY_PATH, 'utf8');

    // Calcular timestamp de expiração
    const expires = Math.floor(Date.now() / 1000) + expiresInSeconds;

    // Canned Policy: URL + Expires (simples)
    const policy = JSON.stringify({
        Statement: [{
            Resource: resourceUrl,
            Condition: {
                DateLessThan: {
                    'AWS:EpochTime': expires
                }
            }
        }]
    });

    // Assinar com RSA-SHA1
    const signer = crypto.createSign('RSA-SHA1');
    signer.update(policy);
    const signature = signer.sign(privateKey, 'base64')
        .replace(/\+/g, '-')   // URL-safe base64
        .replace(/\//g, '~')
        .replace(/=/g, '_');

    // Montar URL assinada (Canned Policy usa Expires no query string)
    return `${resourceUrl}?Expires=${expires}&Signature=${signature}&Key-Pair-Id=${KEY_PAIR_ID}`;
}

// Gerar URL que expira em 1 hora
const url = generateSignedUrl(
    `https://${DISTRIBUTION_DOMAIN}/premium/video-aula-01.mp4`,
    3600  // 1 hora
);

console.log('Signed URL (válida por 1 hora):');
console.log(url);
```

#### 6. Gerar Signed URL — Custom Policy (Node.js)

```javascript
// generate-signed-url-custom.js
const crypto = require('crypto');
const fs = require('fs');

const PRIVATE_KEY_PATH = './private_key.pem';
const KEY_PAIR_ID = 'K1ABCDEFGHIJKL';

function generateCustomPolicySignedUrl(options) {
    const {
        resourceUrl,    // URL ou padrão com wildcard (ex: https://cdn.site.com/premium/*)
        expiresIn,      // Segundos até expirar
        startsIn = 0,   // Segundos até ficar válida (default: imediato)
        allowedIp = null // IP/CIDR permitido (ex: "203.0.113.0/24")
    } = options;

    const privateKey = fs.readFileSync(PRIVATE_KEY_PATH, 'utf8');
    const expires = Math.floor(Date.now() / 1000) + expiresIn;
    const starts = Math.floor(Date.now() / 1000) + startsIn;

    // Custom Policy: mais flexível que Canned
    const condition = {
        DateLessThan: { 'AWS:EpochTime': expires }
    };

    // Opcional: data de início
    if (startsIn > 0) {
        condition.DateGreaterThan = { 'AWS:EpochTime': starts };
    }

    // Opcional: restrição de IP
    if (allowedIp) {
        condition.IpAddress = { 'AWS:SourceIp': allowedIp };
    }

    const policy = JSON.stringify({
        Statement: [{
            Resource: resourceUrl,
            Condition: condition
        }]
    });

    // Assinar
    const signer = crypto.createSign('RSA-SHA1');
    signer.update(policy);
    const signature = signer.sign(privateKey, 'base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '~')
        .replace(/=/g, '_');

    // Custom Policy é incluída na URL (base64)
    const policyBase64 = Buffer.from(policy)
        .toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '~')
        .replace(/=/g, '_');

    // Custom policy usa Policy no query string (não Expires)
    const separator = resourceUrl.includes('?') ? '&' : '?';
    return `${resourceUrl}${separator}Policy=${policyBase64}&Signature=${signature}&Key-Pair-Id=${KEY_PAIR_ID}`;
}

// Exemplo 1: URL com restrição de IP + expiração
const url1 = generateCustomPolicySignedUrl({
    resourceUrl: 'https://cdn.site.com/premium/video-aula-01.mp4',
    expiresIn: 3600,                    // 1 hora
    allowedIp: '203.0.113.100/32'       // Apenas este IP
});
console.log('URL com IP restriction:', url1);

// Exemplo 2: Wildcard — acesso a TODA a pasta /premium/
const url2 = generateCustomPolicySignedUrl({
    resourceUrl: 'https://cdn.site.com/premium/*',
    expiresIn: 86400,                   // 24 horas
    startsIn: 0
});
console.log('URL wildcard /premium/*:', url2);

// Exemplo 3: URL que só funciona amanhã (agendada)
const url3 = generateCustomPolicySignedUrl({
    resourceUrl: 'https://cdn.site.com/premium/live-event.mp4',
    expiresIn: 90000,                   // 25 horas a partir de agora
    startsIn: 86400                     // Só funciona daqui 24h
});
console.log('URL agendada (amanhã):', url3);
```

#### 7. Gerar Signed URL — Python (boto3)

```python
# generate_signed_url.py
import datetime
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
from botocore.signers import CloudFrontSigner

def rsa_signer(message):
    with open('private_key.pem', 'rb') as key_file:
        private_key = serialization.load_pem_private_key(
            key_file.read(), password=None
        )
    return private_key.sign(message, padding.PKCS1v15(), hashes.SHA1())

# Criar signer
key_pair_id = 'K1ABCDEFGHIJKL'
signer = CloudFrontSigner(key_pair_id, rsa_signer)

# Gerar URL com Canned Policy
url = signer.generate_presigned_url(
    url='https://cdn.site.com/premium/video.mp4',
    date_less_than=datetime.datetime.utcnow() + datetime.timedelta(hours=1)
)
print(f"Signed URL: {url}")

# Gerar URL com Custom Policy
custom_policy = signer.build_policy(
    resource='https://cdn.site.com/premium/*',
    date_less_than=datetime.datetime.utcnow() + datetime.timedelta(hours=24),
    date_greater_than=datetime.datetime.utcnow(),
    ip_address='203.0.113.0/24'
)
url_custom = signer.generate_presigned_url(
    url='https://cdn.site.com/premium/video.mp4',
    policy=custom_policy
)
print(f"Custom Policy URL: {url_custom}")
```

#### 8. Gerar Signed URL — AWS CLI

```bash
# AWS CLI tem comando built-in para signed URLs
aws cloudfront sign \
    --url "https://cdn.site.com/premium/video.mp4" \
    --key-pair-id $PUBLIC_KEY_ID \
    --private-key file://private_key.pem \
    --date-less-than "$(date -u -d '+1 hour' +%Y-%m-%dT%H:%M:%S)"
```

### Canned Policy vs Custom Policy

| Aspecto | Canned Policy | Custom Policy |
|---------|---------------|---------------|
| **Parâmetros na URL** | `Expires`, `Signature`, `Key-Pair-Id` | `Policy`, `Signature`, `Key-Pair-Id` |
| **Expiração** | Sim | Sim |
| **Data de início** | Não | Sim |
| **Restrição por IP** | Não | Sim |
| **Wildcard no resource** | Não (URL exata) | Sim (`/premium/*`) |
| **Tamanho da URL** | Menor (sem policy codificada) | Maior (policy em base64) |
| **Cacheabilidade** | Melhor (mesma URL para todos) | Pior (cada policy = URL diferente) |
| **Quando usar** | Links simples com expiração | Controle granular (IP, wildcard, agendamento) |

### Terraform Completo

```hcl
# Gerar key pair
resource "tls_private_key" "signed_url" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

# Public key no CloudFront
resource "aws_cloudfront_public_key" "signed_url" {
  name        = "signed-url-key"
  comment     = "Public key para signed URLs"
  encoded_key = tls_private_key.signed_url.public_key_pem
}

# Key Group
resource "aws_cloudfront_key_group" "signed_url" {
  name    = "signed-url-key-group"
  comment = "Key group para signed URLs"
  items   = [aws_cloudfront_public_key.signed_url.id]
}

# Salvar private key no Secrets Manager
resource "aws_secretsmanager_secret" "signing_key" {
  name = "cloudfront/signing-private-key"
}

resource "aws_secretsmanager_secret_version" "signing_key" {
  secret_id     = aws_secretsmanager_secret.signing_key.id
  secret_string = tls_private_key.signed_url.private_key_pem
}

# Distribution com behavior restrito
resource "aws_cloudfront_distribution" "restricted" {
  enabled             = true
  default_root_object = "index.html"

  origin {
    domain_name              = aws_s3_bucket.content.bucket_regional_domain_name
    origin_id                = "S3-Content"
    origin_access_control_id = aws_cloudfront_origin_access_control.main.id
  }

  # Behavior público (site principal)
  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-Content"
    cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true
  }

  # Behavior restrito (conteúdo premium)
  ordered_cache_behavior {
    path_pattern           = "/premium/*"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-Content"
    cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6"
    viewer_protocol_policy = "https-only"
    compress               = true

    # EXIGIR signed URLs
    trusted_key_groups = [aws_cloudfront_key_group.signed_url.id]
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

# Outputs
output "public_key_id" {
  value = aws_cloudfront_public_key.signed_url.id
}

output "key_group_id" {
  value = aws_cloudfront_key_group.signed_url.id
}
```

### Validação

```bash
# 1. Tentar acessar conteúdo premium SEM assinatura
curl -I "https://$DOMAIN/premium/video.mp4"
# HTTP/2 403 — "Missing Key-Pair-Id query string parameter..."

# 2. Acessar com signed URL válida
SIGNED_URL=$(aws cloudfront sign \
    --url "https://$DOMAIN/premium/video.mp4" \
    --key-pair-id $PUBLIC_KEY_ID \
    --private-key file://private_key.pem \
    --date-less-than "$(date -u -d '+1 hour' +%Y-%m-%dT%H:%M:%S)")

curl -I "$SIGNED_URL"
# HTTP/2 200 — Acesso concedido!

# 3. Testar URL expirada
EXPIRED_URL=$(aws cloudfront sign \
    --url "https://$DOMAIN/premium/video.mp4" \
    --key-pair-id $PUBLIC_KEY_ID \
    --private-key file://private_key.pem \
    --date-less-than "$(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%S)")

curl -I "$EXPIRED_URL"
# HTTP/2 403 — "Access denied: URL has expired"

# 4. Conteúdo público continua acessível sem assinatura
curl -I "https://$DOMAIN/index.html"
# HTTP/2 200 — OK (não está no behavior /premium/*)

# 5. Verificar que signed URLs geram cache entries corretas
curl -I "$SIGNED_URL"
# x-cache: Hit from cloudfront (segunda request = cache hit)
```

### O Que Aprendemos

| Conceito | Descrição |
|----------|-----------|
| **Signed URLs** | URLs com assinatura criptográfica que garante autenticidade e expiração |
| **RSA Key Pair** | Par de chaves assimétricas para assinar e verificar |
| **Public Key** | Chave pública registrada no CloudFront para verificação |
| **Key Group** | Agrupamento de public keys (permite rotação) |
| **Trusted Key Groups** | Configuração no behavior que exige assinatura |
| **Canned Policy** | Assinatura simples com apenas expiração |
| **Custom Policy** | Assinatura avançada com IP, data início, wildcard |
| **URL-safe Base64** | Substituir `+/=` por `-~_` para uso em URLs |

### Troubleshooting

| Problema | Causa | Solução |
|----------|-------|---------|
| "Missing Key-Pair-Id" | URL sem parâmetros de assinatura | Verificar se todos os 3 params estão na URL |
| "Invalid signature" | Private key não corresponde à public key | Verificar que estão usando o par correto |
| "URL has expired" | Timestamp no passado | Verificar timezone (usar UTC) |
| 403 em URL válida | Behavior errado (sem trusted key groups) | Verificar que o path pattern está correto |
| Assinatura inválida com `+` na URL | Encoding incorreto | Usar URL-safe base64 (`+`→`-`, `/`→`~`, `=`→`_`) |

### Dica Expert
> **Rotação de chaves sem downtime:** Key Groups suportam múltiplas public keys. Para rotacionar: (1) Gere novo key pair, (2) Adicione a nova public key ao Key Group, (3) Atualize o backend para usar a nova private key, (4) Aguarde TTL do cache expirar, (5) Remova a public key antiga do Key Group. Zero downtime.

---

## Desafio 24: Signed Cookies — Acesso a Múltiplos Arquivos Restritos

> **Level:** 300 | **Tempo:** 30 min | **Custo:** Free tier

### Objetivo
Usar signed cookies para dar acesso a múltiplos arquivos de uma vez (toda a pasta `/premium/`), sem precisar gerar URL individual para cada arquivo.

### Contexto Real
Na Udemy, quando você compra um curso, ganha acesso a todas as aulas (vídeos, PDFs, exercícios). São centenas de arquivos. Gerar signed URL para cada um seria impraticável. Signed cookies resolvem: um cookie dá acesso a tudo no path.

### Signed URLs vs Signed Cookies

| Aspecto | Signed URLs | Signed Cookies |
|---------|-------------|----------------|
| **Escopo** | Um arquivo por URL | Todos os arquivos no path |
| **Onde fica a assinatura** | Na URL (query string) | Em cookies do browser |
| **Uso típico** | Download individual, link por email | Área de membros, streaming |
| **Compartilhamento** | Mais fácil compartilhar link | Cookies são por browser |
| **RTMP streaming** | Obrigatório | Não suportado |
| **Player de vídeo** | Precisa suportar query params na URL | Funciona naturalmente |
| **Melhor para** | Links temporários, downloads | Sessões de acesso prolongado |

### Passo a Passo

#### 1. Gerar Signed Cookies (Node.js)

```javascript
// generate-signed-cookies.js
const crypto = require('crypto');
const fs = require('fs');

const PRIVATE_KEY_PATH = './private_key.pem';
const KEY_PAIR_ID = 'K1ABCDEFGHIJKL';

function generateSignedCookies(options) {
    const { resource, expiresIn, ipAddress } = options;
    const privateKey = fs.readFileSync(PRIVATE_KEY_PATH, 'utf8');
    const expires = Math.floor(Date.now() / 1000) + expiresIn;

    // Criar policy
    const policy = JSON.stringify({
        Statement: [{
            Resource: resource,
            Condition: {
                DateLessThan: { 'AWS:EpochTime': expires },
                ...(ipAddress && {
                    IpAddress: { 'AWS:SourceIp': ipAddress }
                })
            }
        }]
    });

    // Assinar
    const signer = crypto.createSign('RSA-SHA1');
    signer.update(policy);
    const signature = signer.sign(privateKey, 'base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '~')
        .replace(/=/g, '_');

    const policyBase64 = Buffer.from(policy)
        .toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '~')
        .replace(/=/g, '_');

    // Retornar os 3 cookies necessários
    return {
        'CloudFront-Policy': policyBase64,
        'CloudFront-Signature': signature,
        'CloudFront-Key-Pair-Id': KEY_PAIR_ID
    };
}

// Gerar cookies para acesso a /premium/* por 24h
const cookies = generateSignedCookies({
    resource: 'https://cdn.site.com/premium/*',
    expiresIn: 86400,   // 24 horas
});

console.log('Set-Cookie headers para enviar ao browser:');
Object.entries(cookies).forEach(([name, value]) => {
    console.log(`Set-Cookie: ${name}=${value}; Path=/premium; Domain=cdn.site.com; Secure; HttpOnly; SameSite=Lax`);
});
```

#### 2. Backend Express que Seta os Cookies

```javascript
// server.js
const express = require('express');
const { generateSignedCookies } = require('./generate-signed-cookies');

const app = express();
const DOMAIN = 'cdn.site.com';

// Endpoint de login/autorização
app.post('/api/authorize-premium', authenticateUser, async (req, res) => {
    // Verificar se o usuário tem acesso premium
    if (!req.user.isPremium) {
        return res.status(403).json({ error: 'Premium access required' });
    }

    // Gerar signed cookies
    const cookies = generateSignedCookies({
        resource: `https://${DOMAIN}/premium/*`,
        expiresIn: 86400 // 24h
    });

    // Setar os 3 cookies
    Object.entries(cookies).forEach(([name, value]) => {
        res.cookie(name, value, {
            domain: DOMAIN,
            path: '/premium',
            secure: true,
            httpOnly: true,
            sameSite: 'Lax',
            maxAge: 86400 * 1000 // 24h em ms
        });
    });

    res.json({ success: true, message: 'Premium access granted for 24h' });
});

app.listen(3000);
```

### Validação

```bash
# 1. Acessar sem cookies (deve falhar)
curl -I "https://$DOMAIN/premium/video-01.mp4"
# HTTP/2 403

# 2. Gerar cookies e acessar
# (usando os valores gerados pelo script)
curl -I "https://$DOMAIN/premium/video-01.mp4" \
    -b "CloudFront-Policy=BASE64_POLICY; CloudFront-Signature=SIGNATURE; CloudFront-Key-Pair-Id=K1ABC"
# HTTP/2 200 — Acesso concedido!

# 3. MESMO cookie funciona para QUALQUER arquivo em /premium/
curl -I "https://$DOMAIN/premium/video-02.mp4" \
    -b "CloudFront-Policy=BASE64_POLICY; CloudFront-Signature=SIGNATURE; CloudFront-Key-Pair-Id=K1ABC"
# HTTP/2 200

curl -I "https://$DOMAIN/premium/docs/guia.pdf" \
    -b "CloudFront-Policy=BASE64_POLICY; CloudFront-Signature=SIGNATURE; CloudFront-Key-Pair-Id=K1ABC"
# HTTP/2 200

# 4. Conteúdo fora do /premium/ continua público
curl -I "https://$DOMAIN/index.html"
# HTTP/2 200 (não precisa de cookie)
```

### Dica Expert
> **Cuidado com CORS e cookies cross-domain.** Se seu frontend é `app.site.com` e o CDN é `cdn.site.com`, os cookies precisam ter `Domain=.site.com` (com ponto no início para incluir subdomínios). Além disso, o browser precisa de `SameSite=None; Secure` para enviar cookies cross-origin, e o CloudFront precisa de `Access-Control-Allow-Credentials: true` na response headers policy.

---

## Desafio 25: AWS WAF + CloudFront — Firewall Completo

> **Level:** 300 | **Tempo:** 60 min | **Custo:** ~$5-10/mês (WAF tem custo)

### Objetivo
Criar um Web ACL completo com múltiplas camadas de proteção: managed rules, rate limiting, IP blocking, geo blocking e regras customizadas.

### Contexto Real
Todo site em produção precisa de WAF. Sem ele, você está vulnerável a SQL injection, XSS, bots maliciosos, scrapers, e ataques volumétricos. O WAF é a primeira linha de defesa no edge.

### Arquitetura

```
Internet → CloudFront → WAF Web ACL → Origin
                          │
              ┌───────────┼───────────┐
              │           │           │
          Managed     Rate-Based   Custom
          Rules       Rules        Rules
              │           │           │
         ┌────┤      ┌────┤      ┌────┤
         │    │      │    │      │    │
        SQLi  XSS   IP   Geo   Regex Header
        Rules Rules  Block Block Match Match
```

### Passo a Passo

#### 1. Criar IP Set (whitelist/blacklist)

```bash
# IP Set para IPs bloqueados
aws wafv2 create-ip-set \
    --name "Blocked-IPs" \
    --scope CLOUDFRONT \
    --region us-east-1 \
    --ip-address-version IPV4 \
    --addresses "198.51.100.0/24" "203.0.113.50/32"

# IP Set para IPs internos (whitelist)
aws wafv2 create-ip-set \
    --name "Internal-IPs" \
    --scope CLOUDFRONT \
    --region us-east-1 \
    --ip-address-version IPV4 \
    --addresses "10.0.0.0/8" "172.16.0.0/12" "192.168.0.0/16"
```

#### 2. Criar Regex Pattern Set

```bash
# Padrões de User-Agent maliciosos
aws wafv2 create-regex-pattern-set \
    --name "Bad-User-Agents" \
    --scope CLOUDFRONT \
    --region us-east-1 \
    --regular-expression-list \
        '[{"RegexString": "(?i)(nikto|sqlmap|nmap|masscan|dirbuster|gobuster)"},
          {"RegexString": "(?i)(scanner|crawler|scraper|harvest)"},
          {"RegexString": "^$"}]'
```

#### 3. Criar Web ACL Completo

```bash
aws wafv2 create-web-acl \
    --name "CloudFront-Protection" \
    --scope CLOUDFRONT \
    --region us-east-1 \
    --default-action '{"Allow": {}}' \
    --visibility-config '{
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CloudFrontWAF"
    }' \
    --rules '[
        {
            "Name": "AWS-AWSManagedRulesCommonRuleSet",
            "Priority": 1,
            "Statement": {
                "ManagedRuleGroupStatement": {
                    "VendorName": "AWS",
                    "Name": "AWSManagedRulesCommonRuleSet"
                }
            },
            "OverrideAction": {"None": {}},
            "VisibilityConfig": {
                "SampledRequestsEnabled": true,
                "CloudWatchMetricsEnabled": true,
                "MetricName": "CommonRules"
            }
        },
        {
            "Name": "AWS-AWSManagedRulesSQLiRuleSet",
            "Priority": 2,
            "Statement": {
                "ManagedRuleGroupStatement": {
                    "VendorName": "AWS",
                    "Name": "AWSManagedRulesSQLiRuleSet"
                }
            },
            "OverrideAction": {"None": {}},
            "VisibilityConfig": {
                "SampledRequestsEnabled": true,
                "CloudWatchMetricsEnabled": true,
                "MetricName": "SQLiRules"
            }
        },
        {
            "Name": "AWS-AWSManagedRulesKnownBadInputsRuleSet",
            "Priority": 3,
            "Statement": {
                "ManagedRuleGroupStatement": {
                    "VendorName": "AWS",
                    "Name": "AWSManagedRulesKnownBadInputsRuleSet"
                }
            },
            "OverrideAction": {"None": {}},
            "VisibilityConfig": {
                "SampledRequestsEnabled": true,
                "CloudWatchMetricsEnabled": true,
                "MetricName": "KnownBadInputs"
            }
        },
        {
            "Name": "RateLimit-2000-per-5min",
            "Priority": 4,
            "Statement": {
                "RateBasedStatement": {
                    "Limit": 2000,
                    "AggregateKeyType": "IP"
                }
            },
            "Action": {"Block": {}},
            "VisibilityConfig": {
                "SampledRequestsEnabled": true,
                "CloudWatchMetricsEnabled": true,
                "MetricName": "RateLimit"
            }
        },
        {
            "Name": "Block-Bad-IPs",
            "Priority": 5,
            "Statement": {
                "IPSetReferenceStatement": {
                    "ARN": "arn:aws:wafv2:us-east-1:123456789012:global/ipset/Blocked-IPs/xxx"
                }
            },
            "Action": {"Block": {}},
            "VisibilityConfig": {
                "SampledRequestsEnabled": true,
                "CloudWatchMetricsEnabled": true,
                "MetricName": "BlockedIPs"
            }
        }
    ]'
```

#### 4. Terraform Completo

```hcl
# Web ACL
resource "aws_wafv2_web_acl" "cloudfront" {
  provider    = aws.us_east_1  # WAF para CloudFront DEVE ser us-east-1
  name        = "cloudfront-protection"
  scope       = "CLOUDFRONT"
  description = "WAF para proteção do CloudFront"

  default_action {
    allow {}
  }

  # Regra 1: AWS Managed — Common Rule Set
  rule {
    name     = "AWS-Common-Rules"
    priority = 1

    override_action { none {} }

    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesCommonRuleSet"

        # Excluir regras que geram falsos positivos
        rule_action_override {
          name = "SizeRestrictions_BODY"
          action_to_use { count {} }  # Contar em vez de bloquear
        }
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRules"
    }
  }

  # Regra 2: AWS Managed — SQL Injection
  rule {
    name     = "AWS-SQLi-Rules"
    priority = 2
    override_action { none {} }

    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesSQLiRuleSet"
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLiRules"
    }
  }

  # Regra 3: AWS Managed — Known Bad Inputs (Log4Shell, etc)
  rule {
    name     = "AWS-KnownBadInputs"
    priority = 3
    override_action { none {} }

    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "KnownBadInputs"
    }
  }

  # Regra 4: AWS Managed — Bot Control
  rule {
    name     = "AWS-BotControl"
    priority = 4
    override_action { none {} }

    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesBotControlRuleSet"

        managed_rule_group_configs {
          aws_managed_rules_bot_control_rule_set {
            inspection_level = "COMMON"  # ou "TARGETED" para bots sofisticados
          }
        }
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "BotControl"
    }
  }

  # Regra 5: Rate Limiting — 2000 requests por 5 min por IP
  rule {
    name     = "RateLimit-PerIP"
    priority = 5

    action { block {} }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
    }
  }

  # Regra 6: Rate Limiting específico para login
  rule {
    name     = "RateLimit-Login"
    priority = 6

    action { block {} }

    statement {
      rate_based_statement {
        limit              = 100  # 100 tentativas por 5 min
        aggregate_key_type = "IP"

        scope_down_statement {
          byte_match_statement {
            field_to_match { uri_path {} }
            positional_constraint = "STARTS_WITH"
            search_string         = "/api/login"
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
          }
        }
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitLogin"
    }
  }

  # Regra 7: Bloquear IPs específicos
  rule {
    name     = "Block-Bad-IPs"
    priority = 7

    action { block {} }

    statement {
      ip_set_reference_statement {
        arn = aws_wafv2_ip_set.blocked.arn
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "BlockedIPs"
    }
  }

  # Regra 8: Geo Blocking (bloquear países específicos)
  rule {
    name     = "Geo-Block"
    priority = 8

    action { block {} }

    statement {
      geo_match_statement {
        country_codes = ["RU", "CN", "KP"]  # Rússia, China, Coreia do Norte
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "GeoBlock"
    }
  }

  # Regra 9: Custom — Bloquear User-Agents maliciosos
  rule {
    name     = "Block-Bad-UserAgents"
    priority = 9

    action { block {} }

    statement {
      regex_pattern_set_reference_statement {
        arn = aws_wafv2_regex_pattern_set.bad_user_agents.arn
        field_to_match {
          single_header { name = "user-agent" }
        }
        text_transformation {
          priority = 0
          type     = "LOWERCASE"
        }
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "BadUserAgents"
    }
  }

  # Regra 10: Custom — Exigir header específico para API
  rule {
    name     = "Require-API-Key-Header"
    priority = 10

    action { block {} }

    statement {
      and_statement {
        statement {
          byte_match_statement {
            field_to_match { uri_path {} }
            positional_constraint = "STARTS_WITH"
            search_string         = "/api/"
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
          }
        }
        statement {
          not_statement {
            statement {
              size_constraint_statement {
                field_to_match {
                  single_header { name = "x-api-key" }
                }
                comparison_operator = "GT"
                size                = 0
                text_transformation {
                  priority = 0
                  type     = "NONE"
                }
              }
            }
          }
        }
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "RequireAPIKey"
    }
  }

  visibility_config {
    sampled_requests_enabled   = true
    cloudwatch_metrics_enabled = true
    metric_name                = "CloudFrontWAF"
  }

  tags = {
    Environment = "production"
  }
}

# IP Sets
resource "aws_wafv2_ip_set" "blocked" {
  provider           = aws.us_east_1
  name               = "blocked-ips"
  scope              = "CLOUDFRONT"
  ip_address_version = "IPV4"
  addresses          = ["198.51.100.0/24", "203.0.113.50/32"]
}

# Regex Pattern Set
resource "aws_wafv2_regex_pattern_set" "bad_user_agents" {
  provider = aws.us_east_1
  name     = "bad-user-agents"
  scope    = "CLOUDFRONT"

  regular_expression { regex_string = "(?i)(nikto|sqlmap|nmap|masscan)" }
  regular_expression { regex_string = "(?i)(dirbuster|gobuster|scanner)" }
  regular_expression { regex_string = "^$" }  # User-Agent vazio
}

# Associar WAF ao CloudFront
resource "aws_cloudfront_distribution" "protected" {
  # ...
  web_acl_id = aws_wafv2_web_acl.cloudfront.arn
  # ...
}

# WAF Logging
resource "aws_wafv2_web_acl_logging_configuration" "main" {
  provider                = aws.us_east_1
  log_destination_configs = [aws_cloudwatch_log_group.waf.arn]
  resource_arn            = aws_wafv2_web_acl.cloudfront.arn

  logging_filter {
    default_behavior = "DROP"

    filter {
      behavior    = "KEEP"
      requirement = "MEETS_ANY"

      condition {
        action_condition { action = "BLOCK" }
      }
      condition {
        action_condition { action = "COUNT" }
      }
    }
  }
}

resource "aws_cloudwatch_log_group" "waf" {
  provider          = aws.us_east_1
  name              = "aws-waf-logs-cloudfront"  # DEVE começar com "aws-waf-logs-"
  retention_in_days = 30
}
```

### Validação

```bash
# 1. Testar SQL injection (deve ser bloqueado)
curl -I "https://$DOMAIN/api/users?id=1' OR '1'='1"
# HTTP/2 403 (bloqueado pelo SQLi ruleset)

# 2. Testar XSS (deve ser bloqueado)
curl -I "https://$DOMAIN/search?q=<script>alert('xss')</script>"
# HTTP/2 403 (bloqueado pelo Common ruleset)

# 3. Testar rate limiting
for i in $(seq 1 50); do
    curl -s -o /dev/null -w "%{http_code} " "https://$DOMAIN/api/login" \
        -X POST -d "user=test&pass=test$i"
done
# Após ~100 requests: 403 403 403... (rate limited)

# 4. Testar User-Agent bloqueado
curl -A "sqlmap/1.0" -I "https://$DOMAIN/"
# HTTP/2 403

# 5. Testar API sem x-api-key
curl -I "https://$DOMAIN/api/data"
# HTTP/2 403 (falta x-api-key)

curl -I "https://$DOMAIN/api/data" -H "x-api-key: my-key"
# HTTP/2 200

# 6. Ver métricas WAF no CloudWatch
aws cloudwatch get-metric-statistics \
    --namespace AWS/WAFV2 \
    --metric-name BlockedRequests \
    --dimensions Name=WebACL,Value=cloudfront-protection Name=Rule,Value=ALL Name=Region,Value=us-east-1 \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Sum

# 7. Ver sampled requests
aws wafv2 get-sampled-requests \
    --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:global/webacl/cloudfront-protection/xxx" \
    --rule-metric-name "RateLimit" \
    --scope CLOUDFRONT \
    --region us-east-1 \
    --time-window '{"StartTime": "2025-01-01T00:00:00Z", "EndTime": "2025-12-31T23:59:59Z"}' \
    --max-items 10
```

### Custo do WAF

| Componente | Preço |
|-----------|-------|
| Web ACL | $5.00/mês |
| Regra (por regra) | $1.00/mês |
| Managed Rule Group | $1.00-25.00/mês (varia) |
| Bot Control (Common) | $10.00/mês + $1.00/1M requests |
| Requests analisados | $0.60/1M requests |

> **Exemplo:** Web ACL + 5 regras + Common Rules + SQLi + Bot Control ≈ $25/mês + $0.60/1M requests

### Dica Expert
> **Sempre comece no modo COUNT** (não BLOCK). Habilite o WAF com `override_action { count {} }` por 1-2 semanas para analisar o que seria bloqueado sem impactar o tráfego real. Depois, analise os sampled requests e ajuste as regras antes de mudar para BLOCK. Regras de Bot Control podem bloquear crawlers legítimos (Googlebot) — sempre crie exceções para bots que você quer permitir.

---

## Desafio 26: AWS Shield + CloudFront — Proteção Anti-DDoS

> **Level:** 300 | **Tempo:** 20 min | **Custo:** Shield Standard = Grátis; Advanced = $3000/mês

### Objetivo
Entender as camadas de proteção DDoS do CloudFront e quando investir em Shield Advanced.

### Shield Standard vs Shield Advanced

| Aspecto | Shield Standard | Shield Advanced |
|---------|----------------|-----------------|
| **Custo** | Grátis (incluso) | $3,000/mês + data transfer fees |
| **Proteção** | L3/L4 (SYN flood, UDP reflection) | L3/L4 + L7 (HTTP flood) |
| **Detecção** | Automática | Automática + personalizada |
| **Mitigação** | Automática | Automática + manuais |
| **DRT (Response Team)** | Não | Sim (24/7) |
| **Cost Protection** | Não | Sim (reembolso de spike por DDoS) |
| **Visibilidade** | Básica | Dashboards detalhados, métricas |
| **Health Checks** | Não | Sim (Route 53 health-based detection) |
| **WAF incluso** | Não | Sim (WAF sem custo adicional) |
| **SLA** | Não | Sim |

### Como CloudFront Protege Automaticamente

```
Ataque DDoS → CloudFront absorve na edge (600+ locations)
                    │
                    ├── Ataques volumétricos (L3/L4)
                    │   → Shield Standard mitiga automaticamente
                    │   → Distribuído por 600+ edge locations
                    │   → Capacidade de absorção: Tbps
                    │
                    ├── Ataques HTTP flood (L7)
                    │   → WAF rate limiting
                    │   → Bot Control
                    │   → Shield Advanced (detecção inteligente)
                    │
                    └── Origin protegida
                        → Recebe apenas tráfego legítimo
                        → Rate de requests normalizado
```

### Terraform para Shield Advanced

```hcl
# Habilitar Shield Advanced (conta inteira)
resource "aws_shield_protection" "cloudfront" {
  name         = "cloudfront-protection"
  resource_arn = aws_cloudfront_distribution.main.arn

  tags = {
    Environment = "production"
  }
}

# Grupo de proteção
resource "aws_shield_protection_group" "web" {
  protection_group_id = "web-tier"
  aggregation         = "MAX"
  pattern             = "BY_RESOURCE_TYPE"
  resource_type       = "CLOUDFRONT_DISTRIBUTION"
}

# Habilitar proactive engagement (DRT liga para você)
resource "aws_shield_proactive_engagement" "main" {
  enabled = true

  emergency_contact {
    email_address = "security@meusite.com"
    phone_number  = "+5511999999999"
    contact_notes = "Equipe de SRE - 24/7"
  }
}
```

### Dica Expert
> **Shield Advanced vale a pena quando:** (1) seu site gera >$3000/mês de receita (o custo se paga em proteção), (2) downtime tem impacto legal/regulatório, (3) você precisa de SLA de DDoS mitigation, (4) o WAF "grátis" incluso no Advanced já paga o investimento. Para a maioria dos sites, Shield Standard + WAF com rate limiting é suficiente.

---

## Desafio 27: Geo Restriction — Bloqueio Geográfico

> **Level:** 300 | **Tempo:** 20 min | **Custo:** Free tier

### Objetivo
Implementar restrição geográfica usando CloudFront native e WAF, entendendo as diferenças.

### CloudFront Geo Restriction vs WAF Geo Match

| Aspecto | CF Geo Restriction | WAF Geo Match |
|---------|-------------------|---------------|
| **Nível** | Distribution inteira | Por behavior (via regra) |
| **Tipo** | Whitelist OU Blacklist | Lógica condicional complexa |
| **Granularidade** | País apenas | País + combinações |
| **Resposta** | 403 Forbidden | 403 customizável |
| **Exceções** | Não permite | Sim (combinar com IP whitelist) |
| **Custo** | Grátis | Custo do WAF |
| **Melhor para** | Bloqueio simples | Lógica complexa (ex: bloquear país EXCETO IP X) |

### CloudFront Native

```hcl
resource "aws_cloudfront_distribution" "geo_restricted" {
  # ...

  restrictions {
    geo_restriction {
      restriction_type = "whitelist"  # ou "blacklist"
      locations        = ["BR", "PT", "US", "CA"]  # ISO 3166-1 alpha-2
    }
  }
}
```

### WAF Geo Block (mais flexível)

```hcl
# Bloquear países MAS permitir IPs internos
rule {
  name     = "Geo-Block-With-Exception"
  priority = 1
  action   { block {} }

  statement {
    and_statement {
      # Condição 1: país bloqueado
      statement {
        geo_match_statement {
          country_codes = ["RU", "CN", "KP", "IR"]
        }
      }
      # Condição 2: NÃO é IP da whitelist
      statement {
        not_statement {
          statement {
            ip_set_reference_statement {
              arn = aws_wafv2_ip_set.internal.arn
            }
          }
        }
      }
    }
  }
}
```

### Validação

```bash
# Simular acesso de diferentes países (via CloudFront-Viewer-Country header)
# Nota: em teste real, use VPN de diferentes países

# Verificar geo restriction
curl -I "https://$DOMAIN/"
# Se seu IP está em país permitido: 200
# Se não: 403

# Verificar que a restrição está aplicada
aws cloudfront get-distribution --id $DIST_ID \
    --query "Distribution.DistributionConfig.Restrictions.GeoRestriction"
```

---

## Desafio 28: Field-Level Encryption

> **Level:** 400 | **Tempo:** 45 min | **Custo:** $0.02/10k requests

### Objetivo
Criptografar campos específicos de formulários no edge, antes dos dados chegarem à origin.

### Contexto Real
Um e-commerce coleta número de cartão de crédito. Com FLE, o campo do cartão é criptografado no edge do CloudFront com sua chave pública. O servidor web recebe o campo criptografado e NÃO pode decriptar. Apenas o sistema de pagamento (com a private key) decripta. Isso reduz o escopo de PCI DSS.

### Fluxo

```
1. Browser envia POST com form data:
   credit_card=4111111111111111&name=João

2. CloudFront recebe e criptografa o campo credit_card:
   credit_card=AQB8...encrypted...&name=João

3. Origin (web server) recebe:
   credit_card=AQB8...encrypted...  ← NÃO consegue ler
   name=João                        ← Texto normal

4. Apenas o payment service (com private key) decripta:
   credit_card=4111111111111111     ← Descriptografado
```

### Passo a Passo

```bash
# 1. Gerar key pair para FLE
openssl genrsa -out fle_private_key.pem 2048
openssl rsa -pubout -in fle_private_key.pem -out fle_public_key.pem

# 2. Criar public key no CloudFront
FLE_KEY_ID=$(aws cloudfront create-public-key \
    --public-key-config '{
        "CallerReference": "fle-key-'$(date +%s)'",
        "Name": "FLE-Payment-Key",
        "Comment": "Chave para field-level encryption de dados de pagamento",
        "EncodedKey": "'"$(cat fle_public_key.pem)"'"
    }' \
    --query "PublicKey.Id" --output text)

# 3. Criar field-level encryption profile
FLE_PROFILE_ID=$(aws cloudfront create-field-level-encryption-profile \
    --field-level-encryption-profile-config '{
        "Name": "Payment-FLE-Profile",
        "CallerReference": "fle-profile-'$(date +%s)'",
        "Comment": "Criptografa campos de pagamento",
        "EncryptionEntities": {
            "Quantity": 1,
            "Items": [
                {
                    "PublicKeyId": "'$FLE_KEY_ID'",
                    "ProviderId": "PaymentProvider",
                    "FieldPatterns": {
                        "Quantity": 3,
                        "Items": ["credit_card", "cvv", "card_number"]
                    }
                }
            ]
        }
    }' \
    --query "FieldLevelEncryptionProfile.Id" --output text)

# 4. Criar field-level encryption config
FLE_CONFIG_ID=$(aws cloudfront create-field-level-encryption-config \
    --field-level-encryption-config '{
        "CallerReference": "fle-config-'$(date +%s)'",
        "Comment": "FLE config para pagamento",
        "QueryArgProfileConfig": {
            "ForwardWhenQueryArgProfileIsUnknown": true,
            "QueryArgProfiles": {
                "Quantity": 0
            }
        },
        "ContentTypeProfileConfig": {
            "ForwardWhenContentTypeIsUnknown": true,
            "ContentTypeProfiles": {
                "Quantity": 1,
                "Items": [
                    {
                        "Format": "URLEncoded",
                        "ProfileId": "'$FLE_PROFILE_ID'",
                        "ContentType": "application/x-www-form-urlencoded"
                    }
                ]
            }
        }
    }' \
    --query "FieldLevelEncryption.Id" --output text)

# 5. Associar ao behavior (no default ou em um behavior específico)
# Adicionar "FieldLevelEncryptionId": "$FLE_CONFIG_ID" no behavior
```

### Validação

```bash
# 1. Enviar POST com dados de cartão
curl -X POST "https://$DOMAIN/api/payment" \
    -d "credit_card=4111111111111111&name=João&amount=99.90"

# 2. No server, verificar que credit_card chegou criptografado
# O campo será algo como: AQB8abc123...base64encoded...
# enquanto name e amount chegam normalmente

# 3. Descriptografar no backend (Python)
# from cryptography.hazmat.primitives.asymmetric import padding
# from cryptography.hazmat.primitives import hashes, serialization
# decrypted = private_key.decrypt(encrypted_data, padding.OAEP(...))
```

### Dica Expert
> **FLE é pouco usado na prática** porque a maioria dos e-commerces usa payment gateways (Stripe, PagSeguro) que lidam com criptografia no SDK do cliente. FLE brilha quando você tem um formulário custom e precisa garantir que dados sensíveis nunca tocam seu web server em texto plano. É excelente para compliance PCI DSS Level 1.

---

## Desafio 29: Origin Access — OAC vs OAI Deep Dive

> **Level:** 300 | **Tempo:** 30 min | **Custo:** Free tier

### Objetivo
Entender profundamente as diferenças entre OAC e OAI, e migrar de OAI para OAC.

### OAI vs OAC — Comparativo Completo

| Aspecto | OAI (Legacy) | OAC (Moderno) |
|---------|-------------|---------------|
| **Lançamento** | 2008 | 2022 |
| **Signing** | CloudFront signing | SigV4 |
| **S3 SSE-S3** | Sim | Sim |
| **S3 SSE-KMS** | **Não** | **Sim** |
| **S3 PUT/DELETE** | **Não** | **Sim** |
| **S3 todas regiões** | Problemas em novas regiões | **Todas** |
| **Lambda Function URLs** | **Não** | **Sim** |
| **MediaStore** | **Não** | **Sim** |
| **MediaPackage v2** | **Não** | **Sim** |
| **Elemental** | **Não** | **Sim** |
| **Bucket policy** | Principal: CloudFront OAI | Principal: cloudfront.amazonaws.com + Condition |
| **Recomendação AWS** | Migrar para OAC | **Usar para novas distributions** |

### Migração OAI → OAC

```bash
# 1. Criar OAC
OAC_ID=$(aws cloudfront create-origin-access-control \
    --origin-access-control-config '{
        "Name": "S3-OAC-Migration",
        "SigningProtocol": "sigv4",
        "SigningBehavior": "always",
        "OriginAccessControlOriginType": "s3"
    }' --query "OriginAccessControl.Id" --output text)

# 2. Atualizar bucket policy (aceitar AMBOS durante migração)
cat > bucket-policy-migration.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowOAC",
            "Effect": "Allow",
            "Principal": {"Service": "cloudfront.amazonaws.com"},
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::meu-bucket/*",
            "Condition": {
                "StringEquals": {"AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/E1ABC"}
            }
        },
        {
            "Sid": "AllowOAI-Legacy",
            "Effect": "Allow",
            "Principal": {"AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity E2DEF"},
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::meu-bucket/*"
        }
    ]
}
EOF

aws s3api put-bucket-policy --bucket meu-bucket --policy file://bucket-policy-migration.json

# 3. Atualizar distribution: adicionar OAC, remover OAI
# Editar config: OriginAccessControlId = OAC_ID, S3OriginConfig.OriginAccessIdentity = ""

# 4. Após confirmar que funciona, remover statement do OAI da bucket policy

# 5. Deletar o OAI antigo
aws cloudfront delete-cloud-front-origin-access-identity --id E2DEF --if-match $ETAG
```

### OAC com SSE-KMS

```hcl
# Bucket com SSE-KMS
resource "aws_s3_bucket_server_side_encryption_configuration" "kms" {
  bucket = aws_s3_bucket.content.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

# KMS key com policy para CloudFront
resource "aws_kms_key" "s3" {
  description = "KMS key para S3 bucket com CloudFront OAC"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowRoot"
        Effect = "Allow"
        Principal = { AWS = "arn:aws:iam::123456789012:root" }
        Action    = "kms:*"
        Resource  = "*"
      },
      {
        Sid    = "AllowCloudFrontOAC"
        Effect = "Allow"
        Principal = { Service = "cloudfront.amazonaws.com" }
        Action = [
          "kms:Decrypt",
          "kms:Encrypt",
          "kms:GenerateDataKey*"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.main.arn
          }
        }
      }
    ]
  })
}
```

### Dica Expert
> **Se você ainda tem OAI, migre agora.** A AWS não vai remover OAI de repente, mas todas as novas features são OAC-only. A migração é simples se você seguir o padrão: (1) criar OAC, (2) bucket policy aceita ambos, (3) atualizar distribution, (4) testar, (5) remover OAI. Zero downtime.

---

## Desafio 30: mTLS — Mutual TLS com Trust Stores

> **Level:** 400 | **Tempo:** 60 min | **Custo:** Free tier

### Objetivo
Configurar mTLS (Mutual TLS) onde o CloudFront exige que o CLIENTE apresente um certificado válido para acessar a API.

### Contexto Real
APIs B2B onde parceiros se autenticam por certificado. O CloudFront valida o certificado do cliente contra uma CA (Certificate Authority) de confiança. Se o certificado não é válido, a conexão é recusada antes de chegar à origin.

### Arquitetura

```
┌──────────────┐   TLS Handshake    ┌──────────────┐
│   Cliente     │  + Client Cert    │  CloudFront   │
│   (Parceiro)  │ ←──────────────→  │  (Edge)       │
└──────────────┘                    └──────┬───────┘
                                           │
                                    Valida cert contra
                                    Trust Store (CA)
                                           │
                                    ┌──────┴───────┐
                                    │ Trust Store   │
                                    │ (S3 bucket)   │
                                    │ - CA cert 1   │
                                    │ - CA cert 2   │
                                    └──────────────┘
                                           │
                                    Se válido:
                                           │
                                    ┌──────┴───────┐
                                    │   Origin      │
                                    │   (API)       │
                                    └──────────────┘
```

### Passo a Passo

#### 1. Criar CA (Certificate Authority) com OpenSSL

```bash
# Criar CA private key
openssl genrsa -out ca-key.pem 4096

# Criar CA certificate (auto-assinado, válido por 10 anos)
openssl req -new -x509 -sha256 -key ca-key.pem -out ca-cert.pem -days 3650 \
    -subj "/C=BR/ST=SP/L=Sao Paulo/O=MinhaEmpresa/CN=MinhaEmpresa CA"

# Verificar o CA cert
openssl x509 -in ca-cert.pem -text -noout | head -20
```

#### 2. Gerar Certificado de Cliente (assinado pela CA)

```bash
# Gerar private key do cliente
openssl genrsa -out client-key.pem 2048

# Gerar CSR (Certificate Signing Request)
openssl req -new -key client-key.pem -out client.csr \
    -subj "/C=BR/ST=SP/L=Sao Paulo/O=Parceiro ABC/CN=parceiro-abc.api.com"

# Assinar com a CA (válido por 1 ano)
openssl x509 -req -in client.csr -CA ca-cert.pem -CAkey ca-key.pem \
    -CAcreateserial -out client-cert.pem -days 365 -sha256

# Verificar o certificado do cliente
openssl verify -CAfile ca-cert.pem client-cert.pem
# client-cert.pem: OK
```

#### 3. Criar Trust Store no S3

```bash
# Upload do CA certificate para S3
aws s3 cp ca-cert.pem s3://meu-bucket-truststore/ca-bundle.pem

# Se tiver múltiplas CAs, concatenar em um bundle:
# cat ca1-cert.pem ca2-cert.pem > ca-bundle.pem
```

#### 4. Configurar CloudFront para mTLS

```bash
# Criar trust store
TRUST_STORE_ID=$(aws cloudfront create-trust-store \
    --name "B2B-API-TrustStore" \
    --s3-bucket "meu-bucket-truststore" \
    --s3-prefix "ca-bundle.pem" \
    --query "TrustStore.Id" --output text)

# Atualizar distribution para exigir mTLS
# No ViewerCertificate, adicionar ClientCertConfig
```

#### 5. Terraform

```hcl
# Trust Store
resource "aws_cloudfront_trust_store" "b2b" {
  name = "b2b-api-truststore"

  s3_config {
    s3_bucket = aws_s3_bucket.truststore.id
    s3_prefix = "ca-bundle.pem"
  }
}

# Distribution com mTLS
resource "aws_cloudfront_distribution" "mtls_api" {
  enabled = true

  origin {
    domain_name = aws_lb.api.dns_name
    origin_id   = "ALB-API"
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "ALB-API"
    cache_policy_id        = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"
    origin_request_policy_id = "216adef6-5c7f-47e4-b989-5492eafa07d3"
    viewer_protocol_policy = "https-only"
  }

  viewer_certificate {
    acm_certificate_arn            = aws_acm_certificate.api.arn
    ssl_support_method             = "sni-only"
    minimum_protocol_version       = "TLSv1.2_2021"

    # mTLS config
    client_certificate_config {
      trust_store_arn = aws_cloudfront_trust_store.b2b.arn
      # enforce = true → Rejeita conexões sem certificado válido
      # enforce = false → Passa para Lambda@Edge para decisão
    }
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }
}
```

### Validação

```bash
# 1. Tentar acessar SEM certificado de cliente
curl -I "https://api.site.com/v1/data"
# curl: (56) TLS handshake failed — ou HTTP/2 403

# 2. Acessar COM certificado de cliente válido
curl --cert client-cert.pem --key client-key.pem \
    "https://api.site.com/v1/data"
# HTTP/2 200 — Acesso concedido!

# 3. Testar com certificado de CA diferente (não confiável)
openssl genrsa -out fake-ca-key.pem 2048
openssl req -new -x509 -key fake-ca-key.pem -out fake-ca-cert.pem -days 365 -subj "/CN=FakeCA"
openssl genrsa -out fake-client-key.pem 2048
openssl req -new -key fake-client-key.pem -out fake-client.csr -subj "/CN=FakeClient"
openssl x509 -req -in fake-client.csr -CA fake-ca-cert.pem -CAkey fake-ca-key.pem -CAcreateserial -out fake-client-cert.pem -days 365

curl --cert fake-client-cert.pem --key fake-client-key.pem \
    "https://api.site.com/v1/data"
# Falha! CA não está na trust store

# 4. Verificar trust store
aws cloudfront list-trust-stores
aws cloudfront get-trust-store --id $TRUST_STORE_ID
```

### O Que Aprendemos

| Conceito | Descrição |
|----------|-----------|
| **mTLS** | Mutual TLS — ambos os lados apresentam certificado |
| **Trust Store** | Lista de CAs confiáveis para validar certificados de cliente |
| **CA Bundle** | Arquivo com um ou mais CA certificates concatenados |
| **Client Certificate** | Certificado apresentado pelo cliente durante TLS handshake |
| **CSR** | Certificate Signing Request — pedido de assinatura |
| **Zero Trust** | Autenticação forte antes de qualquer acesso |

### Dica Expert
> **mTLS é a autenticação mais forte disponível no CloudFront.** Nem JWT nem API key se comparam — o certificado é validado no nível TLS, antes de qualquer código executar. Para onboarding de parceiros: (1) gere um certificado de cliente assinado pela sua CA, (2) envie o cert + key para o parceiro de forma segura, (3) adicione a CA na trust store. Para revogar: use CRL (Certificate Revocation List) ou simplesmente gere nova CA e recrie os certificados válidos.

---

## Resumo do Módulo 05

Ao completar este módulo, você domina:

- [x] Signed URLs com Canned e Custom Policies
- [x] Signed Cookies para acesso a múltiplos arquivos
- [x] WAF completo com 10+ tipos de regra
- [x] Shield Standard e Advanced
- [x] Geo Restriction (CloudFront native e WAF)
- [x] Field-Level Encryption para dados sensíveis
- [x] OAC vs OAI e migração
- [x] mTLS com Trust Stores para autenticação B2B

### Stack de Segurança Recomendado para Produção

```
MÍNIMO (todo site):
  ✅ HTTPS (TLS 1.2+)
  ✅ OAC para S3
  ✅ Security Response Headers (HSTS, CSP)
  ✅ WAF com Common Rules + SQLi + Rate Limiting

INTERMEDIÁRIO (e-commerce, SaaS):
  ✅ Tudo acima +
  ✅ WAF Bot Control
  ✅ Signed URLs/Cookies para conteúdo premium
  ✅ Custom headers para proteção de origin
  ✅ VPC Origins

ENTERPRISE (fintech, saúde, governo):
  ✅ Tudo acima +
  ✅ Shield Advanced
  ✅ mTLS para APIs B2B
  ✅ Field-Level Encryption
  ✅ WAF com regex customizado
  ✅ Real-time logging + SIEM integration
```

---

## Desafio Bônus: Security Assessment Completo — Ataque, Defesa e Validação

> **Level:** 400 | **Tempo:** 180 min | **Custo:** ~$2-5

### Objetivo

Simular um **security assessment completo** de uma distribuição CloudFront em produção: testar ataques reais, validar que as defesas funcionam, identificar bugs silenciosos de segurança, executar testes com ferramentas profissionais e criar um **pentest checklist** reutilizável.

### Contexto

Configurar segurança é metade do trabalho. A outra metade é **validar que funciona**. Em auditorias de segurança e pentests reais, os assessores testam exatamente o que vamos testar aqui. A diferença entre um profissional e um iniciante é que o profissional testa cada camada e documenta o resultado.

```
┌──────────────────────────────────────────────────────────────────┐
│           Security Assessment — 7 Camadas de Teste                │
│                                                                   │
│  Camada 7: App Logic    → JWT expiration bug, signed URL bypass  │
│  Camada 6: WAF          → SQLi, XSS, bot, rate limit, payloads  │
│  Camada 5: CORS/Headers → HSTS, CSP, X-Frame, preflight          │
│  Camada 4: TLS/SSL      → Cipher suites, certificate, protocols  │
│  Camada 3: Origin       → Direct access bypass, hotlinking       │
│  Camada 2: Access       → OAC, Signed URLs, Geo restriction      │
│  Camada 1: Network      → DDoS, Shield, rate limiting            │
│                                                                   │
│  Cada camada testada = uma superfície de ataque validada         │
└──────────────────────────────────────────────────────────────────┘
```

### Arquitetura do Lab

```
                        Atacante (você simulando)
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
     curl + payloads      hey (load test)     browser (app.html)
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────────────────────────────────────────┐
    │  WAF Web ACL                                     │
    │  ├── AWSManagedRulesCommonRuleSet               │
    │  ├── AWSManagedRulesSQLiRuleSet                 │
    │  ├── AWSManagedRulesKnownBadInputsRuleSet       │
    │  ├── Rate Limit (100 req/5min)                  │
    │  ├── Geo Block (RU, CN, KP)                     │
    │  ├── Bad User-Agent Regex                       │
    │  └── Custom: Block /admin/* sem IP whitelist    │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │  CloudFront Distribution                         │
    │  ├── Security Headers (HSTS, CSP, X-Frame)      │
    │  ├── TLS 1.2+ (TLSv1.2_2021)                   │
    │  ├── Signed URLs para /premium/*                │
    │  ├── /api/* → ALB (com custom header protection)│
    │  └── Default → S3 (OAC)                         │
    └─────────────────┬──────────┬────────────────────┘
                      │          │
               ┌──────┘          └──────┐
               ▼                        ▼
         ┌──────────┐            ┌──────────┐
         │   S3     │            │   ALB    │
         │  (OAC)   │            │(Custom H)│
         │ Frontend │            │   API    │
         └──────────┘            └──────────┘
```

---

### Parte 1: App HTML para Testes de Segurança no Browser

```html
<!-- security-test-app.html — Upload para S3 como index.html -->
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <title>CloudFront Security Assessment Lab</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Courier New', monospace; background: #0d1117; color: #c9d1d9; padding: 20px; }
    h1 { color: #58a6ff; margin-bottom: 5px; }
    h2 { color: #f0883e; margin: 20px 0 10px; border-bottom: 1px solid #21262d; padding-bottom: 5px; }
    .subtitle { color: #8b949e; margin-bottom: 20px; }
    .test-group { background: #161b22; border: 1px solid #21262d; border-radius: 8px; padding: 15px; margin: 10px 0; }
    .result { padding: 8px 12px; margin: 5px 0; border-radius: 4px; font-size: 0.9em; }
    .pass { background: #0d2818; border-left: 4px solid #2ea043; }
    .fail { background: #2d1117; border-left: 4px solid #f85149; }
    .warn { background: #2d2000; border-left: 4px solid #d29922; }
    .info { background: #0d1d30; border-left: 4px solid #58a6ff; }
    button { background: #238636; color: #fff; border: none; padding: 10px 20px;
             cursor: pointer; border-radius: 6px; margin: 5px; font-family: inherit; }
    button:hover { background: #2ea043; }
    button.danger { background: #da3633; }
    button.danger:hover { background: #f85149; }
    pre { background: #0d1117; border: 1px solid #21262d; padding: 10px;
          border-radius: 4px; overflow-x: auto; font-size: 0.85em; margin: 5px 0; }
    .header-table { width: 100%; border-collapse: collapse; margin: 10px 0; }
    .header-table td { padding: 4px 8px; border: 1px solid #21262d; font-size: 0.85em; }
    .header-table td:first-child { color: #58a6ff; width: 40%; }
    .score { font-size: 2em; font-weight: bold; margin: 10px 0; }
    .score.a { color: #2ea043; }
    .score.b { color: #d29922; }
    .score.c { color: #f0883e; }
    .score.f { color: #f85149; }
    #results { margin-top: 15px; }
    .flex { display: flex; gap: 10px; flex-wrap: wrap; }
  </style>
</head>
<body>
  <h1>CloudFront Security Assessment Lab</h1>
  <p class="subtitle">Simule ataques e valide cada camada de segurança da sua distribution.</p>

  <div class="flex">
    <button onclick="runFullAssessment()">Rodar Assessment Completo</button>
    <button onclick="clearResults()">Limpar</button>
  </div>

  <h2>1. Security Headers</h2>
  <div class="test-group">
    <button onclick="testSecurityHeaders()">Testar Headers</button>
    <p style="color:#8b949e;margin-top:5px;">Valida HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy</p>
  </div>

  <h2>2. TLS/SSL</h2>
  <div class="test-group">
    <button onclick="testTLS()">Verificar TLS</button>
    <p style="color:#8b949e;margin-top:5px;">Verifica se a conexão usa TLS 1.2+ e protocolo seguro</p>
  </div>

  <h2>3. WAF — Injeções</h2>
  <div class="test-group">
    <div class="flex">
      <button class="danger" onclick="testSQLi()">Testar SQLi</button>
      <button class="danger" onclick="testXSS()">Testar XSS</button>
      <button class="danger" onclick="testPathTraversal()">Testar Path Traversal</button>
      <button class="danger" onclick="testCommandInjection()">Testar Command Injection</button>
    </div>
    <p style="color:#8b949e;margin-top:5px;">Envia payloads maliciosos — WAF deve bloquear (403)</p>
  </div>

  <h2>4. Origin Protection</h2>
  <div class="test-group">
    <button onclick="testOriginDirect()">Testar Acesso Direto ao S3</button>
    <button onclick="testHotlinking()">Testar Hotlinking</button>
    <p style="color:#8b949e;margin-top:5px;">Tenta acessar o origin diretamente (sem CloudFront) — deve falhar</p>
  </div>

  <h2>5. Signed URLs</h2>
  <div class="test-group">
    <button onclick="testSignedURLExpired()">Testar URL Expirada</button>
    <button onclick="testUnsignedAccess()">Testar Sem Assinatura</button>
    <p style="color:#8b949e;margin-top:5px;">Testa acesso a conteúdo protegido sem credenciais válidas</p>
  </div>

  <h2>6. CORS</h2>
  <div class="test-group">
    <button onclick="testCORSValid()">CORS — Origin Permitida</button>
    <button onclick="testCORSInvalid()">CORS — Origin Bloqueada</button>
    <p style="color:#8b949e;margin-top:5px;">Valida que CORS permite origins legítimas e bloqueia outros</p>
  </div>

  <h2>7. Rate Limiting</h2>
  <div class="test-group">
    <button class="danger" onclick="testRateLimit()">Disparar 50 Requests Rápidos</button>
    <p style="color:#8b949e;margin-top:5px;">Envia burst de requests — WAF deve começar a bloquear (429/403)</p>
  </div>

  <div id="results"></div>

  <script>
    const DOMAIN = window.location.origin;
    const resultsDiv = document.getElementById('results');

    function log(msg, status = 'info') {
      const el = document.createElement('div');
      el.className = `result ${status}`;
      el.innerHTML = msg;
      resultsDiv.appendChild(el);
      el.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
    }

    function clearResults() { resultsDiv.innerHTML = ''; }

    // ===== 1. SECURITY HEADERS =====
    async function testSecurityHeaders() {
      log('<b>--- Security Headers Assessment ---</b>');
      try {
        const r = await fetch(DOMAIN + '/', { method: 'HEAD' });
        const headers = Object.fromEntries(r.headers.entries());

        const checks = [
          { name: 'strict-transport-security', expected: 'max-age=', desc: 'HSTS', critical: true },
          { name: 'x-frame-options', expected: 'DENY', desc: 'Clickjack Protection', critical: true },
          { name: 'x-content-type-options', expected: 'nosniff', desc: 'MIME Sniffing', critical: true },
          { name: 'content-security-policy', expected: null, desc: 'CSP', critical: true },
          { name: 'referrer-policy', expected: null, desc: 'Referrer Policy', critical: false },
          { name: 'permissions-policy', expected: null, desc: 'Permissions Policy', critical: false },
          { name: 'x-xss-protection', expected: '1', desc: 'XSS Filter (legacy)', critical: false },
        ];

        let score = 0;
        let total = checks.length;

        checks.forEach(check => {
          const val = headers[check.name];
          const found = !!val;
          const matches = found && (check.expected === null || val.includes(check.expected));

          if (matches) {
            score++;
            log(`✅ <b>${check.desc}</b>: <code>${check.name}: ${val}</code>`, 'pass');
          } else if (found) {
            score += 0.5;
            log(`⚠️ <b>${check.desc}</b>: presente mas valor inesperado: <code>${val}</code>`, 'warn');
          } else {
            log(`❌ <b>${check.desc}</b>: <code>${check.name}</code> AUSENTE ${check.critical ? '(CRÍTICO!)' : ''}`, 'fail');
          }
        });

        const pct = Math.round((score / total) * 100);
        const grade = pct >= 90 ? 'A' : pct >= 70 ? 'B' : pct >= 50 ? 'C' : 'F';
        log(`<div class="score ${grade.toLowerCase()}">Score: ${grade} (${pct}%)</div>
             <b>Validação externa:</b><br>
             → <a href="https://securityheaders.com/?q=${DOMAIN}" target="_blank" style="color:#58a6ff">securityheaders.com</a><br>
             → <a href="https://observatory.mozilla.org/analyze/${window.location.hostname}" target="_blank" style="color:#58a6ff">Mozilla Observatory</a><br>
             → <a href="https://www.ssllabs.com/ssltest/analyze.html?d=${window.location.hostname}" target="_blank" style="color:#58a6ff">SSL Labs</a>`,
          grade === 'A' ? 'pass' : grade === 'F' ? 'fail' : 'warn');
      } catch (e) {
        log(`Erro: ${e.message}`, 'fail');
      }
    }

    // ===== 2. TLS/SSL =====
    async function testTLS() {
      log('<b>--- TLS/SSL Assessment ---</b>');

      // Verificar protocolo atual
      const protocol = window.location.protocol;
      if (protocol === 'https:') {
        log('✅ <b>HTTPS ativo</b> — conexão criptografada', 'pass');
      } else {
        log('❌ <b>HTTP!</b> Conexão NÃO criptografada. Habilite redirect-to-https.', 'fail');
      }

      // Testar redirect HTTP → HTTPS
      try {
        const httpUrl = DOMAIN.replace('https://', 'http://');
        log(`ℹ️ Para testar redirect HTTP→HTTPS e cipher suites, use:<pre>
# Testar redirect
curl -s -o /dev/null -w "%{http_code} → %{redirect_url}" http://${window.location.hostname}/

# Testar TLS (deve rejeitar TLS 1.0/1.1)
openssl s_client -connect ${window.location.hostname}:443 -tls1 2>&1 | grep -i "alert\\|error"
openssl s_client -connect ${window.location.hostname}:443 -tls1_1 2>&1 | grep -i "alert\\|error"

# Deve funcionar com TLS 1.2+
openssl s_client -connect ${window.location.hostname}:443 -tls1_2 2>&1 | head -5

# SSL Labs (mais completo)
# https://www.ssllabs.com/ssltest/analyze.html?d=${window.location.hostname}

# Verificar cipher suites
nmap --script ssl-enum-ciphers -p 443 ${window.location.hostname}</pre>`, 'info');
      } catch (e) {
        log(`Info: ${e.message}`, 'warn');
      }
    }

    // ===== 3. WAF — INJECTION TESTS =====
    async function testSQLi() {
      log('<b>--- WAF: SQL Injection Test ---</b>');
      const payloads = [
        { name: "Basic OR", url: "/api/users?id=1' OR '1'='1" },
        { name: "UNION SELECT", url: "/api/users?id=1 UNION SELECT username,password FROM users--" },
        { name: "DROP TABLE", url: "/api/users?id=1; DROP TABLE users;--" },
        { name: "Blind SQLi", url: "/api/users?id=1 AND 1=1" },
        { name: "Time-based", url: "/api/users?id=1' AND SLEEP(5)--" },
      ];
      for (const p of payloads) {
        try {
          const r = await fetch(DOMAIN + p.url);
          if (r.status === 403) {
            log(`✅ <b>${p.name}</b> BLOQUEADO (403) — <code>${p.url}</code>`, 'pass');
          } else {
            log(`❌ <b>${p.name}</b> NÃO bloqueado (${r.status}) — WAF não detectou!<br><code>${p.url}</code>`, 'fail');
          }
        } catch (e) {
          log(`⚠️ <b>${p.name}</b> — Erro de rede (possível WAF block): ${e.message}`, 'warn');
        }
      }
    }

    async function testXSS() {
      log('<b>--- WAF: XSS Test ---</b>');
      const payloads = [
        { name: "Script tag", url: "/search?q=<script>alert('xss')</script>" },
        { name: "Event handler", url: "/search?q=<img src=x onerror=alert(1)>" },
        { name: "SVG onload", url: "/search?q=<svg onload=alert(1)>" },
        { name: "JavaScript URI", url: "/search?q=javascript:alert(document.cookie)" },
        { name: "Encoded", url: "/search?q=%3Cscript%3Ealert(1)%3C/script%3E" },
      ];
      for (const p of payloads) {
        try {
          const r = await fetch(DOMAIN + p.url);
          if (r.status === 403) {
            log(`✅ <b>${p.name}</b> BLOQUEADO (403)`, 'pass');
          } else {
            log(`❌ <b>${p.name}</b> NÃO bloqueado (${r.status})`, 'fail');
          }
        } catch (e) {
          log(`⚠️ <b>${p.name}</b> — Erro: ${e.message}`, 'warn');
        }
      }
    }

    async function testPathTraversal() {
      log('<b>--- WAF: Path Traversal Test ---</b>');
      const payloads = [
        { name: "Basic traversal", url: "/../../../../etc/passwd" },
        { name: "Encoded traversal", url: "/%2e%2e/%2e%2e/%2e%2e/etc/passwd" },
        { name: "Double encoded", url: "/%252e%252e/%252e%252e/etc/passwd" },
        { name: "Null byte", url: "/file%00.jpg" },
      ];
      for (const p of payloads) {
        try {
          const r = await fetch(DOMAIN + p.url);
          if (r.status === 403 || r.status === 400) {
            log(`✅ <b>${p.name}</b> BLOQUEADO (${r.status})`, 'pass');
          } else {
            log(`❌ <b>${p.name}</b> NÃO bloqueado (${r.status})`, 'fail');
          }
        } catch (e) {
          log(`⚠️ <b>${p.name}</b> — Erro: ${e.message}`, 'warn');
        }
      }
    }

    async function testCommandInjection() {
      log('<b>--- WAF: Command Injection Test ---</b>');
      const payloads = [
        { name: "Pipe command", url: "/api/ping?host=google.com|cat /etc/passwd" },
        { name: "Semicolon", url: "/api/ping?host=google.com;ls -la" },
        { name: "Backtick", url: "/api/ping?host=`whoami`" },
        { name: "$(command)", url: "/api/ping?host=$(cat /etc/passwd)" },
      ];
      for (const p of payloads) {
        try {
          const r = await fetch(DOMAIN + p.url);
          if (r.status === 403) {
            log(`✅ <b>${p.name}</b> BLOQUEADO (403)`, 'pass');
          } else {
            log(`❌ <b>${p.name}</b> NÃO bloqueado (${r.status})`, 'fail');
          }
        } catch (e) {
          log(`⚠️ <b>${p.name}</b> — Erro: ${e.message}`, 'warn');
        }
      }
    }

    // ===== 4. ORIGIN PROTECTION =====
    async function testOriginDirect() {
      log('<b>--- Origin Protection: Acesso Direto ao S3 ---</b>');
      log(`ℹ️ Para testar acesso direto ao origin, use curl no terminal:<pre>
# Tentar acessar S3 diretamente (deve retornar 403 Forbidden)
curl -s -o /dev/null -w "Status: %{http_code}" \\
  https://SEU-BUCKET.s3.us-east-1.amazonaws.com/index.html

# Se retornar 200 → BUG! Bucket está público.
# Se retornar 403 → OK! OAC está funcionando.

# Tentar acessar ALB diretamente (deve bloquear sem custom header)
curl -s -o /dev/null -w "Status: %{http_code}" \\
  https://SEU-ALB.us-east-1.elb.amazonaws.com/api/health

# Se retornar 200 → BUG! ALB não valida custom header.
# Se retornar 403 → OK! Origin protection funciona.

# Verificar que CloudFront envia custom header
curl -s -I https://${window.location.hostname}/api/health | grep -i "x-cache"
# Se x-cache: Hit → OK, CloudFront está servindo</pre>`, 'info');
    }

    async function testHotlinking() {
      log('<b>--- Origin Protection: Hotlinking Test ---</b>');
      log(`ℹ️ Hotlinking = outro site usa suas imagens/vídeos diretamente.
Teste no terminal:<pre>
# Simular request com Referer de outro site (hotlinking)
curl -s -o /dev/null -w "Status: %{http_code}" \\
  -H "Referer: https://site-pirata.com/artigo" \\
  https://${window.location.hostname}/images/produto.jpg

# Simular request sem Referer (acesso direto)
curl -s -o /dev/null -w "Status: %{http_code}" \\
  https://${window.location.hostname}/images/produto.jpg

# Para BLOQUEAR hotlinking, use CloudFront Function:
# function handler(event) {
#   var request = event.request;
#   var referer = request.headers['referer'];
#   var allowedDomains = ['meusite.com', 'www.meusite.com'];
#   
#   // Permitir acesso direto (sem referer) e bots de SEO
#   if (!referer) return request;
#   
#   var refererDomain = referer.value.split('/')[2];
#   var allowed = allowedDomains.some(d => refererDomain.endsWith(d));
#   
#   if (!allowed) {
#     return {
#       statusCode: 403,
#       statusDescription: 'Forbidden',
#       body: { encoding: 'text', data: 'Hotlinking not allowed' }
#     };
#   }
#   return request;
# }</pre>`, 'info');
    }

    // ===== 5. SIGNED URLs =====
    async function testSignedURLExpired() {
      log('<b>--- Signed URL: Expiração ---</b>');
      try {
        // Simular URL com Policy expirada (o Expires está no passado)
        const expiredUrl = DOMAIN + '/premium/video.mp4?Expires=1609459200&Signature=FAKE&Key-Pair-Id=K1EXAMPLE';
        const r = await fetch(expiredUrl);
        if (r.status === 403) {
          log('✅ <b>URL expirada BLOQUEADA</b> (403) — CloudFront rejeitou corretamente', 'pass');
        } else {
          log(`❌ <b>URL expirada retornou ${r.status}</b> — deveria ser 403!`, 'fail');
        }
      } catch (e) {
        log(`⚠️ Erro: ${e.message}`, 'warn');
      }

      // Testar sem assinatura
      try {
        const unsignedUrl = DOMAIN + '/premium/video.mp4';
        const r = await fetch(unsignedUrl);
        if (r.status === 403) {
          log('✅ <b>Acesso sem assinatura BLOQUEADO</b> (403) — Signed URL obrigatória', 'pass');
        } else if (r.status === 404) {
          log('⚠️ Retornou 404 — path /premium/* pode não existir neste lab', 'warn');
        } else {
          log(`❌ <b>Acesso sem assinatura retornou ${r.status}</b> — deveria ser 403!`, 'fail');
        }
      } catch (e) {
        log(`⚠️ Erro: ${e.message}`, 'warn');
      }
    }

    async function testUnsignedAccess() {
      log('<b>--- Signed URL: Acesso Sem Credenciais ---</b>');
      log(`ℹ️ Teste completo no terminal:<pre>
# 1. Acessar conteúdo protegido SEM assinatura (deve falhar)
curl -s -o /dev/null -w "Status: %{http_code}" \\
  https://${window.location.hostname}/premium/video.mp4
# Esperado: 403

# 2. Gerar signed URL válida
PRIVATE_KEY_FILE="private_key.pem"
KEY_PAIR_ID="K1EXAMPLE"
EXPIRES=\$(date -d "+1 hour" +%s)
RESOURCE="https://${window.location.hostname}/premium/*"

# AWS CLI
aws cloudfront sign --url "https://${window.location.hostname}/premium/video.mp4" \\
  --key-pair-id \$KEY_PAIR_ID \\
  --private-key file://\$PRIVATE_KEY_FILE \\
  --date-less-than \$EXPIRES

# 3. Acessar com signed URL (deve funcionar)
curl -s -o /dev/null -w "Status: %{http_code}" \\
  "SIGNED_URL_AQUI"
# Esperado: 200

# 4. BUG SILENCIOSO: Token expirado mas cacheado
# Se o CloudFront cacheou a resposta com TTL > 0,
# mesmo após a URL expirar, o cache ainda serve o conteúdo!
# Para evitar: NÃO inclua parâmetros de assinatura no cache key
# Use CachingDisabled ou cache key SEM query strings</pre>`, 'info');
    }

    // ===== 6. CORS =====
    async function testCORSValid() {
      log('<b>--- CORS: Origin Permitida ---</b>');
      try {
        const r = await fetch(DOMAIN + '/api/data', {
          method: 'OPTIONS',
          headers: {
            'Origin': window.location.origin,
            'Access-Control-Request-Method': 'GET'
          }
        });
        const acao = r.headers.get('access-control-allow-origin');
        if (acao) {
          log(`✅ <b>CORS permitido</b> para ${window.location.origin}<br>
            access-control-allow-origin: ${acao}`, 'pass');
        } else {
          log(`⚠️ CORS header ausente para origin própria (pode não ter CORS configurado)`, 'warn');
        }
      } catch (e) {
        log(`⚠️ Erro: ${e.message}`, 'warn');
      }
    }

    async function testCORSInvalid() {
      log('<b>--- CORS: Origin Bloqueada ---</b>');
      log(`ℹ️ O browser impede enviar Origin forjado via fetch. Teste via curl:<pre>
# Origin legítima (deve incluir CORS headers)
curl -s -X OPTIONS \\
  -H "Origin: https://meusite.com" \\
  -H "Access-Control-Request-Method: POST" \\
  -I https://${window.location.hostname}/api/data | grep -i access-control

# Origin maliciosa (NÃO deve incluir CORS headers)
curl -s -X OPTIONS \\
  -H "Origin: https://evil-hacker.com" \\
  -H "Access-Control-Request-Method: POST" \\
  -I https://${window.location.hostname}/api/data | grep -i access-control

# Se access-control-allow-origin: * → RISCO!
# Qualquer site pode fazer requests cross-origin.
# Use origins específicas na Response Headers Policy.</pre>`, 'info');
    }

    // ===== 7. RATE LIMITING =====
    async function testRateLimit() {
      log('<b>--- Rate Limiting: Burst Test (50 requests) ---</b>');
      log('Disparando 50 requests em sequência rápida...', 'info');

      let blocked = 0;
      let success = 0;
      let errors = 0;

      for (let i = 0; i < 50; i++) {
        try {
          const r = await fetch(DOMAIN + '/api/data?burst=' + i, { cache: 'no-store' });
          if (r.status === 403 || r.status === 429) {
            blocked++;
          } else {
            success++;
          }
        } catch (e) {
          errors++;
        }
      }

      if (blocked > 0) {
        log(`✅ <b>Rate limiting ativo!</b><br>
          Sucesso: ${success} | Bloqueados: ${blocked} | Erros: ${errors}<br>
          WAF começou a bloquear após ${success} requests.`, 'pass');
      } else {
        log(`⚠️ <b>Nenhum request bloqueado</b> (${success} OK, ${errors} erros)<br>
          Possíveis causas:<br>
          - Rate limit configurado > 50 req/5min<br>
          - Rate limit usa IP-based e o browser enviou poucos<br>
          - WAF não está ativo nesta distribution<br><br>
          Para teste mais agressivo, use no terminal:<pre>
# Instalar hey (load testing tool)
# https://github.com/rakyll/hey
hey -n 500 -c 50 https://${window.location.hostname}/api/data

# Ou com Apache Bench
ab -n 500 -c 50 https://${window.location.hostname}/api/data

# Ou com k6
# k6 run --vus 50 --duration 30s script.js</pre>`, 'warn');
      }
    }

    // ===== FULL ASSESSMENT =====
    async function runFullAssessment() {
      clearResults();
      log('<h2 style="color:#58a6ff">Security Assessment Completo</h2>');
      log(`Início: ${new Date().toLocaleString()} | Target: ${DOMAIN}`, 'info');

      await testSecurityHeaders();
      await testTLS();
      await testSQLi();
      await testXSS();
      await testPathTraversal();
      await testCommandInjection();
      await testOriginDirect();
      await testHotlinking();
      await testSignedURLExpired();
      await testCORSValid();
      await testRateLimit();

      log(`<h2 style="color:#58a6ff">Assessment Completo</h2>
        Fim: ${new Date().toLocaleString()}<br>
        Revise os resultados acima. Items ❌ requerem ação imediata.`, 'info');
    }
  </script>
</body>
</html>
```

```bash
# Upload do app de teste para S3
aws s3 cp /tmp/security-test-app.html "s3://$BUCKET/security-lab.html" \
  --content-type "text/html"

echo "Acesse: https://$DOMAIN/security-lab.html"
```

---

### Parte 2: Testes de Segurança via CLI — Payloads Reais

#### 2.1 — OWASP Top 10 Mapping para CloudFront

```
┌──────────────────────────────────────────────────────────────────────┐
│              OWASP Top 10 (2021) × CloudFront Defenses               │
│                                                                       │
│  A01: Broken Access Control                                          │
│  ├── Defesa: Signed URLs/Cookies (D.23-24)                          │
│  ├── Defesa: WAF IP whitelist para /admin/* (D.25)                  │
│  ├── Defesa: OAC bloqueia acesso direto ao S3 (D.29)               │
│  └── Teste: curl sem assinatura, curl direto ao S3                  │
│                                                                       │
│  A02: Cryptographic Failures                                         │
│  ├── Defesa: TLS 1.2+ obrigatório (TLSv1.2_2021)                   │
│  ├── Defesa: HSTS com preload (D.14, Mod03)                        │
│  ├── Defesa: Field-Level Encryption (D.28)                          │
│  └── Teste: openssl s_client, SSL Labs scan                        │
│                                                                       │
│  A03: Injection (SQLi, XSS, Command)                                │
│  ├── Defesa: WAF SQLiRuleSet + CommonRuleSet (D.25)                │
│  ├── Defesa: CSP header bloqueia inline scripts (D.14)              │
│  └── Teste: payloads SQLi/XSS via curl (abaixo)                    │
│                                                                       │
│  A04: Insecure Design                                                │
│  ├── Defesa: Behavior ordering correto (Mod02 Bônus)               │
│  ├── Defesa: Cache key não inclui dados sensíveis                   │
│  └── Teste: verificar que tokens não vazam no cache                 │
│                                                                       │
│  A05: Security Misconfiguration                                      │
│  ├── Defesa: Security headers (X-Frame, X-Content-Type) (D.14)     │
│  ├── Defesa: Remover Server header (CloudFront faz por default)     │
│  └── Teste: securityheaders.com, Mozilla Observatory                │
│                                                                       │
│  A06: Vulnerable & Outdated Components                               │
│  ├── Defesa: CloudFront Functions runtime atualizado (cloudfront-   │
│  │   js-2.0), Lambda@Edge com runtime recente                       │
│  └── Teste: verificar runtime versions                              │
│                                                                       │
│  A07: Identification & Authentication Failures                       │
│  ├── Defesa: JWT validation via Lambda@Edge (D.18, Mod04)           │
│  ├── Defesa: mTLS para APIs B2B (D.30)                             │
│  ├── Defesa: Signed URLs com expiração curta (D.23)                │
│  └── Teste: tokens expirados, certificados revogados               │
│                                                                       │
│  A08: Software & Data Integrity Failures                             │
│  ├── Defesa: SRI (Subresource Integrity) via CSP                   │
│  ├── Defesa: Cache invalidation controlada                          │
│  └── Teste: verificar que assets não foram adulterados              │
│                                                                       │
│  A09: Security Logging & Monitoring Failures                         │
│  ├── Defesa: Standard Logs + Athena (D.39, Mod07)                  │
│  ├── Defesa: Real-Time Logs + alarmes (D.40-42, Mod07)             │
│  └── Teste: verificar que logs capturam ataques                    │
│                                                                       │
│  A10: Server-Side Request Forgery (SSRF)                            │
│  ├── Defesa: WAF KnownBadInputsRuleSet (D.25)                     │
│  ├── Defesa: Origin Request Policy não repassa Host arbitrário     │
│  └── Teste: curl com Host header manipulado                        │
└──────────────────────────────────────────────────────────────────────┘
```

#### 2.2 — Script de Security Assessment via CLI

```bash
#!/bin/bash
# security-assessment.sh — Assessment de segurança completo via CLI
# Uso: ./security-assessment.sh <cloudfront-domain>

set -euo pipefail

DOMAIN="${1:?Uso: $0 <cloudfront-domain> (ex: d111222333.cloudfront.net)}"
PASS=0
FAIL=0
WARN=0
REPORT="/tmp/cf-security-report-$(date +%Y%m%d-%H%M%S).txt"

echo "============================================" | tee "$REPORT"
echo "  CloudFront Security Assessment" | tee -a "$REPORT"
echo "  Target: $DOMAIN" | tee -a "$REPORT"
echo "  Date: $(date)" | tee -a "$REPORT"
echo "============================================" | tee -a "$REPORT"

pass() { echo "  ✅ PASS: $1" | tee -a "$REPORT"; ((PASS++)); }
fail() { echo "  ❌ FAIL: $1" | tee -a "$REPORT"; ((FAIL++)); }
warn() { echo "  ⚠️  WARN: $1" | tee -a "$REPORT"; ((WARN++)); }
info() { echo "  ℹ️  INFO: $1" | tee -a "$REPORT"; }

# ==========================================
# 1. TLS/SSL
# ==========================================
echo "" | tee -a "$REPORT"
echo "--- [1/7] TLS/SSL ---" | tee -a "$REPORT"

# HTTPS redirect
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -m 10 "http://$DOMAIN/" 2>/dev/null || echo "000")
if [ "$HTTP_CODE" = "301" ] || [ "$HTTP_CODE" = "302" ]; then
  pass "HTTP→HTTPS redirect ativo ($HTTP_CODE)"
else
  fail "HTTP→HTTPS redirect NÃO ativo (status: $HTTP_CODE)"
fi

# TLS 1.0 (deve falhar)
TLS10=$(echo | timeout 5 openssl s_client -connect "$DOMAIN:443" -tls1 2>&1 || true)
if echo "$TLS10" | grep -qi "alert\|error\|wrong version\|no protocols"; then
  pass "TLS 1.0 REJEITADO (correto)"
else
  fail "TLS 1.0 ACEITO — desabilite protocolos antigos!"
fi

# TLS 1.1 (deve falhar)
TLS11=$(echo | timeout 5 openssl s_client -connect "$DOMAIN:443" -tls1_1 2>&1 || true)
if echo "$TLS11" | grep -qi "alert\|error\|wrong version\|no protocols"; then
  pass "TLS 1.1 REJEITADO (correto)"
else
  fail "TLS 1.1 ACEITO — desabilite protocolos antigos!"
fi

# TLS 1.2 (deve funcionar)
TLS12=$(echo | timeout 5 openssl s_client -connect "$DOMAIN:443" -tls1_2 2>&1 || true)
if echo "$TLS12" | grep -q "Verify return code: 0"; then
  pass "TLS 1.2 funcional"
elif echo "$TLS12" | grep -qi "CONNECTED"; then
  pass "TLS 1.2 funcional (connected)"
else
  fail "TLS 1.2 NÃO funcional"
fi

# ==========================================
# 2. Security Headers
# ==========================================
echo "" | tee -a "$REPORT"
echo "--- [2/7] Security Headers ---" | tee -a "$REPORT"

HEADERS=$(curl -s -I -m 10 "https://$DOMAIN/" 2>/dev/null)

check_header() {
  local name="$1"
  local critical="${2:-false}"
  if echo "$HEADERS" | grep -qi "^$name:"; then
    local value=$(echo "$HEADERS" | grep -i "^$name:" | head -1 | cut -d: -f2- | xargs)
    pass "$name: $value"
  else
    if [ "$critical" = "true" ]; then
      fail "$name AUSENTE (CRÍTICO)"
    else
      warn "$name ausente (recomendado)"
    fi
  fi
}

check_header "strict-transport-security" "true"
check_header "x-frame-options" "true"
check_header "x-content-type-options" "true"
check_header "content-security-policy" "true"
check_header "referrer-policy" "false"
check_header "permissions-policy" "false"

# Verificar se Server header está exposto
if echo "$HEADERS" | grep -qi "^server: Apache\|^server: nginx\|^server: IIS"; then
  fail "Server header expõe tecnologia do backend"
elif echo "$HEADERS" | grep -qi "^server: AmazonS3"; then
  warn "Server: AmazonS3 — considere remover via Response Headers Policy"
else
  pass "Server header não expõe tecnologia"
fi

# ==========================================
# 3. WAF — Injection Tests
# ==========================================
echo "" | tee -a "$REPORT"
echo "--- [3/7] WAF Injection Tests ---" | tee -a "$REPORT"

test_waf() {
  local name="$1"
  local url="$2"
  local code=$(curl -s -o /dev/null -w "%{http_code}" -m 10 "https://$DOMAIN$url" 2>/dev/null || echo "000")
  if [ "$code" = "403" ]; then
    pass "WAF bloqueou $name (403)"
  elif [ "$code" = "000" ]; then
    warn "WAF $name — timeout/conexão recusada"
  else
    fail "WAF NÃO bloqueou $name (status: $code)"
  fi
}

# SQL Injection
test_waf "SQLi (OR 1=1)" "/api/test?id=1'+OR+'1'%3D'1"
test_waf "SQLi (UNION SELECT)" "/api/test?id=1+UNION+SELECT+username,password+FROM+users--"
test_waf "SQLi (DROP TABLE)" "/api/test?id=1;+DROP+TABLE+users;--"

# XSS
test_waf "XSS (script tag)" "/search?q=%3Cscript%3Ealert(1)%3C%2Fscript%3E"
test_waf "XSS (event handler)" "/search?q=%3Cimg+src%3Dx+onerror%3Dalert(1)%3E"

# Path Traversal
test_waf "Path Traversal" "/../../../../etc/passwd"

# Command Injection
test_waf "Command Injection" "/api/test?host=google.com%7Ccat+/etc/passwd"

# Known Bad Inputs
test_waf "User-Agent: sqlmap" "-H 'User-Agent: sqlmap/1.0' /"
# Para User-Agent, curl precisa de tratamento especial
UA_CODE=$(curl -s -o /dev/null -w "%{http_code}" -m 10 -H "User-Agent: sqlmap/1.0-dev" "https://$DOMAIN/" 2>/dev/null || echo "000")
if [ "$UA_CODE" = "403" ]; then
  pass "WAF bloqueou User-Agent malicioso (sqlmap)"
else
  warn "WAF não bloqueou User-Agent sqlmap (status: $UA_CODE)"
fi

# SSRF attempt
test_waf "SSRF (169.254)" "/api/proxy?url=http://169.254.169.254/latest/meta-data/"

# ==========================================
# 4. Origin Protection
# ==========================================
echo "" | tee -a "$REPORT"
echo "--- [4/7] Origin Protection ---" | tee -a "$REPORT"

# Extrair origin domain da distribution (se possível via CLI)
info "Testes manuais de origin protection:"
info "  - Acessar S3 bucket diretamente deve retornar 403"
info "  - Acessar ALB diretamente sem custom header deve retornar 403"
info "  Comandos:"
info "    curl -s -w '%{http_code}' https://BUCKET.s3.amazonaws.com/index.html"
info "    curl -s -w '%{http_code}' https://ALB.elb.amazonaws.com/api/health"

# Testar que CloudFront está servindo (via header)
if echo "$HEADERS" | grep -qi "x-cache"; then
  pass "CloudFront está servindo requests (x-cache header presente)"
else
  warn "x-cache header não encontrado"
fi

if echo "$HEADERS" | grep -qi "via.*cloudfront"; then
  pass "Via header confirma CloudFront"
else
  warn "Via header não confirma CloudFront"
fi

# ==========================================
# 5. Signed URL Validation
# ==========================================
echo "" | tee -a "$REPORT"
echo "--- [5/7] Signed URLs ---" | tee -a "$REPORT"

# Testar acesso sem assinatura a path protegido
UNSIGNED_CODE=$(curl -s -o /dev/null -w "%{http_code}" -m 10 "https://$DOMAIN/premium/test.mp4" 2>/dev/null || echo "000")
if [ "$UNSIGNED_CODE" = "403" ]; then
  pass "Acesso sem assinatura BLOQUEADO (403) em /premium/*"
elif [ "$UNSIGNED_CODE" = "404" ]; then
  info "/premium/* retornou 404 — path pode não existir (configure para testar)"
else
  warn "Acesso sem assinatura retornou $UNSIGNED_CODE (esperado 403 se configurado)"
fi

# Testar URL com assinatura expirada
EXPIRED_CODE=$(curl -s -o /dev/null -w "%{http_code}" -m 10 \
  "https://$DOMAIN/premium/test.mp4?Expires=1609459200&Signature=INVALID&Key-Pair-Id=KEXAMPLE" \
  2>/dev/null || echo "000")
if [ "$EXPIRED_CODE" = "403" ]; then
  pass "URL expirada/inválida BLOQUEADA (403)"
else
  warn "URL expirada retornou $EXPIRED_CODE"
fi

# ==========================================
# 6. CORS
# ==========================================
echo "" | tee -a "$REPORT"
echo "--- [6/7] CORS ---" | tee -a "$REPORT"

# Testar com origin legítima
CORS_VALID=$(curl -s -I -m 10 -X OPTIONS \
  -H "Origin: https://$DOMAIN" \
  -H "Access-Control-Request-Method: GET" \
  "https://$DOMAIN/api/data" 2>/dev/null || true)

if echo "$CORS_VALID" | grep -qi "access-control-allow-origin"; then
  pass "CORS headers presentes para origin legítima"
else
  info "CORS headers ausentes (pode não estar configurado neste path)"
fi

# Testar com origin maliciosa
CORS_EVIL=$(curl -s -I -m 10 -X OPTIONS \
  -H "Origin: https://evil-hacker.com" \
  -H "Access-Control-Request-Method: POST" \
  "https://$DOMAIN/api/data" 2>/dev/null || true)

if echo "$CORS_EVIL" | grep -qi "access-control-allow-origin: \*"; then
  fail "CORS permite qualquer origin (wildcard *) — RISCO!"
elif echo "$CORS_EVIL" | grep -qi "access-control-allow-origin: https://evil"; then
  fail "CORS permite origin maliciosa!"
else
  pass "CORS não permite origin maliciosa"
fi

# ==========================================
# 7. Rate Limiting
# ==========================================
echo "" | tee -a "$REPORT"
echo "--- [7/7] Rate Limiting ---" | tee -a "$REPORT"

info "Disparando 30 requests rápidos..."
BLOCKED=0
SUCCESS=0
for i in $(seq 1 30); do
  CODE=$(curl -s -o /dev/null -w "%{http_code}" -m 5 "https://$DOMAIN/?rl=$i" 2>/dev/null || echo "000")
  if [ "$CODE" = "403" ] || [ "$CODE" = "429" ]; then
    ((BLOCKED++))
  elif [ "$CODE" = "200" ] || [ "$CODE" = "304" ]; then
    ((SUCCESS++))
  fi
done

if [ "$BLOCKED" -gt 0 ]; then
  pass "Rate limiting ativo! Bloqueou $BLOCKED de 30 requests"
else
  warn "Nenhum request bloqueado ($SUCCESS OK). Rate limit pode ser > 30 req/5min."
  info "Para teste completo, instale 'hey' e execute:"
  info "  hey -n 500 -c 50 https://$DOMAIN/"
fi

# ==========================================
# RELATÓRIO FINAL
# ==========================================
echo "" | tee -a "$REPORT"
echo "============================================" | tee -a "$REPORT"
echo "  RESULTADO FINAL" | tee -a "$REPORT"
echo "============================================" | tee -a "$REPORT"
TOTAL=$((PASS + FAIL + WARN))
echo "  ✅ PASS: $PASS" | tee -a "$REPORT"
echo "  ❌ FAIL: $FAIL" | tee -a "$REPORT"
echo "  ⚠️  WARN: $WARN" | tee -a "$REPORT"
echo "  TOTAL: $TOTAL testes" | tee -a "$REPORT"
echo "" | tee -a "$REPORT"

if [ "$FAIL" -eq 0 ]; then
  echo "  🏆 SCORE: A — Excelente! Nenhuma falha crítica." | tee -a "$REPORT"
elif [ "$FAIL" -le 2 ]; then
  echo "  📊 SCORE: B — Bom, mas ${FAIL} item(s) precisam de atenção." | tee -a "$REPORT"
elif [ "$FAIL" -le 5 ]; then
  echo "  📊 SCORE: C — Razoável. ${FAIL} falhas encontradas." | tee -a "$REPORT"
else
  echo "  🚨 SCORE: F — Crítico! ${FAIL} falhas de segurança." | tee -a "$REPORT"
fi

echo "" | tee -a "$REPORT"
echo "  Relatório salvo em: $REPORT" | tee -a "$REPORT"
echo "" | tee -a "$REPORT"
echo "  Validação externa recomendada:" | tee -a "$REPORT"
echo "  → https://securityheaders.com/?q=https://$DOMAIN" | tee -a "$REPORT"
echo "  → https://observatory.mozilla.org/analyze/$DOMAIN" | tee -a "$REPORT"
echo "  → https://www.ssllabs.com/ssltest/analyze.html?d=$DOMAIN" | tee -a "$REPORT"
```

---

### Parte 3: Bug Silencioso — Token Expirado Servido do Cache

Este é um dos bugs de segurança mais perigosos e menos conhecidos em CloudFront:

```
┌──────────────────────────────────────────────────────────────────┐
│      BUG: Signed URL Expirada Servida do Cache                    │
│                                                                   │
│  Timeline:                                                       │
│  T+0s    User gera signed URL (expira em 60s)                   │
│  T+5s    User faz request → CloudFront busca no origin → 200    │
│          CloudFront CACHEIA a resposta (TTL: 300s do behavior)   │
│  T+60s   Signed URL EXPIRA                                      │
│  T+120s  Outro user acessa a MESMA URL (expirada)              │
│          CloudFront serve DO CACHE → 200 ← BUG!                │
│          O conteúdo protegido foi servido com URL expirada!     │
│                                                                   │
│  Root Cause:                                                     │
│  CloudFront valida a assinatura apenas no PRIMEIRO request.     │
│  Após cachear, serve sem revalidar a assinatura.                │
│                                                                   │
│  Impacto:                                                        │
│  Alguém com uma URL expirada (ou compartilhada) acessa conteúdo │
│  premium até o cache TTL expirar.                                │
└──────────────────────────────────────────────────────────────────┘
```

#### Como Reproduzir

```bash
# 1. Gerar signed URL com expiração curta (60s)
SIGNED_URL=$(aws cloudfront sign \
  --url "https://$DOMAIN/premium/video.mp4" \
  --key-pair-id K1EXAMPLE \
  --private-key file://private_key.pem \
  --date-less-than "$(date -d '+60 seconds' +%Y-%m-%dT%H:%M:%S)")

# 2. Acessar — popula o cache
curl -s -o /dev/null -w "Status: %{http_code}, x-cache: %{x-cache}\n" "$SIGNED_URL"
# Status: 200, x-cache: Miss from cloudfront

# 3. Aguardar a URL expirar
sleep 65

# 4. Acessar novamente — cache ainda serve!
curl -s -o /dev/null -w "Status: %{http_code}\n" "$SIGNED_URL"
# Status: 200 ← BUG! URL expirada mas cache serviu

# 5. Acessar com URL DIFERENTE expirada para o mesmo path
EXPIRED_URL="https://$DOMAIN/premium/video.mp4?Expires=1609459200&Signature=DIFFERENT&Key-Pair-Id=K1EXAMPLE"
curl -s -o /dev/null -w "Status: %{http_code}\n" "$EXPIRED_URL"
# Status: 403 ← OK — assinatura diferente, cache key diferente
```

#### Soluções

```hcl
# SOLUÇÃO 1 (Recomendada): Excluir parâmetros de assinatura do cache key
# Assim, todos os requests para /premium/video.mp4 usam o mesmo cache entry
# MAS CloudFront valida assinatura em CADA request (cache key não muda)

resource "aws_cloudfront_cache_policy" "signed_content" {
  name        = "Signed-Content-Policy"
  comment     = "Cache key SEM parâmetros de assinatura"
  default_ttl = 86400
  max_ttl     = 604800
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true

    headers_config { header_behavior = "none" }
    cookies_config { cookie_behavior = "none" }

    query_strings_config {
      query_string_behavior = "none"
      # NENHUMA query string no cache key!
      # Expires, Signature, Key-Pair-Id ficam FORA do cache key
      # CloudFront valida assinatura antes de checar cache
    }
  }
}

# SOLUÇÃO 2: CachingDisabled para conteúdo signed
# Máxima segurança, mas sem benefício de cache
# Use quando o conteúdo muda frequentemente
data "aws_cloudfront_cache_policy" "disabled" {
  name = "Managed-CachingDisabled"
}

# SOLUÇÃO 3: TTL curto alinhado com expiração da signed URL
# Se signed URL expira em 1h, TTL do behavior = 5 min
# Janela de risco: máximo 5 minutos após expiração
resource "aws_cloudfront_cache_policy" "signed_short_ttl" {
  name        = "Signed-Short-TTL"
  default_ttl = 300   # 5 min
  max_ttl     = 300   # 5 min (forçar)
  min_ttl     = 60    # 1 min

  parameters_in_cache_key_and_forwarded_to_origin {
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true
    headers_config { header_behavior = "none" }
    cookies_config { cookie_behavior = "none" }
    query_strings_config { query_string_behavior = "none" }
  }
}
```

```
┌───────────────────────────────────────────────────────────────┐
│         Comparação das Soluções                                │
│                                                                │
│  Solução           │ Segurança │ Performance │ Custo Origin   │
│  ──────────────────┼───────────┼─────────────┼──────────────  │
│  QS fora do cache  │ ★★★★★    │ ★★★★★      │ Baixo          │
│  key (Solução 1)   │           │ (cache OK)  │ (cache hits)   │
│  ──────────────────┼───────────┼─────────────┼──────────────  │
│  CachingDisabled   │ ★★★★★    │ ★☆☆☆☆      │ Alto           │
│  (Solução 2)       │ (perfeito)│ (sem cache) │ (todo request) │
│  ──────────────────┼───────────┼─────────────┼──────────────  │
│  TTL curto         │ ★★★★☆    │ ★★★☆☆      │ Médio          │
│  (Solução 3)       │ (5min gap)│ (cache mín.)│ (mais misses)  │
└───────────────────────────────────────────────────────────────┘
```

---

### Parte 4: Hotlinking Protection Completa

```javascript
// cf-function-anti-hotlink.js
// Bloqueia hotlinking de imagens, vídeos e assets
function handler(event) {
    var request = event.request;
    var uri = request.uri;

    // Apenas proteger assets (não bloquear HTML/API)
    var protectedExtensions = [
        '.jpg', '.jpeg', '.png', '.gif', '.webp', '.avif', '.svg',
        '.mp4', '.webm', '.m3u8', '.ts',
        '.pdf', '.zip'
    ];

    var isProtected = protectedExtensions.some(function(ext) {
        return uri.toLowerCase().endsWith(ext);
    });

    if (!isProtected) {
        return request;
    }

    var referer = request.headers['referer'];

    // Permitir: sem referer (acesso direto, bots legítimos, bookmarks)
    if (!referer) {
        return request;
    }

    var refererValue = referer.value.toLowerCase();

    // Domínios permitidos
    var allowedDomains = [
        'meusite.com',
        'www.meusite.com',
        'staging.meusite.com',
        // Google (imagens nos resultados de busca)
        'google.com', 'google.com.br',
        // Social media (quando compartilham links)
        't.co',  // Twitter
    ];

    var allowed = allowedDomains.some(function(domain) {
        return refererValue.indexOf(domain) !== -1;
    });

    if (!allowed) {
        // Opção A: Retornar 403
        // return {
        //     statusCode: 403,
        //     statusDescription: 'Forbidden',
        //     headers: { 'content-type': { value: 'text/plain' } },
        //     body: { encoding: 'text', data: 'Hotlinking not allowed' }
        // };

        // Opção B: Redirecionar para imagem placeholder (mais elegante)
        return {
            statusCode: 302,
            statusDescription: 'Found',
            headers: {
                'location': { value: '/images/hotlink-placeholder.jpg' },
                'cache-control': { value: 'no-store' }
            }
        };
    }

    return request;
}
```

```bash
# Publicar e associar
aws cloudfront create-function \
  --name "anti-hotlink" \
  --function-config '{"Comment":"Block hotlinking","Runtime":"cloudfront-js-2.0"}' \
  --function-code fileb://cf-function-anti-hotlink.js

aws cloudfront publish-function \
  --name "anti-hotlink" \
  --if-match $(aws cloudfront describe-function --name "anti-hotlink" --query 'ETag' --output text)

# Testar
echo "=== Hotlinking Tests ==="

# Request legítimo (referer do próprio site)
curl -s -o /dev/null -w "Status: %{http_code}\n" \
  -H "Referer: https://meusite.com/produto/123" \
  "https://$DOMAIN/images/produto.jpg"
# Esperado: 200

# Request hotlinking (referer de site pirata)
curl -s -o /dev/null -w "Status: %{http_code}, Redirect: %{redirect_url}\n" \
  -H "Referer: https://site-pirata.com/artigo-copiado" \
  "https://$DOMAIN/images/produto.jpg"
# Esperado: 302 → /images/hotlink-placeholder.jpg

# Request sem referer (acesso direto — permitido)
curl -s -o /dev/null -w "Status: %{http_code}\n" \
  "https://$DOMAIN/images/produto.jpg"
# Esperado: 200

# Request de Google Images (permitido)
curl -s -o /dev/null -w "Status: %{http_code}\n" \
  -H "Referer: https://www.google.com/search?q=produto" \
  "https://$DOMAIN/images/produto.jpg"
# Esperado: 200
```

---

### Parte 5: Load Testing com hey e k6

```bash
# ==========================================
# Instalar hey (Go HTTP load generator)
# ==========================================
# Linux
wget https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64 -O hey
chmod +x hey
sudo mv hey /usr/local/bin/

# Mac
brew install hey

# ==========================================
# Testes de Rate Limiting com hey
# ==========================================

# Teste 1: Burst de 200 requests, 50 concorrentes
hey -n 200 -c 50 "https://$DOMAIN/api/data"

# Analisar resultado:
# Status code distribution:
#   [200] 150 responses   ← Antes do rate limit
#   [403] 50 responses    ← Bloqueados pelo WAF
#
# Se todos retornarem 200, o rate limit está muito alto
# ou não está configurado

# Teste 2: Requests por segundo por 30 segundos
hey -z 30s -c 20 "https://$DOMAIN/"

# Teste 3: POST requests (simular formulário)
hey -n 100 -c 10 -m POST \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test"}' \
  "https://$DOMAIN/api/login"
```

```javascript
// k6-security-test.js — Script k6 para teste de rate limiting
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const blocked = new Rate('blocked_requests');

export const options = {
  scenarios: {
    // Cenário 1: Ramp-up gradual (simula tráfego normal crescendo)
    normal_traffic: {
      executor: 'ramping-vus',
      startVUs: 1,
      stages: [
        { duration: '30s', target: 10 },   // Ramp up
        { duration: '1m', target: 10 },    // Sustain
        { duration: '30s', target: 50 },   // Spike!
        { duration: '30s', target: 50 },   // Sustain spike
        { duration: '30s', target: 0 },    // Ramp down
      ],
    },
  },
  thresholds: {
    'blocked_requests': ['rate<0.1'],  // Menos de 10% bloqueado em tráfego normal
  },
};

const BASE_URL = __ENV.TARGET || 'https://d111222333.cloudfront.net';

export default function () {
  // Mix de requests realista
  const pages = [
    { url: '/', weight: 40 },
    { url: '/api/products', weight: 30 },
    { url: '/images/logo.png', weight: 20 },
    { url: '/static/app.js', weight: 10 },
  ];

  // Weighted random selection
  const rand = Math.random() * 100;
  let cumulative = 0;
  let selectedUrl = '/';

  for (const page of pages) {
    cumulative += page.weight;
    if (rand <= cumulative) {
      selectedUrl = page.url;
      break;
    }
  }

  const res = http.get(`${BASE_URL}${selectedUrl}`);

  // Track blocked requests
  const isBlocked = res.status === 403 || res.status === 429;
  blocked.add(isBlocked);

  check(res, {
    'not rate limited': (r) => r.status !== 403 && r.status !== 429,
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  // Log quando começar a ser bloqueado
  if (isBlocked) {
    console.warn(`BLOCKED: ${selectedUrl} → ${res.status}`);
  }

  sleep(0.1 + Math.random() * 0.5); // 100-600ms entre requests
}
```

```bash
# Executar k6
k6 run --env TARGET="https://$DOMAIN" k6-security-test.js

# Resultado esperado com rate limit funcionando:
#
# ✓ not rate limited ......... 85% (durante ramp normal)
# ✗ not rate limited ......... 60% (durante spike)
#
# Se 100% passa mesmo durante spike → rate limit está muito alto
# Se < 50% passa durante tráfego normal → rate limit muito agressivo
```

---

### Parte 6: Pentest Checklist — CloudFront

```
┌──────────────────────────────────────────────────────────────────────┐
│                 PENTEST CHECKLIST — CloudFront                        │
│                                                                       │
│  PRÉ-ASSESSMENT                                                     │
│  □ Identificar domínio e distribution ID                             │
│  □ Mapear todos os behaviors (paths) via curl/browser                │
│  □ Identificar origins (S3, ALB, API GW, custom)                    │
│  □ Documentar tecnologias do backend (Server header, errors)        │
│                                                                       │
│  CAMADA 1: REDE E PROTOCOLO                                         │
│  □ Verificar redirect HTTP → HTTPS                                   │
│  □ Testar TLS 1.0 rejeitado                                         │
│  □ Testar TLS 1.1 rejeitado                                         │
│  □ Verificar TLS 1.2/1.3 funcional                                  │
│  □ Verificar cipher suites (SSL Labs: ssllabs.com)                  │
│  □ Verificar certificado válido e não expirado                      │
│  □ Testar HSTS header com preload                                   │
│                                                                       │
│  CAMADA 2: WAF                                                       │
│  □ SQL Injection (5 payloads: OR, UNION, DROP, blind, time-based)   │
│  □ XSS (5 payloads: script, event handler, SVG, URI, encoded)      │
│  □ Path Traversal (4 payloads: basic, encoded, double, null byte)   │
│  □ Command Injection (4 payloads: pipe, semicolon, backtick, $())   │
│  □ SSRF (tentativa de acessar 169.254.169.254)                      │
│  □ User-Agent malicioso (sqlmap, nikto, nmap)                       │
│  □ Rate limiting (burst de 200+ requests)                           │
│  □ Geo blocking (VPN para país bloqueado, se aplicável)             │
│                                                                       │
│  CAMADA 3: HEADERS DE SEGURANÇA                                     │
│  □ Strict-Transport-Security (HSTS)                                 │
│  □ X-Frame-Options (DENY ou SAMEORIGIN)                             │
│  □ X-Content-Type-Options (nosniff)                                 │
│  □ Content-Security-Policy (CSP)                                    │
│  □ Referrer-Policy                                                   │
│  □ Permissions-Policy                                                │
│  □ Server header não expõe tecnologia                                │
│  □ securityheaders.com score A+                                     │
│  □ Mozilla Observatory score A+                                      │
│                                                                       │
│  CAMADA 4: ACESSO E AUTENTICAÇÃO                                    │
│  □ Signed URL sem assinatura retorna 403                            │
│  □ Signed URL com assinatura expirada retorna 403                   │
│  □ Signed URL com assinatura inválida retorna 403                   │
│  □ Cache não serve conteúdo signed após expiração                   │
│  □ Signed Cookie sem cookies retorna 403                             │
│  □ JWT expirado é rejeitado pelo Lambda@Edge                        │
│  □ JWT com signature inválida é rejeitado                           │
│  □ /admin/* bloqueado por IP whitelist no WAF                       │
│                                                                       │
│  CAMADA 5: ORIGIN PROTECTION                                        │
│  □ S3 bucket inacessível diretamente (OAC)                         │
│  □ ALB inacessível sem custom header do CloudFront                  │
│  □ API Gateway com API key/auth (não aberto)                        │
│  □ Hotlinking bloqueado (Referer check)                             │
│                                                                       │
│  CAMADA 6: CORS                                                     │
│  □ Origins legítimas permitidas                                      │
│  □ Origins maliciosas bloqueadas                                     │
│  □ Wildcard * NÃO usado em produção (exceto APIs públicas)         │
│  □ Credentials mode alinhado com allow-credentials header           │
│  □ Preflight (OPTIONS) cacheado (max-age > 0)                      │
│                                                                       │
│  CAMADA 7: DADOS E CACHE                                            │
│  □ Dados sensíveis não estão no cache key                           │
│  □ Authorization header não está em logs                            │
│  □ Cookies de sessão não estão no cache key (evitar cache poisoning)│
│  □ Error pages customizadas (não expõem stack traces)               │
│  □ Field-Level Encryption para dados PCI (se aplicável)             │
│                                                                       │
│  PÓS-ASSESSMENT                                                     │
│  □ Documentar todas as findings (pass/fail/warn)                    │
│  □ Priorizar remediação por severidade                              │
│  □ Re-testar após remediação                                        │
│  □ Agendar próximo assessment (trimestral)                          │
└──────────────────────────────────────────────────────────────────────┘
```

### Validação

```bash
# Executar o script de assessment completo
chmod +x security-assessment.sh
./security-assessment.sh d111222333.cloudfront.net

# Verificar output:
# ✅ PASS: 15+
# ❌ FAIL: 0
# ⚠️  WARN: poucos

# Verificar externamente:
echo "Abra no browser:"
echo "  https://securityheaders.com/?q=https://$DOMAIN"
echo "  https://observatory.mozilla.org/analyze/$DOMAIN"
echo "  https://www.ssllabs.com/ssltest/analyze.html?d=$DOMAIN"
echo "  https://$DOMAIN/security-lab.html"
```

```
✅ Script de assessment CLI (7 categorias, 30+ testes)
✅ App HTML interativo para testes no browser
✅ OWASP Top 10 mapeado para defesas CloudFront
✅ Bug de signed URL + cache reproduzido e 3 soluções documentadas
✅ Anti-hotlinking com CloudFront Function
✅ Load testing com hey e k6 para rate limiting
✅ Pentest checklist completo (7 camadas, 40+ items)
✅ Integração com ferramentas externas (SSL Labs, Mozilla Observatory, securityheaders.com)
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| Security Assessment | Testar CADA camada de defesa sistematicamente |
| OWASP Top 10 | CloudFront+WAF cobre 9 dos 10 items (A01-A10) |
| Bug de cache + signed URL | Token expirado servido do cache — resolver com QS fora do cache key |
| Hotlinking | CF Function validando Referer com redirect para placeholder |
| Rate limit testing | hey/k6 são mais realistas que bash loops |
| Pentest checklist | 7 camadas, 40+ items, reutilizável trimestralmente |
| Ferramentas externas | SSL Labs (TLS), securityheaders.com (headers), Observatory (overall) |
| Script automatizado | Assessment CLI reproduzível com score A/B/C/F |

> **💡 Expert Tip:** O bug mais perigoso que documentamos aqui — token expirado servido do cache — é **real e comum em produção**. Vi isso em 3 auditorias diferentes. A causa raiz é sempre a mesma: signed URLs com query strings no cache key. CloudFront valida a assinatura no primeiro request, cacheia a resposta, e os requests subsequentes servem do cache sem revalidar. A solução é contraintuitiva: remova as query strings do cache key. CloudFront ainda valida a assinatura em cada request (antes de checar o cache), mas usa o path como cache key, evitando que assinaturas diferentes criem entradas de cache separadas. Execute este assessment trimestralmente e após cada mudança significativa na distribuição.

---

## Desafio Bônus 2: Segurança AWS — Ecossistema Completo de Serviços

> **Level:** 400 | **Tempo:** 180 min | **Custo:** ~$3-5

### Objetivo

Integrar CloudFront com **todo o ecossistema de segurança da AWS**: CloudTrail para auditoria, Config para compliance, Security Hub para postura, GuardDuty para detecção de ameaças, Firewall Manager para gestão centralizada, IAM com least privilege, Secrets Manager para credenciais e SCPs para governança organizacional.

### Contexto

Em produção enterprise, CloudFront não opera sozinho. Ele faz parte de um **security mesh** — um conjunto interconectado de serviços AWS que protegem, auditam, detectam e respondem a ameaças. Um especialista CloudFront precisa dominar essas integrações.

```
┌──────────────────────────────────────────────────────────────────────┐
│                AWS Security Mesh para CloudFront                      │
│                                                                       │
│                         ┌──────────────┐                             │
│                         │  CloudFront  │                             │
│                         └──────┬───────┘                             │
│                                │                                      │
│         ┌──────────┬───────────┼───────────┬──────────┐              │
│         │          │           │           │          │              │
│    ┌────┴───┐ ┌────┴───┐ ┌────┴───┐ ┌─────┴──┐ ┌────┴───┐         │
│    │  WAF   │ │Shield  │ │  ACM   │ │  KMS   │ │  IAM   │         │
│    │(filtra)│ │(DDoS)  │ │(certs) │ │(crypto)│ │(access)│         │
│    └────────┘ └────────┘ └────────┘ └────────┘ └────────┘         │
│                                                                       │
│    ┌─────────────────────────────────────────────────────────┐       │
│    │              Camada de Observabilidade                    │       │
│    │                                                          │       │
│    │  CloudTrail  → Quem fez o quê? Quando? De onde?         │       │
│    │  Config      → Está em compliance? Mudou algo?          │       │
│    │  Security Hub→ Qual a postura geral? Findings?          │       │
│    │  GuardDuty   → Há ameaças ativas? IPs suspeitos?        │       │
│    └─────────────────────────────────────────────────────────┘       │
│                                                                       │
│    ┌─────────────────────────────────────────────────────────┐       │
│    │              Camada de Governança                         │       │
│    │                                                          │       │
│    │  Organizations + SCPs → O que é PERMITIDO configurar?   │       │
│    │  Firewall Manager    → WAF centralizado cross-account   │       │
│    │  Secrets Manager     → Credenciais seguras e rotativas  │       │
│    └─────────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────┘
```

---

### Parte 1: AWS CloudTrail — Auditoria de API Calls

CloudTrail registra **toda chamada de API** feita na sua conta AWS, incluindo CloudFront. Essencial para: auditoria de segurança, investigação de incidentes, compliance (SOC 2, PCI DSS, HIPAA).

#### O Que CloudTrail Captura para CloudFront

```
Eventos CloudTrail para CloudFront:
├── Management Events (quem mudou o quê)
│   ├── CreateDistribution
│   ├── UpdateDistribution       ← Quem alterou behaviors/origins?
│   ├── DeleteDistribution
│   ├── CreateInvalidation
│   ├── CreateFunction           ← Quem deployou código no edge?
│   ├── PublishFunction
│   ├── UpdateFunction
│   ├── AssociateWebACL          ← Quem anexou/removeu WAF?
│   ├── CreateOriginAccessControl
│   ├── CreateCachePolicy
│   ├── CreateKeyGroup           ← Quem alterou chaves de assinatura?
│   └── TagResource / UntagResource
│
└── Informações por Evento
    ├── eventTime: quando
    ├── userIdentity: quem (IAM user/role/assumed role)
    ├── sourceIPAddress: de onde
    ├── eventName: o que (CreateDistribution, etc.)
    ├── requestParameters: com quais parâmetros
    ├── responseElements: resultado
    └── errorCode: se falhou (AccessDenied, etc.)
```

#### Configurar CloudTrail para CloudFront

```bash
# Criar trail que captura eventos CloudFront
aws cloudtrail create-trail \
  --name "security-audit-trail" \
  --s3-bucket-name "empresa-cloudtrail-logs" \
  --is-multi-region-trail \
  --include-global-service-events \
  --enable-log-file-validation

# Iniciar logging
aws cloudtrail start-logging --name "security-audit-trail"

# CloudWatch Logs integration (para alertas em tempo real)
aws cloudtrail update-trail \
  --name "security-audit-trail" \
  --cloud-watch-logs-log-group-arn "arn:aws:logs:us-east-1:111111111111:log-group:cloudtrail-logs:*" \
  --cloud-watch-logs-role-arn "arn:aws:iam::111111111111:role/CloudTrailCloudWatchRole"
```

#### Queries de Auditoria — CloudTrail + Athena

```sql
-- Tabela Athena para CloudTrail (DDL)
CREATE EXTERNAL TABLE cloudtrail_logs (
  eventVersion STRING,
  userIdentity STRUCT<
    type: STRING,
    principalId: STRING,
    arn: STRING,
    accountId: STRING,
    invokedBy: STRING,
    accessKeyId: STRING,
    userName: STRING,
    sessionContext: STRUCT<
      attributes: STRUCT<mfaAuthenticated: STRING, creationDate: STRING>,
      sessionIssuer: STRUCT<type: STRING, principalId: STRING, arn: STRING, accountId: STRING, userName: STRING>
    >
  >,
  eventTime STRING,
  eventSource STRING,
  eventName STRING,
  awsRegion STRING,
  sourceIPAddress STRING,
  userAgent STRING,
  errorCode STRING,
  errorMessage STRING,
  requestParameters STRING,
  responseElements STRING,
  requestId STRING,
  eventId STRING,
  readOnly STRING,
  resources ARRAY<STRUCT<arn: STRING, accountId: STRING, type: STRING>>,
  eventType STRING
)
PARTITIONED BY (region STRING, year STRING, month STRING, day STRING)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION 's3://empresa-cloudtrail-logs/AWSLogs/111111111111/CloudTrail/'
TBLPROPERTIES ('projection.enabled' = 'true',
  'projection.region.type' = 'enum',
  'projection.region.values' = 'us-east-1,sa-east-1',
  'projection.year.type' = 'integer', 'projection.year.range' = '2024,2030',
  'projection.month.type' = 'integer', 'projection.month.range' = '1,12', 'projection.month.digits' = '2',
  'projection.day.type' = 'integer', 'projection.day.range' = '1,31', 'projection.day.digits' = '2',
  'storage.location.template' = 's3://empresa-cloudtrail-logs/AWSLogs/111111111111/CloudTrail/${region}/${year}/${month}/${day}');

-- 1. Quem alterou distributions nos últimos 7 dias?
SELECT
  eventTime,
  userIdentity.arn AS who,
  eventName AS action,
  sourceIPAddress AS from_ip,
  errorCode
FROM cloudtrail_logs
WHERE eventSource = 'cloudfront.amazonaws.com'
  AND eventName LIKE '%Distribution%'
  AND year = '2026' AND month = '04'
  AND CAST(day AS INTEGER) >= DAY(current_date) - 7
ORDER BY eventTime DESC;

-- 2. Quem publicou CloudFront Functions?
SELECT
  eventTime,
  userIdentity.arn AS who,
  eventName,
  json_extract_scalar(requestParameters, '$.name') AS function_name,
  sourceIPAddress
FROM cloudtrail_logs
WHERE eventSource = 'cloudfront.amazonaws.com'
  AND eventName IN ('CreateFunction', 'UpdateFunction', 'PublishFunction')
  AND year = '2026'
ORDER BY eventTime DESC;

-- 3. Tentativas de acesso NEGADO a CloudFront
SELECT
  eventTime,
  userIdentity.arn AS who,
  eventName AS attempted_action,
  errorCode,
  errorMessage,
  sourceIPAddress
FROM cloudtrail_logs
WHERE eventSource = 'cloudfront.amazonaws.com'
  AND errorCode = 'AccessDenied'
  AND year = '2026'
ORDER BY eventTime DESC
LIMIT 50;

-- 4. Quem removeu WAF de alguma distribution?
SELECT
  eventTime,
  userIdentity.arn AS who,
  eventName,
  requestParameters,
  sourceIPAddress
FROM cloudtrail_logs
WHERE eventSource = 'cloudfront.amazonaws.com'
  AND eventName IN ('UpdateDistribution')
  AND requestParameters LIKE '%WebACLId%'
  AND year = '2026'
ORDER BY eventTime DESC;

-- 5. Mudanças fora do horário comercial (suspeitas)
SELECT
  eventTime,
  userIdentity.arn AS who,
  eventName,
  sourceIPAddress
FROM cloudtrail_logs
WHERE eventSource = 'cloudfront.amazonaws.com'
  AND eventName NOT LIKE 'Get%'
  AND eventName NOT LIKE 'List%'
  AND eventName NOT LIKE 'Describe%'
  AND (
    HOUR(from_iso8601_timestamp(eventTime)) < 8
    OR HOUR(from_iso8601_timestamp(eventTime)) > 20
  )
  AND year = '2026'
ORDER BY eventTime DESC;
```

#### Alarme: Mudança Crítica em CloudFront

```hcl
# CloudWatch Metric Filter para detectar mudanças críticas
resource "aws_cloudwatch_log_metric_filter" "cf_critical_changes" {
  name           = "cloudfront-critical-changes"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  pattern        = <<-PATTERN
    { ($.eventSource = "cloudfront.amazonaws.com") &&
      ($.eventName = "UpdateDistribution" ||
       $.eventName = "DeleteDistribution" ||
       $.eventName = "UpdateFunction" ||
       $.eventName = "PublishFunction" ||
       $.eventName = "CreateInvalidation") }
  PATTERN

  metric_transformation {
    name          = "CloudFrontCriticalChanges"
    namespace     = "SecurityAudit"
    value         = "1"
    default_value = "0"
  }
}

resource "aws_cloudwatch_metric_alarm" "cf_critical_changes" {
  alarm_name          = "cloudfront-critical-change-detected"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "CloudFrontCriticalChanges"
  namespace           = "SecurityAudit"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Alguém fez mudança crítica em CloudFront (UpdateDistribution, DeleteDistribution, PublishFunction)"

  alarm_actions = [aws_sns_topic.security_alerts.arn]
}

# Alarme específico: WAF removido de distribution
resource "aws_cloudwatch_log_metric_filter" "cf_waf_removed" {
  name           = "cloudfront-waf-removed"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  pattern        = <<-PATTERN
    { ($.eventSource = "cloudfront.amazonaws.com") &&
      ($.eventName = "UpdateDistribution") &&
      ($.requestParameters.distributionConfig.webACLId = "") }
  PATTERN

  metric_transformation {
    name      = "CloudFrontWAFRemoved"
    namespace = "SecurityAudit"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "cf_waf_removed" {
  alarm_name          = "CRITICAL-cloudfront-waf-removed"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "CloudFrontWAFRemoved"
  namespace           = "SecurityAudit"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "CRITICAL: WAF foi removido de uma CloudFront distribution!"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}
```

---

### Parte 2: AWS Config — Compliance Contínua

AWS Config monitora **o estado de configuração** dos seus recursos e verifica contra regras de compliance.

#### Config Rules para CloudFront

```bash
# Habilitar AWS Config (se ainda não habilitado)
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::111111111111:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

aws configservice start-configuration-recorder --configuration-recorder-name default
```

```hcl
# ============================================
# AWS CONFIG RULES PARA CLOUDFRONT
# ============================================

# Regra 1: CloudFront DEVE ter WAF associado
resource "aws_config_config_rule" "cf_waf_enabled" {
  name = "cloudfront-associated-with-waf"

  source {
    owner             = "AWS"
    source_identifier = "CLOUDFRONT_ASSOCIATED_WITH_WAF"
  }

  scope {
    compliance_resource_types = ["AWS::CloudFront::Distribution"]
  }

  tags = { Security = "mandatory" }
}

# Regra 2: CloudFront DEVE usar HTTPS (viewer-protocol-policy)
resource "aws_config_config_rule" "cf_viewer_policy" {
  name = "cloudfront-viewer-policy-https"

  source {
    owner             = "AWS"
    source_identifier = "CLOUDFRONT_VIEWER_POLICY_HTTPS"
  }

  scope {
    compliance_resource_types = ["AWS::CloudFront::Distribution"]
  }
}

# Regra 3: CloudFront DEVE ter logging habilitado
resource "aws_config_config_rule" "cf_logging_enabled" {
  name = "cloudfront-accesslogs-enabled"

  source {
    owner             = "AWS"
    source_identifier = "CLOUDFRONT_ACCESSLOGS_ENABLED"
  }

  scope {
    compliance_resource_types = ["AWS::CloudFront::Distribution"]
  }
}

# Regra 4: CloudFront origin NÃO deve usar HTTP (deve usar HTTPS para origin)
resource "aws_config_config_rule" "cf_origin_ssl" {
  name = "cloudfront-no-deprecated-ssl"

  source {
    owner             = "AWS"
    source_identifier = "CLOUDFRONT_NO_DEPRECATED_SSL_PROTOCOLS"
  }

  scope {
    compliance_resource_types = ["AWS::CloudFront::Distribution"]
  }
}

# Regra 5: CloudFront DEVE ter TLS 1.2 mínimo
resource "aws_config_config_rule" "cf_minimum_tls" {
  name = "cloudfront-minimum-tls-version"

  source {
    owner             = "AWS"
    source_identifier = "CLOUDFRONT_SNI_ENABLED"
  }

  scope {
    compliance_resource_types = ["AWS::CloudFront::Distribution"]
  }
}

# Regra 6: S3 bucket origin NÃO deve ser público
resource "aws_config_config_rule" "s3_no_public" {
  name = "s3-bucket-public-read-prohibited"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }
}

# Regra 7: CloudFront default root object configurado
resource "aws_config_config_rule" "cf_default_root" {
  name = "cloudfront-default-root-object-configured"

  source {
    owner             = "AWS"
    source_identifier = "CLOUDFRONT_DEFAULT_ROOT_OBJECT_CONFIGURED"
  }

  scope {
    compliance_resource_types = ["AWS::CloudFront::Distribution"]
  }
}

# Regra customizada: CloudFront DEVE ter Origin Access Control (não OAI legado)
resource "aws_config_config_rule" "cf_oac_required" {
  name = "cloudfront-uses-oac-not-oai"

  source {
    owner = "CUSTOM_LAMBDA"
    source_identifier = aws_lambda_function.config_rule_oac.arn

    source_detail {
      event_source = "aws.config"
      message_type = "ConfigurationItemChangeNotification"
    }
  }

  scope {
    compliance_resource_types = ["AWS::CloudFront::Distribution"]
  }

  depends_on = [aws_lambda_permission.config_rule_oac]
}

# ============================================
# AUTO-REMEDIATION: Se WAF for removido, re-associar
# ============================================
resource "aws_config_remediation_configuration" "cf_waf_remediation" {
  config_rule_name = aws_config_config_rule.cf_waf_enabled.name
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-EnableCloudFrontWAF"  # Custom SSM document

  parameter {
    name         = "DistributionId"
    resource_value = "RESOURCE_ID"
  }

  parameter {
    name         = "WebACLArn"
    static_value = aws_wafv2_web_acl.main.arn
  }

  automatic = true
  maximum_automatic_attempts = 3
  retry_attempt_seconds      = 60
}
```

#### Verificar Compliance

```bash
# Ver compliance de todas as distributions
aws configservice get-compliance-details-by-resource \
  --resource-type "AWS::CloudFront::Distribution" \
  --resource-id "E1EXAMPLE" \
  --output table

# Ver todas as regras non-compliant
aws configservice describe-compliance-by-config-rule \
  --compliance-types NON_COMPLIANT \
  --output table

# Dashboard de compliance (resumo)
aws configservice get-compliance-summary-by-resource-type \
  --resource-types "AWS::CloudFront::Distribution" \
  --output json
```

---

### Parte 3: AWS Security Hub — Postura de Segurança

Security Hub agrega findings de múltiplos serviços (Config, GuardDuty, Inspector, WAF) num dashboard centralizado.

```bash
# Habilitar Security Hub
aws securityhub enable-security-hub \
  --enable-default-standards

# Habilitar o standard AWS Foundational Security Best Practices
aws securityhub batch-enable-standards \
  --standards-subscription-requests '[{
    "StandardsArn": "arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0"
  }]'
```

#### Controls do Security Hub para CloudFront

```
AWS Foundational Security Best Practices — CloudFront Controls:

┌──────────────┬────────────────────────────────────────────┬──────────┐
│ Control ID   │ Descrição                                  │ Severity │
├──────────────┼────────────────────────────────────────────┼──────────┤
│ CloudFront.1 │ Default root object deve estar configurado │ CRITICAL │
│ CloudFront.3 │ Viewer deve usar HTTPS                     │ MEDIUM   │
│ CloudFront.4 │ Origin failover deve estar configurado     │ LOW      │
│ CloudFront.5 │ Access logging deve estar habilitado       │ MEDIUM   │
│ CloudFront.6 │ WAF deve estar associado                   │ MEDIUM   │
│ CloudFront.7 │ Deve usar SSL/TLS certificate customizado  │ LOW      │
│ CloudFront.8 │ Deve usar SNI para HTTPS requests          │ LOW      │
│ CloudFront.9 │ Tráfego para custom origins deve ser       │ MEDIUM   │
│              │ criptografado (HTTPS)                      │          │
│ CloudFront.10│ Não deve usar protocolos SSL depreciados   │ CRITICAL │
│              │ entre edge locations e custom origins       │          │
│ CloudFront.12│ Não deve apontar para S3 origins que       │ MEDIUM   │
│              │ não existem                                │          │
│ CloudFront.13│ Deve usar Origin Access Control (não OAI)  │ MEDIUM   │
│ CloudFront.14│ Distributions devem ser taggeadas          │ LOW      │
└──────────────┴────────────────────────────────────────────┴──────────┘
```

#### Terraform — Security Hub + Automação

```hcl
# Habilitar Security Hub
resource "aws_securityhub_account" "main" {}

# Habilitar AWS Foundational Security Best Practices
resource "aws_securityhub_standards_subscription" "aws_fsbp" {
  depends_on    = [aws_securityhub_account.main]
  standards_arn = "arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0"
}

# Habilitar CIS AWS Foundations Benchmark
resource "aws_securityhub_standards_subscription" "cis" {
  depends_on    = [aws_securityhub_account.main]
  standards_arn = "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.4.0"
}

# Custom Action para auto-remediation de findings
resource "aws_securityhub_action_target" "remediate_cf" {
  depends_on  = [aws_securityhub_account.main]
  name        = "RemediateCloudFront"
  identifier  = "RemediateCF"
  description = "Auto-remediate CloudFront security findings"
}

# EventBridge rule para findings CloudFront CRITICAL
resource "aws_cloudwatch_event_rule" "cf_critical_findings" {
  name        = "securityhub-cf-critical"
  description = "CloudFront CRITICAL findings from Security Hub"

  event_pattern = jsonencode({
    source      = ["aws.securityhub"]
    detail-type = ["Security Hub Findings - Imported"]
    detail = {
      findings = {
        ProductFields = {
          "StandardsControlArn" = [{ "prefix" = "arn:aws:securityhub" }]
        }
        Resources = {
          Type = ["AwsCloudFrontDistribution"]
        }
        Severity = {
          Label = ["CRITICAL", "HIGH"]
        }
        Compliance = {
          Status = ["FAILED"]
        }
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "cf_finding_alert" {
  rule = aws_cloudwatch_event_rule.cf_critical_findings.name
  arn  = aws_sns_topic.security_alerts.arn

  input_transformer {
    input_paths = {
      title    = "$.detail.findings[0].Title"
      severity = "$.detail.findings[0].Severity.Label"
      resource = "$.detail.findings[0].Resources[0].Id"
      account  = "$.detail.findings[0].AwsAccountId"
    }
    input_template = <<-EOF
      "SECURITY HUB FINDING: <title> | Severity: <severity> | Resource: <resource> | Account: <account>"
    EOF
  }
}
```

#### Consultar Findings

```bash
# Listar findings CloudFront FAILED
aws securityhub get-findings \
  --filters '{
    "ResourceType": [{"Value": "AwsCloudFrontDistribution", "Comparison": "EQUALS"}],
    "ComplianceStatus": [{"Value": "FAILED", "Comparison": "EQUALS"}]
  }' \
  --query 'Findings[].{
    Title: Title,
    Severity: Severity.Label,
    Resource: Resources[0].Id,
    Status: Compliance.Status
  }' \
  --output table

# Resumo por severity
aws securityhub get-findings \
  --filters '{
    "ResourceType": [{"Value": "AwsCloudFrontDistribution", "Comparison": "EQUALS"}],
    "ComplianceStatus": [{"Value": "FAILED", "Comparison": "EQUALS"}]
  }' \
  --query 'Findings[].Severity.Label' \
  --output json | jq 'group_by(.) | map({severity: .[0], count: length})'
```

---

### Parte 4: AWS IAM — Least Privilege para CloudFront

```
┌──────────────────────────────────────────────────────────────────┐
│         IAM Policies para CloudFront — 4 Níveis                   │
│                                                                   │
│  Nível 1: Read-Only (desenvolvedores, suporte)                  │
│  Nível 2: Operator (DevOps — deploy, invalidação)               │
│  Nível 3: Admin (infra team — full CloudFront)                  │
│  Nível 4: Security Admin (WAF, Shield, keys)                    │
└──────────────────────────────────────────────────────────────────┘
```

```hcl
# ============================================
# NÍVEL 1: Read-Only
# ============================================
resource "aws_iam_policy" "cf_readonly" {
  name = "CloudFront-ReadOnly"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "CloudFrontReadOnly"
        Effect = "Allow"
        Action = [
          "cloudfront:Get*",
          "cloudfront:List*",
          "cloudfront:DescribeFunction"
        ]
        Resource = "*"
      },
      {
        Sid    = "ViewWAFReadOnly"
        Effect = "Allow"
        Action = [
          "wafv2:Get*",
          "wafv2:List*"
        ]
        Resource = "*"
      }
    ]
  })
}

# ============================================
# NÍVEL 2: Operator (deploy + invalidation)
# ============================================
resource "aws_iam_policy" "cf_operator" {
  name = "CloudFront-Operator"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "CloudFrontReadAndInvalidate"
        Effect = "Allow"
        Action = [
          "cloudfront:Get*",
          "cloudfront:List*",
          "cloudfront:DescribeFunction",
          "cloudfront:CreateInvalidation",  # Deploy: invalidar cache
          "cloudfront:TestFunction",         # Testar functions
          "cloudfront:UpdateFunction",       # Atualizar functions
          "cloudfront:PublishFunction"        # Publicar functions
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyDestructiveActions"
        Effect = "Deny"
        Action = [
          "cloudfront:DeleteDistribution",
          "cloudfront:DeleteFunction",
          "cloudfront:DeleteKeyGroup",
          "cloudfront:DeletePublicKey"
        ]
        Resource = "*"
      }
    ]
  })
}

# ============================================
# NÍVEL 3: Admin (full CloudFront, restrição por tag)
# ============================================
resource "aws_iam_policy" "cf_admin" {
  name = "CloudFront-Admin"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "CloudFrontFullAccess"
        Effect = "Allow"
        Action = [
          "cloudfront:*"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "aws:ResourceTag/Environment" = ["production", "staging"]
          }
        }
      },
      {
        Sid    = "CloudFrontReadAll"
        Effect = "Allow"
        Action = [
          "cloudfront:Get*",
          "cloudfront:List*"
        ]
        Resource = "*"
      },
      {
        Sid    = "RequiredServices"
        Effect = "Allow"
        Action = [
          "acm:DescribeCertificate",
          "acm:ListCertificates",
          "s3:GetBucketPolicy",
          "s3:PutBucketPolicy",
          "wafv2:Get*",
          "wafv2:List*",
          "wafv2:AssociateWebACL",
          "wafv2:DisassociateWebACL"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyDeleteInProd"
        Effect = "Deny"
        Action = [
          "cloudfront:DeleteDistribution"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "aws:ResourceTag/Environment" = "production"
          }
        }
      }
    ]
  })
}

# ============================================
# NÍVEL 4: Security Admin (WAF, Shield, keys)
# ============================================
resource "aws_iam_policy" "cf_security_admin" {
  name = "CloudFront-SecurityAdmin"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "WAFFullAccess"
        Effect = "Allow"
        Action = ["wafv2:*"]
        Resource = "*"
      },
      {
        Sid    = "ShieldAccess"
        Effect = "Allow"
        Action = ["shield:*"]
        Resource = "*"
      },
      {
        Sid    = "CloudFrontSecurityOnly"
        Effect = "Allow"
        Action = [
          "cloudfront:GetDistribution",
          "cloudfront:UpdateDistribution",
          "cloudfront:Get*",
          "cloudfront:List*",
          "cloudfront:CreatePublicKey",
          "cloudfront:CreateKeyGroup",
          "cloudfront:UpdateKeyGroup",
          "cloudfront:CreateFieldLevelEncryptionConfig",
          "cloudfront:CreateFieldLevelEncryptionProfile"
        ]
        Resource = "*"
      },
      {
        Sid    = "KMSForCloudFront"
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey",
          "kms:GenerateDataKey"
        ]
        Resource = "arn:aws:kms:*:*:key/*"
        Condition = {
          StringEquals = {
            "kms:ViaService" = "cloudfront.amazonaws.com"
          }
        }
      }
    ]
  })
}
```

#### Condition Keys Específicas de CloudFront

```hcl
# Permitir CreateDistribution SOMENTE com WAF obrigatório
resource "aws_iam_policy" "cf_require_waf" {
  name = "CloudFront-RequireWAF"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyDistributionWithoutWAF"
        Effect = "Deny"
        Action = [
          "cloudfront:CreateDistribution",
          "cloudfront:UpdateDistribution"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "cloudfront:WebACLId" = ""  # Bloqueia se WAF está vazio
          }
        }
      }
    ]
  })
}
```

---

### Parte 5: AWS Secrets Manager — Credenciais Seguras

```
┌──────────────────────────────────────────────────────────────────┐
│         Secrets Manager — O Que Armazenar                         │
│                                                                   │
│  1. Private Keys para Signed URLs (RSA .pem)                    │
│  2. API Keys de terceiros (proxied via CloudFront)               │
│  3. Custom origin headers (X-Origin-Verify)                      │
│  4. mTLS client certificates                                     │
│  5. WAF IP sets (atualizados dinamicamente)                      │
└──────────────────────────────────────────────────────────────────┘
```

```hcl
# ============================================
# SECRETS MANAGER — Signed URL Private Key
# ============================================

# Armazenar private key para signed URLs
resource "aws_secretsmanager_secret" "cf_signing_key" {
  name        = "cloudfront/signing-key"
  description = "RSA private key for CloudFront signed URLs"

  tags = {
    Service = "cloudfront"
    Usage   = "signed-urls"
  }
}

resource "aws_secretsmanager_secret_version" "cf_signing_key" {
  secret_id     = aws_secretsmanager_secret.cf_signing_key.id
  secret_string = file("${path.module}/keys/private_key.pem")

  lifecycle {
    ignore_changes = [secret_string]  # Não sobrescrever em re-apply
  }
}

# Rotação automática (a cada 90 dias)
resource "aws_secretsmanager_secret_rotation" "cf_signing_key" {
  secret_id           = aws_secretsmanager_secret.cf_signing_key.id
  rotation_lambda_arn = aws_lambda_function.rotate_signing_key.arn

  rotation_rules {
    automatically_after_days = 90
  }
}

# ============================================
# SECRETS MANAGER — Origin Custom Header
# ============================================

resource "aws_secretsmanager_secret" "origin_verify" {
  name        = "cloudfront/origin-verify-header"
  description = "Custom header value for CloudFront → ALB origin protection"
}

resource "aws_secretsmanager_secret_version" "origin_verify" {
  secret_id = aws_secretsmanager_secret.origin_verify.id
  secret_string = jsonencode({
    header_name  = "X-Origin-Verify"
    header_value = random_password.origin_verify.result
  })
}

resource "random_password" "origin_verify" {
  length  = 64
  special = false
}

# ============================================
# SECRETS MANAGER — API Key de Terceiro
# ============================================

resource "aws_secretsmanager_secret" "external_api_key" {
  name        = "cloudfront/external-api-key"
  description = "API key for proxied external API (e.g., weather service)"
}

# ============================================
# Lambda@Edge lendo do Secrets Manager
# ============================================
# ATENÇÃO: Lambda@Edge NÃO suporta variáveis de ambiente!
# A private key deve ser empacotada no deployment package
# OU buscada do Secrets Manager no cold start com cache.

# Para CloudFront Functions: use KeyValueStore em vez de Secrets Manager
# (CF Functions não têm acesso a AWS services)
```

```python
# Lambda@Edge com Secrets Manager (exemplo de caching)
import boto3
import json
import time

# Cache global (persiste entre invocações no mesmo container)
_secrets_cache = {}
_cache_ttl = 300  # 5 minutos

def get_secret(secret_name, region='us-east-1'):
    """Busca secret com cache para evitar chamadas excessivas."""
    now = time.time()
    if secret_name in _secrets_cache:
        cached = _secrets_cache[secret_name]
        if now - cached['timestamp'] < _cache_ttl:
            return cached['value']

    client = boto3.client('secretsmanager', region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    value = response['SecretString']

    _secrets_cache[secret_name] = {
        'value': value,
        'timestamp': now
    }
    return value


def handler(event, context):
    """Lambda@Edge que valida API key armazenada no Secrets Manager."""
    request = event['Records'][0]['cf']['request']

    # Buscar API key esperada (cacheada)
    secret = json.loads(get_secret('cloudfront/external-api-key'))
    expected_key = secret.get('api_key')

    # Validar header da request
    headers = request.get('headers', {})
    provided_key = headers.get('x-api-key', [{}])[0].get('value', '')

    if provided_key != expected_key:
        return {
            'status': '403',
            'statusDescription': 'Forbidden',
            'body': 'Invalid API key'
        }

    return request
```

---

### Parte 6: AWS Firewall Manager — WAF Centralizado

```
┌──────────────────────────────────────────────────────────────────┐
│         Firewall Manager — Gestão Centralizada de WAF             │
│                                                                   │
│  Problema: 10 contas AWS, cada uma com CloudFront distributions  │
│  Como garantir que TODAS têm WAF com as mesmas regras?           │
│                                                                   │
│  ┌──────────────┐                                                │
│  │   Conta Admin │  Firewall Manager Policy                      │
│  │   (Security)  │  ├── Managed Rules: Common + SQLi + Bots     │
│  │               │  ├── Rate Limit: 2000 req/5min               │
│  │               │  └── Auto-apply a TODAS as distributions     │
│  └──────┬───────┘                                                │
│          │                                                        │
│     ┌────┴─────┬──────────┬──────────┬──────────┐               │
│     │          │          │          │          │               │
│  ┌──┴──┐  ┌───┴──┐  ┌───┴──┐  ┌───┴──┐  ┌───┴──┐            │
│  │Conta│  │Conta │  │Conta │  │Conta │  │Conta │            │
│  │Prod │  │Stage │  │Dev   │  │TeamA │  │TeamB │            │
│  │ CF  │  │ CF   │  │ CF   │  │ CF   │  │ CF   │            │
│  │+WAF │  │+WAF  │  │+WAF  │  │+WAF  │  │+WAF  │            │
│  └─────┘  └──────┘  └──────┘  └──────┘  └──────┘            │
│  (WAF aplicado automaticamente pelo Firewall Manager)          │
└──────────────────────────────────────────────────────────────────┘
```

```hcl
# ============================================
# FIREWALL MANAGER — Na conta admin/security
# ============================================

# Pré-requisito: designar conta admin do Firewall Manager
# aws fms put-admin-account --admin-account 111111111111

# Policy: WAF obrigatório em todas as CloudFront distributions
resource "aws_fms_policy" "cloudfront_waf" {
  name                        = "CloudFront-Mandatory-WAF"
  resource_type               = "AWS::CloudFront::Distribution"
  exclude_resource_tags       = false
  remediation_enabled         = true  # Auto-remediate: aplicar WAF automaticamente

  # Aplicar em TODAS as contas da Organization
  include_map {
    orgunit = ["ou-root-xxxxxxxx"]  # Root OU = todas as contas
  }

  security_service_policy_data {
    type = "WAFV2"

    managed_service_data = jsonencode({
      type = "WAFV2"

      preProcessRuleGroups = [
        {
          managedRuleGroupIdentifier = {
            vendorName        = "AWS"
            managedRuleGroupName = "AWSManagedRulesCommonRuleSet"
          }
          overrideAction = { type = "NONE" }
          ruleGroupOrder = 1
          ruleGroupType  = "ManagedRuleGroup"
        },
        {
          managedRuleGroupIdentifier = {
            vendorName        = "AWS"
            managedRuleGroupName = "AWSManagedRulesSQLiRuleSet"
          }
          overrideAction = { type = "NONE" }
          ruleGroupOrder = 2
          ruleGroupType  = "ManagedRuleGroup"
        },
        {
          managedRuleGroupIdentifier = {
            vendorName        = "AWS"
            managedRuleGroupName = "AWSManagedRulesKnownBadInputsRuleSet"
          }
          overrideAction = { type = "NONE" }
          ruleGroupOrder = 3
          ruleGroupType  = "ManagedRuleGroup"
        }
      ]

      postProcessRuleGroups = []

      defaultAction = { type = "ALLOW" }

      overrideCustomerWebACLAssociation = false  # Não sobrescrever WAF existente
    })
  }
}

# Policy: Logging obrigatório em todas as distributions
resource "aws_fms_policy" "cloudfront_logging" {
  name                  = "CloudFront-Mandatory-Logging"
  resource_type         = "AWS::CloudFront::Distribution"
  remediation_enabled   = true
  exclude_resource_tags = false

  include_map {
    orgunit = ["ou-root-xxxxxxxx"]
  }

  security_service_policy_data {
    type                 = "WAFV2"
    managed_service_data = jsonencode({
      type = "WAFV2"
      loggingConfiguration = {
        logDestinationConfigs = [
          "arn:aws:s3:::empresa-waf-logs-central"
        ]
      }
    })
  }
}
```

```bash
# Verificar compliance do Firewall Manager
aws fms get-compliance-detail \
  --policy-id "policy-id-here" \
  --member-account "222222222222" \
  --output json

# Listar todas as policies
aws fms list-policies --output table

# Ver status de compliance por conta
aws fms list-compliance-status \
  --policy-id "policy-id-here" \
  --output table
```

---

### Parte 7: SCPs — Guardrails Organizacionais

```hcl
# ============================================
# SCPs (Service Control Policies) — Na conta Management
# ============================================

# SCP 1: CloudFront DEVE sempre ter WAF
resource "aws_organizations_policy" "cf_require_waf" {
  name    = "RequireWAFOnCloudFront"
  type    = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyCloudFrontWithoutWAF"
        Effect = "Deny"
        Action = [
          "cloudfront:CreateDistribution",
          "cloudfront:UpdateDistribution"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "cloudfront:WebACLId" = ""
          }
        }
      }
    ]
  })
}

# SCP 2: Proibir desabilitar logging
resource "aws_organizations_policy" "cf_require_logging" {
  name    = "RequireLoggingOnCloudFront"
  type    = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyDisableCloudFrontLogs"
        Effect = "Deny"
        Action = [
          "cloudfront:UpdateDistribution"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "cloudfront:LoggingEnabled" = "false"
          }
        }
      }
    ]
  })
}

# SCP 3: Proibir protocolos TLS antigos
resource "aws_organizations_policy" "cf_modern_tls" {
  name    = "RequireModernTLSOnCloudFront"
  type    = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyDeprecatedTLSProtocols"
        Effect = "Deny"
        Action = [
          "cloudfront:CreateDistribution",
          "cloudfront:UpdateDistribution"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "cloudfront:MinimumProtocolVersion" = [
              "SSLv3", "TLSv1", "TLSv1_2016", "TLSv1.1_2016"
            ]
          }
        }
      }
    ]
  })
}

# SCP 4: Proibir criar distributions sem tags obrigatórias
resource "aws_organizations_policy" "cf_require_tags" {
  name    = "RequireTagsOnCloudFront"
  type    = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyUntaggedDistribution"
        Effect = "Deny"
        Action = [
          "cloudfront:CreateDistribution"
        ]
        Resource = "*"
        Condition = {
          "Null" = {
            "aws:RequestTag/Environment" = "true"
            "aws:RequestTag/Team"        = "true"
          }
        }
      }
    ]
  })
}

# Anexar SCPs à Organization ou OUs específicas
resource "aws_organizations_policy_attachment" "require_waf_all" {
  policy_id = aws_organizations_policy.cf_require_waf.id
  target_id = "r-root"  # Root = toda a Organization
}

resource "aws_organizations_policy_attachment" "require_waf_prod" {
  policy_id = aws_organizations_policy.cf_require_waf.id
  target_id = "ou-xxxx-prod"  # Apenas OU de produção
}
```

---

### Parte 8: Arquitetura Completa — Security Mesh

```hcl
# ============================================
# TERRAFORM: Security Mesh Completo
# ============================================

# --- CloudTrail ---
resource "aws_cloudtrail" "security" {
  name                       = "security-audit"
  s3_bucket_name             = aws_s3_bucket.cloudtrail.id
  is_multi_region_trail      = true
  include_global_service_events = true
  enable_log_file_validation = true

  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_cloudwatch.arn

  event_selector {
    read_write_type           = "All"
    include_management_events = true
  }
}

resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "cloudtrail-logs"
  retention_in_days = 365
}

# --- AWS Config ---
resource "aws_config_configuration_recorder" "main" {
  name     = "default"
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

resource "aws_config_delivery_channel" "main" {
  name           = "default"
  s3_bucket_name = aws_s3_bucket.config.id

  snapshot_delivery_properties {
    delivery_frequency = "TwentyFour_Hours"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true
  depends_on = [aws_config_delivery_channel.main]
}

# --- Security Hub ---
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "fsbp" {
  depends_on    = [aws_securityhub_account.main]
  standards_arn = "arn:aws:securityhub:${data.aws_region.current.name}::standards/aws-foundational-security-best-practices/v/1.0.0"
}

# --- GuardDuty ---
resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs {
      enable = true  # Detecta acesso anômalo ao S3 (origin do CloudFront)
    }
  }
}

# --- SNS para alertas centralizados ---
resource "aws_sns_topic" "security_alerts" {
  name = "security-alerts"
}

resource "aws_sns_topic_subscription" "security_email" {
  topic_arn = aws_sns_topic.security_alerts.arn
  protocol  = "email"
  endpoint  = "security-team@empresa.com"
}

# --- EventBridge: GuardDuty → WAF (auto-block IPs suspeitos) ---
resource "aws_cloudwatch_event_rule" "guardduty_to_waf" {
  name = "guardduty-block-suspicious-ips"

  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      severity = [{ numeric = [">=", 7] }]  # HIGH severity
    }
  })
}

resource "aws_cloudwatch_event_target" "block_ip_lambda" {
  rule = aws_cloudwatch_event_rule.guardduty_to_waf.name
  arn  = aws_lambda_function.block_suspicious_ip.arn
}
```

```python
# Lambda: GuardDuty finding → adicionar IP ao WAF IP Set
import boto3
import json

wafv2 = boto3.client('wafv2', region_name='us-east-1')

IP_SET_NAME = 'GuardDuty-Blocked-IPs'
IP_SET_ID = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
IP_SET_SCOPE = 'CLOUDFRONT'

def handler(event, context):
    """Extrai IP suspeito do GuardDuty finding e bloqueia no WAF."""
    finding = event['detail']
    severity = finding.get('severity', 0)

    # Extrair IPs remotos do finding
    ips_to_block = set()

    # Ação de rede (port scan, brute force, etc.)
    service = finding.get('service', {})
    action = service.get('action', {})

    network = action.get('networkConnectionAction', {})
    remote_ip = network.get('remoteIpDetails', {}).get('ipAddressV4')
    if remote_ip:
        ips_to_block.add(f"{remote_ip}/32")

    # Ação de API call (acesso não autorizado)
    aws_api = action.get('awsApiCallAction', {})
    api_ip = aws_api.get('remoteIpDetails', {}).get('ipAddressV4')
    if api_ip:
        ips_to_block.add(f"{api_ip}/32")

    if not ips_to_block:
        print("No IPs found in finding")
        return

    # Obter IP Set atual
    ip_set = wafv2.get_ip_set(
        Name=IP_SET_NAME,
        Scope=IP_SET_SCOPE,
        Id=IP_SET_ID
    )

    current_addresses = ip_set['IPSet']['Addresses']
    lock_token = ip_set['LockToken']

    # Adicionar novos IPs
    updated_addresses = list(set(current_addresses) | ips_to_block)

    # Limitar a 10.000 IPs (limite do WAF)
    if len(updated_addresses) > 10000:
        updated_addresses = updated_addresses[-10000:]

    wafv2.update_ip_set(
        Name=IP_SET_NAME,
        Scope=IP_SET_SCOPE,
        Id=IP_SET_ID,
        Addresses=updated_addresses,
        LockToken=lock_token
    )

    print(f"Blocked IPs: {ips_to_block}")
    print(f"Total IPs in set: {len(updated_addresses)}")

    return {
        'blocked': list(ips_to_block),
        'total_in_set': len(updated_addresses)
    }
```

---

### Mapa de Integração Completo

```
┌──────────────────────────────────────────────────────────────────────┐
│                     SECURITY MESH — FLUXO DE DADOS                    │
│                                                                       │
│  ┌─────────┐     requests     ┌──────────┐    logs     ┌─────────┐  │
│  │ Viewers │ ───────────────→ │CloudFront│ ──────────→ │ S3 Logs │  │
│  └─────────┘                  │  + WAF   │             └────┬────┘  │
│                               └────┬─────┘                  │       │
│                                    │                         │       │
│                          API calls │                  ┌──────┴────┐  │
│                                    ▼                  │  Athena   │  │
│  ┌──────────────────────────────────────────────┐     │  (query)  │  │
│  │                CloudTrail                     │     └───────────┘  │
│  │  Registra: CreateDist, UpdateDist, etc.       │                   │
│  └──────────┬───────────────────────────────────┘                   │
│             │                                                        │
│             ▼                                                        │
│  ┌──────────────────┐    findings    ┌────────────────┐             │
│  │  CloudWatch Logs  │ ─────────────→│  Security Hub  │             │
│  │  (metric filters) │               │  (dashboard)   │             │
│  └──────────────────┘               └───────┬────────┘             │
│             │                                │                       │
│        alarmes                          aggregated                   │
│             ▼                           findings                     │
│  ┌──────────────────┐                       │                       │
│  │   SNS → Slack    │                       ▼                       │
│  │   SNS → Email    │               ┌──────────────┐               │
│  │   SNS → PagerDuty│               │  EventBridge │               │
│  └──────────────────┘               │  (automação) │               │
│                                      └──────┬───────┘               │
│  ┌──────────────┐                           │                       │
│  │  AWS Config   │ ─── non-compliant ───────┤                       │
│  │  (7 rules)    │     findings             │                       │
│  └──────────────┘                           ▼                       │
│                                      ┌──────────────┐               │
│  ┌──────────────┐                    │   Lambda     │               │
│  │  GuardDuty   │ ── threats ──────→ │  (auto-fix)  │               │
│  │  (detection) │   HIGH severity    │  Block IP    │               │
│  └──────────────┘                    │  Re-attach WAF│              │
│                                      └──────────────┘               │
│  ┌───────────────────────────────────────┐                          │
│  │  Governança                            │                          │
│  │  ├── Organizations + SCPs (preventivo) │                          │
│  │  ├── Firewall Manager (WAF central)    │                          │
│  │  ├── Secrets Manager (credenciais)     │                          │
│  │  └── IAM Least Privilege (4 níveis)    │                          │
│  └───────────────────────────────────────┘                          │
└──────────────────────────────────────────────────────────────────────┘
```

### Validação

```bash
# 1. CloudTrail — verificar que está capturando eventos CloudFront
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudfront.amazonaws.com \
  --max-results 5 \
  --query 'Events[].{Time:EventTime, Name:EventName, User:Username}' \
  --output table

# 2. Config — verificar compliance das distributions
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name "cloudfront-associated-with-waf" \
  --compliance-types NON_COMPLIANT \
  --output table

# 3. Security Hub — verificar findings CloudFront
aws securityhub get-findings \
  --filters '{"ResourceType":[{"Value":"AwsCloudFrontDistribution","Comparison":"EQUALS"}]}' \
  --query 'Findings[].{Title:Title,Severity:Severity.Label,Status:Compliance.Status}' \
  --output table

# 4. GuardDuty — verificar detector ativo
aws guardduty list-detectors --output text

# 5. IAM — verificar policies
aws iam list-policies --scope Local \
  --query 'Policies[?contains(PolicyName,`CloudFront`)].PolicyName' \
  --output table

# 6. Secrets Manager — verificar secrets
aws secretsmanager list-secrets \
  --filters Key=name,Values=cloudfront \
  --query 'SecretList[].{Name:Name,Rotation:RotationEnabled}' \
  --output table

# 7. Firewall Manager — verificar policies
aws fms list-policies \
  --query 'PolicyList[].{Name:PolicyName,Type:ResourceType}' \
  --output table
```

```
✅ CloudTrail capturando API calls de CloudFront
✅ 5 queries Athena para auditoria de mudanças
✅ Alarmes para mudanças críticas (WAF removido, distribution deletada)
✅ 7 Config Rules para compliance contínua
✅ Auto-remediation: re-attach WAF se removido
✅ Security Hub habilitado com 13 controls CloudFront
✅ EventBridge alertando em findings CRITICAL/HIGH
✅ GuardDuty detectando ameaças e bloqueando IPs via Lambda→WAF
✅ 4 níveis IAM (ReadOnly, Operator, Admin, SecurityAdmin)
✅ Secrets Manager com rotação para signing keys
✅ Firewall Manager aplicando WAF em todas as contas
✅ 4 SCPs de governança (require WAF, logging, TLS moderno, tags)
✅ Lambda auto-block de IPs suspeitos (GuardDuty→WAF)
```

### O Que Aprendemos

| Serviço | Papel no Security Mesh | Quando Usar |
|---------|----------------------|-------------|
| **CloudTrail** | Auditoria — quem fez o quê, quando, de onde | Sempre (obrigatório para compliance) |
| **Config** | Compliance — estado dos recursos vs regras | Sempre (7 rules para CloudFront) |
| **Security Hub** | Postura — dashboard centralizado de findings | Sempre (13 controls CloudFront) |
| **GuardDuty** | Detecção — ameaças ativas e IPs suspeitos | Sempre (integra com WAF auto-block) |
| **Firewall Manager** | Governança — WAF centralizado cross-account | Multi-conta (10+ contas) |
| **IAM** | Acesso — quem pode fazer o quê | Sempre (4 níveis de permissão) |
| **Secrets Manager** | Credenciais — keys, tokens, headers seguros | Sempre que tiver secrets |
| **SCPs** | Guardrails — o que é PROIBIDO fazer | Organizations (compliance forçada) |
| **EventBridge** | Automação — conecta findings a ações | Sempre (auto-remediation) |

> **💡 Expert Tip:** O serviço mais subestimado nesta lista é **AWS Config com auto-remediation**. Configurar uma regra que detecta "CloudFront sem WAF" e automaticamente re-associa o WAF é a diferença entre "detectamos a falha em 5 minutos" e "o WAF ficou desligado por 3 dias porque ninguém percebeu". A combinação Config Rules + SSM Automation + EventBridge cria um sistema imunológico para sua infraestrutura — ele detecta e cura problemas automaticamente. Em auditorias SOC 2, essa automação é ouro: demonstra não só que você tem controles, mas que eles são self-healing.

---

**Próximo:** [Módulo 06 — Integrações com Serviços AWS →](modulo-06-integracoes-aws.md)
