# Módulo 07 — Federation: Social & Enterprise

> **Nível:** 300 (Advanced)
> **Tempo Total Estimado:** 10-14 horas de labs
> **Custo Estimado:** ~$1
> **Objetivo do Módulo:** Implementar federation com social providers (Google, Facebook, Apple) e enterprise providers (SAML 2.0 com Okta/Azure AD, OIDC), attribute mapping e linking de users federados com perfis locais.

---

## Mapa do Módulo

```mermaid
graph TB
    MOD[Módulo 07<br/>Federation]

    MOD --> D37[D.37 Google Login]
    MOD --> D38[D.38 Facebook Login]
    MOD --> D39[D.39 Apple Login]
    MOD --> D40_F[D.40 SAML 2.0<br/>Okta / Azure AD]
    MOD --> D41_F[D.41 OIDC Federation<br/>Custom Provider]
    MOD --> D42_F[D.42 Attribute Mapping<br/>+ User Linking]

    style MOD fill:#e94560,color:#fff
    style D37 fill:#0f3460,color:#fff
    style D40_F fill:#533483,color:#fff
    style D42_F fill:#16213e,color:#fff
```

---

## Desafio 37: Google Login

> **Level:** 300 | **Tempo:** 90 min | **Custo:** $0

### Fluxo

```mermaid
sequenceDiagram
    participant U as Usuário
    participant APP as App
    participant COG as Cognito
    participant G as Google OAuth2

    U->>APP: Click "Login with Google"
    APP->>COG: Redirect /authorize?identity_provider=Google
    COG->>G: OAuth2 redirect
    G->>U: Google consent screen
    U->>G: Allow
    G->>COG: Authorization code + user info
    COG->>COG: Create/update user profile
    COG-->>APP: Redirect callback with Cognito code
    APP->>COG: Exchange code for tokens
    COG-->>APP: ID + Access + Refresh tokens
```

### Passo a Passo

```
1. Google Cloud Console → APIs & Services → Credentials
2. Create OAuth 2.0 Client ID
3. Authorized redirect URI: https://YOUR_DOMAIN.auth.REGION.amazoncognito.com/oauth2/idpresponse
4. Copiar Client ID e Client Secret
```

```hcl
resource "aws_cognito_identity_provider" "google" {
  user_pool_id  = aws_cognito_user_pool.main.id
  provider_name = "Google"
  provider_type = "Google"

  provider_details = {
    client_id        = var.google_client_id
    client_secret    = var.google_client_secret
    authorize_scopes = "openid email profile"
  }

  attribute_mapping = {
    email    = "email"
    name     = "name"
    picture  = "picture"
    username = "sub"
  }
}

# App Client: incluir Google como IdP
resource "aws_cognito_user_pool_client" "spa" {
  # ...
  supported_identity_providers = ["COGNITO", "Google"]
}
```

---

## Desafio 40: SAML 2.0 — Okta / Azure AD

> **Level:** 300 | **Tempo:** 120 min | **Custo:** $0

### Arquitetura

```mermaid
graph LR
    U[Usuário Enterprise] --> COG[Cognito<br/>Managed Login]
    COG -->|SAML Request| OKTA[Okta / Azure AD<br/>SAML IdP]
    OKTA -->|SAML Assertion| COG
    COG -->|JWT Tokens| APP[App]

    style COG fill:#0f3460,color:#fff
    style OKTA fill:#533483,color:#fff
```

```hcl
resource "aws_cognito_identity_provider" "okta" {
  user_pool_id  = aws_cognito_user_pool.main.id
  provider_name = "Okta"
  provider_type = "SAML"

  provider_details = {
    MetadataURL = "https://empresa.okta.com/app/xxx/sso/saml/metadata"
  }

  attribute_mapping = {
    email      = "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"
    name       = "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name"
    "custom:department" = "department"
  }
}
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| Social login | Google, Facebook, Apple — OAuth2/OIDC |
| SAML 2.0 | Enterprise SSO (Okta, Azure AD, OneLogin) |
| Attribute mapping | Mapear claims do IdP para atributos do Cognito |
| User linking | Vincular user federado a perfil local existente |
| MetadataURL | XML com configuração do IdP (endpoints, certificates) |

> **💡 Expert Tip:** Para B2B SaaS, SAML federation é obrigatório. Clientes enterprise exigem "login com meu Azure AD/Okta". Cognito suporta múltiplos SAML IdPs — cada cliente empresa pode ter seu próprio IdP configurado. Use attribute mapping para extrair `department`, `role` ou `tenant` do assertion SAML.

---

## Resumo

```
✅ D.37-42: Social (Google/Facebook/Apple) + Enterprise (SAML/OIDC) + Attribute Mapping
Próximo: Módulo 08 — Frontend Integration
```

**Próximo:** [Módulo 08 — Frontend Integration →](modulo-08-frontend-integration.md)
