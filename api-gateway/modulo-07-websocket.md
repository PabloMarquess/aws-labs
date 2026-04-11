# Módulo 07 — WebSocket API

> **Nível:** 300 (Advanced)
> **Tempo Total Estimado:** 10-14 horas de labs
> **Custo Estimado:** ~$2-5 (API calls + DynamoDB)
> **Objetivo do Módulo:** Dominar WebSocket APIs — rotas ($connect, $disconnect, $default, custom), connection management com DynamoDB, broadcasting, construir um chat completo e autenticação em WebSocket.

---

## Mapa do Módulo

```mermaid
graph TB
    MOD[Módulo 07<br/>WebSocket API]

    MOD --> D40[D.40 Primeira WebSocket<br/>$connect, $disconnect]
    MOD --> D41[D.41 Custom Routes<br/>Route Selection]
    MOD --> D42[D.42 Connection Management<br/>DynamoDB]
    MOD --> D43[D.43 Broadcasting<br/>Send to All]
    MOD --> D44[D.44 Chat App<br/>Completo]
    MOD --> D45[D.45 Auth &<br/>Rate Limiting]

    style MOD fill:#e94560,color:#fff
    style D40 fill:#0f3460,color:#fff
    style D42 fill:#533483,color:#fff
    style D44 fill:#16213e,color:#fff
```

---

## Desafio 40: Primeira WebSocket API

> **Level:** 300 | **Tempo:** 90 min | **Custo:** ~$0

### Objetivo

Criar uma WebSocket API com as rotas fundamentais: `$connect`, `$disconnect` e `$default`.

### Como WebSocket Funciona no API Gateway

```mermaid
sequenceDiagram
    participant C as Cliente (wscat)
    participant WS as WebSocket API
    participant L1 as Lambda: onConnect
    participant L2 as Lambda: onMessage
    participant L3 as Lambda: onDisconnect

    C->>WS: WebSocket upgrade request
    WS->>L1: $connect route
    L1-->>WS: 200 OK
    WS-->>C: Connection established<br/>connectionId: abc123

    C->>WS: {"action":"sendMessage","text":"Hello"}
    WS->>L2: $default (ou custom route)
    L2-->>WS: Response

    C->>WS: Close connection
    WS->>L3: $disconnect route
    L3-->>WS: 200 OK
```

### Passo a Passo

```bash
# 1. Criar WebSocket API
WS_API_ID=$(aws apigatewayv2 create-api \
  --name "chat-ws-api" \
  --protocol-type WEBSOCKET \
  --route-selection-expression '$request.body.action' \
  --query 'ApiId' --output text)

echo "WebSocket API ID: $WS_API_ID"

# 2. Criar integrations para cada rota
CONNECT_INT=$(aws apigatewayv2 create-integration \
  --api-id "$WS_API_ID" \
  --integration-type AWS_PROXY \
  --integration-uri "arn:aws:apigateway:$REGION:lambda:path/2015-03-31/functions/arn:aws:lambda:$REGION:$ACCOUNT_ID:function:ws-connect/invocations" \
  --query 'IntegrationId' --output text)

DISCONNECT_INT=$(aws apigatewayv2 create-integration \
  --api-id "$WS_API_ID" \
  --integration-type AWS_PROXY \
  --integration-uri "arn:aws:apigateway:$REGION:lambda:path/2015-03-31/functions/arn:aws:lambda:$REGION:$ACCOUNT_ID:function:ws-disconnect/invocations" \
  --query 'IntegrationId' --output text)

DEFAULT_INT=$(aws apigatewayv2 create-integration \
  --api-id "$WS_API_ID" \
  --integration-type AWS_PROXY \
  --integration-uri "arn:aws:apigateway:$REGION:lambda:path/2015-03-31/functions/arn:aws:lambda:$REGION:$ACCOUNT_ID:function:ws-default/invocations" \
  --query 'IntegrationId' --output text)

# 3. Criar rotas
aws apigatewayv2 create-route --api-id "$WS_API_ID" \
  --route-key '$connect' --target "integrations/$CONNECT_INT"

aws apigatewayv2 create-route --api-id "$WS_API_ID" \
  --route-key '$disconnect' --target "integrations/$DISCONNECT_INT"

aws apigatewayv2 create-route --api-id "$WS_API_ID" \
  --route-key '$default' --target "integrations/$DEFAULT_INT"

# 4. Deploy
aws apigatewayv2 create-stage --api-id "$WS_API_ID" \
  --stage-name prod --auto-deploy

WS_URL="wss://$WS_API_ID.execute-api.$REGION.amazonaws.com/prod"
echo "WebSocket URL: $WS_URL"
```

### Lambda: Connection Handler

