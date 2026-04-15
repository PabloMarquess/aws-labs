# Módulo 03 — OAuth2, OIDC & Managed Login

> **Nível:** 200 (Intermediate)
> **Tempo Total Estimado:** 10-14 horas de labs
> **Custo Estimado:** ~$0
> **Objetivo do Módulo:** Dominar OAuth2 flows, OIDC tokens, Managed Login pages, custom domain, branding, Resource Servers e custom scopes.

---

## Mapa do Módulo

```mermaid
graph TB
    MOD[Módulo 03<br/>OAuth2 & Managed Login]

    MOD --> D13[D.13 OAuth2 Flows<br/>Authorization Code + PKCE]
    MOD --> D14[D.14 App Clients<br/>Public vs Confidential]
    MOD --> D15[D.15 Managed Login<br/>Hosted UI Domain]
    MOD --> D16[D.16 Branding<br/>Custom CSS + Logo]
    MOD --> D17[D.17 Client Credentials<br/>Machine-to-Machine]
    MOD --> D18[D.18 Resource Servers<br/>Custom Scopes]

    style MOD fill:#e94560,color:#fff
    style D13 fill:#0f3460,color:#fff
    style D15 fill:#533483,color:#fff
    style D17 fill:#16213e,color:#fff
```

---

## Desafio 13: OAuth2 Flows — Authorization Code + PKCE

> **Level:** 200 | **Tempo:** 90 min | **Custo:** $0

### Authorization Code Flow

```mermaid
sequenceDiagram
    participant U as Usuário
    participant APP as SPA/Mobile
    participant COG as Cognito<br/>Managed Login
    participant API as API Backend

    APP->>APP: 1. Gerar code_verifier + code_challenge (PKCE)
    APP->>COG: 2. Redirect /authorize?response_type=code<br/>&code_challenge=Z

    COG->>U: 3. Login page
    U->>COG: 4. Email + Password + MFA

    COG->>APP: 5. Redirect callback?code=AUTH_CODE

    APP->>COG: 6. POST /token<br/>grant_type=authorization_code<br/>&code=AUTH_CODE&code_verifier=V

    COG-->>APP: 7. {id_token, access_token, refresh_token}

    APP->>API: 8. Authorization: Bearer access_token
```

### OAuth2 Flows Comparativo

| Flow | Quando Usar | Seguro? |
|------|------------|---------|
| **Authorization Code + PKCE** | SPAs, Mobile Apps | Mais seguro |
| **Authorization Code** (sem PKCE) | Server-side apps (com client_secret) | Seguro |
| **Client Credentials** | Machine-to-machine (sem user) | Seguro |
| **Implicit** (deprecated) | Nunca mais usar | Inseguro |

```bash
# Configurar domínio
aws cognito-idp create-user-pool-domain \
  --user-pool-id "$POOL_ID" \
  --domain "app-auth-lab"

# URL de login
echo "https://app-auth-lab.auth.$REGION.amazoncognito.com/login?client_id=$CLIENT_ID&response_type=code&scope=openid+email+profile&redirect_uri=http://localhost:3000/callback"

# Trocar code por tokens
curl -s -X POST \
  "https://app-auth-lab.auth.$REGION.amazoncognito.com/oauth2/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code&client_id=$CLIENT_ID&code=AUTH_CODE&redirect_uri=http://localhost:3000/callback"
```

### Terraform

```hcl
resource "aws_cognito_user_pool_domain" "main" {
  domain       = "app-auth-${var.environment}"
  user_pool_id = aws_cognito_user_pool.main.id
}

resource "aws_cognito_user_pool_client" "spa" {
  name         = "spa-client"
  user_pool_id = aws_cognito_user_pool.main.id

  generate_secret                      = false
  allowed_oauth_flows                  = ["code"]
  allowed_oauth_scopes                 = ["openid", "email", "profile"]
  allowed_oauth_flows_user_pool_client = true
  supported_identity_providers         = ["COGNITO"]

  callback_urls = ["http://localhost:3000/callback", "https://app.meusite.com/callback"]
  logout_urls   = ["http://localhost:3000/logout", "https://app.meusite.com/logout"]
}
```

> **💡 Expert Tip:** SEMPRE use Authorization Code + PKCE para SPAs. O flow Implicit é deprecated pela OAuth 2.1 spec. Com PKCE, mesmo que alguém intercepte o authorization code, não consegue trocar por tokens sem o code_verifier.

---

## Desafio 17: Client Credentials — Machine-to-Machine

> **Level:** 200 | **Tempo:** 60 min | **Custo:** $0

### Fluxo M2M

```mermaid
sequenceDiagram
    participant SVC as Service A
    participant COG as Cognito
    participant API as Service B

    SVC->>COG: POST /token<br/>grant_type=client_credentials<br/>&scope=api/read api/write
    COG-->>SVC: {access_token}
    SVC->>API: Authorization: Bearer access_token
```

```hcl
resource "aws_cognito_resource_server" "api" {
  user_pool_id = aws_cognito_user_pool.main.id
  identifier   = "api"
  name         = "App API"

  scope {
    scope_name        = "read"
    scope_description = "Read access"
  }
  scope {
    scope_name        = "write"
    scope_description = "Write access"
  }
}

resource "aws_cognito_user_pool_client" "m2m" {
  name                                 = "service-a-m2m"
  user_pool_id                         = aws_cognito_user_pool.main.id
  generate_secret                      = true
  allowed_oauth_flows                  = ["client_credentials"]
  allowed_oauth_scopes                 = ["api/read", "api/write"]
  allowed_oauth_flows_user_pool_client = true
  supported_identity_providers         = []
}
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| Authorization Code + PKCE | Flow mais seguro para SPAs/mobile |
| Client Credentials | M2M — sem user, apenas service identity |
| Resource Server | Define custom OAuth2 scopes |
| Managed Login | Hosted UI pages gerenciadas pela AWS |
| Custom domain | Branding do domínio de login |

---

## Resumo do Módulo 03

```
┌──────────────────────────────────────────────────────────────┐
│  ✅ D.13: Authorization Code + PKCE                          │
│  ✅ D.14: Public vs Confidential Clients                     │
│  ✅ D.15: Managed Login Domain                               │
│  ✅ D.16: Branding (CSS + Logo)                              │
│  ✅ D.17: Client Credentials (M2M)                           │
│  ✅ D.18: Resource Servers + Custom Scopes                   │
│  Próximo: Módulo 04 — Lambda Triggers                        │
└──────────────────────────────────────────────────────────────┘
```

**Próximo:** [Módulo 04 — Lambda Triggers →](modulo-04-lambda-triggers.md)
