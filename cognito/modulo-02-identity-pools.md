# Módulo 02 — Identity Pools

> **Nível:** 100-200 (Foundational/Intermediate)
> **Tempo Total Estimado:** 10-14 horas de labs
> **Custo Estimado:** ~$0
> **Objetivo do Módulo:** Dominar Cognito Identity Pools — gerar credenciais AWS temporárias para usuários autenticados, role mapping por grupo, acesso user-scoped a S3 e DynamoDB, e developer-authenticated identities.

---

## Mapa do Módulo

```mermaid
graph TB
    MOD[Módulo 02<br/>Identity Pools]

    MOD --> D7[D.7 Primeiro Identity Pool<br/>AWS Credentials]
    MOD --> D8[D.8 Auth vs Unauth<br/>Roles]
    MOD --> D9[D.9 User Pool +<br/>Identity Pool]
    MOD --> D10[D.10 Role Mapping<br/>por Grupo]
    MOD --> D11[D.11 User-Scoped<br/>S3 Access]
    MOD --> D12[D.12 User-Scoped<br/>DynamoDB]

    style MOD fill:#e94560,color:#fff
    style D7 fill:#0f3460,color:#fff
    style D10 fill:#533483,color:#fff
    style D11 fill:#16213e,color:#fff
```

---

## User Pool vs Identity Pool

```mermaid
graph LR
    subgraph UP ["User Pool (AuthN)"]
        UP_IN[Login] --> UP_OUT[JWT Tokens<br/>ID + Access + Refresh]
    end

    subgraph IP ["Identity Pool (AuthZ)"]
        IP_IN[JWT Token] --> IP_OUT[AWS Credentials<br/>AccessKeyId + SecretKey<br/>+ SessionToken]
    end

    UP_OUT -->|"Troca token"| IP_IN
    IP_OUT --> AWS[S3, DynamoDB,<br/>Lambda, etc.]

    style UP fill:#0f3460,color:#fff
    style IP fill:#533483,color:#fff
    style AWS fill:#16213e,color:#fff
```

| Aspecto | User Pool | Identity Pool |
|---------|-----------|--------------|
| **Pergunta** | Quem é você? | O que você pode fazer na AWS? |
| **Output** | JWT Tokens | AWS Credentials (STS) |
| **Uso** | Autenticação | Acesso direto a serviços AWS |
| **Obrigatório?** | Sim (se usar Cognito para auth) | Não (só se precisar de AWS credentials) |

---

## Desafio 7: Primeiro Identity Pool

> **Level:** 100 | **Tempo:** 90 min | **Custo:** $0

### Objetivo

Criar Identity Pool, trocar token do User Pool por credenciais AWS temporárias e acessar S3.

```hcl
resource "aws_cognito_identity_pool" "main" {
  identity_pool_name               = "app-identity-pool"
  allow_unauthenticated_identities = false

  cognito_identity_providers {
    client_id               = aws_cognito_user_pool_client.spa.id
    provider_name           = aws_cognito_user_pool.main.endpoint
    server_side_token_check = true
  }
}

# Roles
resource "aws_cognito_identity_pool_roles_attachment" "main" {
  identity_pool_id = aws_cognito_identity_pool.main.id

  roles = {
    "authenticated"   = aws_iam_role.cognito_authenticated.arn
    "unauthenticated" = aws_iam_role.cognito_unauthenticated.arn
  }
}

# IAM Role para authenticated users
resource "aws_iam_role" "cognito_authenticated" {
  name = "CognitoAuthenticatedUser"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = "cognito-identity.amazonaws.com" }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "cognito-identity.amazonaws.com:aud" = aws_cognito_identity_pool.main.id
        }
        "ForAnyValue:StringLike" = {
          "cognito-identity.amazonaws.com:amr" = "authenticated"
        }
      }
    }]
  })
}

# Policy: user-scoped S3 access
resource "aws_iam_role_policy" "cognito_s3" {
  name = "UserScopedS3"
  role = aws_iam_role.cognito_authenticated.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
      Resource = "arn:aws:s3:::app-user-files/$${cognito-identity.amazonaws.com:sub}/*"
    }]
  })
}
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| Identity Pool | Troca JWT tokens por AWS credentials temporárias |
| Authenticated role | IAM role para users logados |
| Unauthenticated role | IAM role para users anônimos (ex: download público) |
| `cognito-identity.amazonaws.com:sub` | User ID único — usado para scoped access |
| User-scoped S3 | Cada user acessa apenas `user-files/SEU_ID/*` |

> **💡 Expert Tip:** Identity Pool é necessário APENAS quando o frontend precisa acessar serviços AWS diretamente (S3 upload, DynamoDB read). Se toda a lógica passa por API Gateway + Lambda, você NÃO precisa de Identity Pool — o User Pool sozinho é suficiente.

---

## Resumo do Módulo 02

```
┌──────────────────────────────────────────────────────────────┐
│  ✅ D.7: Identity Pool + AWS Credentials                     │
│  ✅ D.8: Authenticated vs Unauthenticated Roles              │
│  ✅ D.9: User Pool + Identity Pool Integration               │
│  ✅ D.10: Role Mapping por Grupo                             │
│  ✅ D.11: User-Scoped S3 Access                              │
│  ✅ D.12: User-Scoped DynamoDB Access                        │
│  Próximo: Módulo 03 — OAuth2, OIDC & Managed Login          │
└──────────────────────────────────────────────────────────────┘
```

**Próximo:** [Módulo 03 — OAuth2, OIDC & Managed Login →](modulo-03-oauth2-managed-login.md)