```python
# ws-connect/handler.py
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('ws-connections')

def handler(event, context):
    connection_id = event['requestContext']['connectionId']

    table.put_item(Item={
        'connectionId': connection_id,
        'connectedAt': event['requestContext']['connectedAt'],
    })

    return {'statusCode': 200}


# ws-disconnect/handler.py
def handler(event, context):
    connection_id = event['requestContext']['connectionId']
    table.delete_item(Key={'connectionId': connection_id})
    return {'statusCode': 200}


# ws-default/handler.py
def handler(event, context):
    connection_id = event['requestContext']['connectionId']
    body = json.loads(event.get('body', '{}'))

    # Enviar resposta de volta ao cliente
    apigw = boto3.client('apigatewaymanagementapi',
        endpoint_url=f"https://{event['requestContext']['domainName']}/{event['requestContext']['stage']}")

    apigw.post_to_connection(
        ConnectionId=connection_id,
        Data=json.dumps({'message': f'Received: {body}'}).encode()
    )

    return {'statusCode': 200}
```

### Testar com wscat

```bash
# Instalar wscat
npm install -g wscat

# Conectar
wscat -c "$WS_URL"

# Enviar mensagem
> {"action":"sendMessage","text":"Hello WebSocket!"}
# Recebe: {"message": "Received: {...}"}

# Desconectar: Ctrl+C
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| `$connect` | Rota chamada quando cliente conecta |
| `$disconnect` | Rota chamada quando cliente desconecta |
| `$default` | Rota chamada quando nenhuma custom route faz match |
| `connectionId` | ID único da conexão — usado para enviar mensagens de volta |
| `route-selection-expression` | Campo do body que determina qual rota usar |
| `post_to_connection` | API para enviar mensagem do server para o client |

---

## Desafio 42: Connection Management com DynamoDB

> **Level:** 300 | **Tempo:** 90 min | **Custo:** ~$0.50

### Arquitetura

```mermaid
graph TB
    subgraph Clients
        C1[Client 1<br/>conn: abc]
        C2[Client 2<br/>conn: def]
        C3[Client 3<br/>conn: ghi]
    end

    C1 --> WS[WebSocket API]
    C2 --> WS
    C3 --> WS

    WS --> L[Lambda]
    L --> DDB[(DynamoDB<br/>ws-connections<br/>PK: connectionId)]

    L -->|"post_to_connection<br/>(broadcast)"| WS

    style WS fill:#0f3460,color:#fff
    style DDB fill:#533483,color:#fff
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| Connection table | DynamoDB com connectionId como PK |
| TTL | Limpar conexões stale automaticamente |
| Broadcast | Scan connections + post_to_connection para cada |
| Stale connections | GoneException quando connection não existe mais |

---

## Desafio 44: Chat Application Completo

> **Level:** 300 | **Tempo:** 120 min | **Custo:** ~$1

### Arquitetura do Chat

```mermaid
graph TB
    subgraph Clients
        U1[User A]
        U2[User B]
        U3[User C]
    end

    U1 --> WS[WebSocket API]
    U2 --> WS
    U3 --> WS

    WS --> CONNECT[Lambda<br/>onConnect]
    WS --> MSG[Lambda<br/>sendMessage]
    WS --> JOIN[Lambda<br/>joinRoom]
    WS --> DC[Lambda<br/>onDisconnect]

    CONNECT --> CONN_DB[(DynamoDB<br/>connections)]
    MSG --> CONN_DB
    MSG --> MSG_DB[(DynamoDB<br/>messages)]
    JOIN --> CONN_DB

    MSG -->|"Broadcast to room"| WS

    style WS fill:#0f3460,color:#fff
    style CONN_DB fill:#533483,color:#fff
    style MSG_DB fill:#16213e,color:#fff
```

### O Que Aprendemos

| Conceito | Detalhe |
|----------|---------|
| Custom routes | `sendMessage`, `joinRoom` — baseado em `action` field |
| Rooms | GSI no DynamoDB por `roomId` para broadcast seletivo |
| Message history | Tabela separada com mensagens persistidas |
| Error handling | `GoneException` quando client desconectou |

> **💡 Expert Tip:** WebSocket API do API Gateway tem limite de 128 KB por frame e idle timeout de 10 minutos. Para manter conexões vivas, implemente ping/pong a cada 5 minutos. Para aplicações com milhares de conexões simultâneas, considere usar DynamoDB Streams + Lambda para fan-out em vez de scan + post_to_connection sequencial.

---

## Resumo do Módulo 07

```
┌──────────────────────────────────────────────────────────────┐
│               MÓDULO 07 — CONQUISTAS                          │
│                                                               │
│  ✅ Desafio 40: WebSocket Fundamentals                       │
│  ✅ Desafio 41: Custom Routes                                │
│  ✅ Desafio 42: Connection Management (DynamoDB)             │
│  ✅ Desafio 43: Broadcasting                                 │
│  ✅ Desafio 44: Chat App Completo                            │
│  ✅ Desafio 45: Auth & Rate Limiting                         │
│                                                               │
│  Próximo: Módulo 08 — Performance & Cost                     │
└──────────────────────────────────────────────────────────────┘
```

**Próximo:** [Módulo 08 — Performance & Cost →](modulo-08-performance-cost.md)
