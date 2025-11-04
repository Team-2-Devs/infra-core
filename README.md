# Trackunit Project
Mobile application that uses AI to recognize construction machinery from images and provide relevant information to users.

---

## System Architecture

### Core Services

- **[GraphGateway](https://github.com/team-2-devs/graph-gateway)**  
  Microservice serving as the API facade and application gateway for the system. Provides a GraphQL API with queries, mutations, and subscriptions. Publishes `RequestAnalysis` commands to RabbitMQ and forwards `analysis/started` and `analysis/completed` events to the onAnalysisStarted and onAnalysisCompleted GraphQL subscription fields.

- **[SvcAnalysisOrchestrator](https://github.com/team-2-devs/svc-analysis-orchestrator)**  
  Microservice that orchestrates analysis. Consumes `RequestAnalysis` commands from RabbitMQ and publishes `analysis.started` and `analysis.completed` event exchanges.

### Shared Components

- **[Messaging](https://github.com/team-2-devs/messaging)**  
  NuGet package containing shared `MessageContracts` and reusable RabbitMQ publisher interfaces and implementations.

### Infrastructure

- **InfraCore**  
  Docker Compose-based orchestration environment bundling all services and infrastructure components, including **Kong Gateway**, **oauth2-proxy**, and **RabbitMQ**.

- **Kong Gateway**  
  API Gateway running in db-less mode. Routes `/graphql` requests to **oauth2-proxy**, which validates incoming tokens before forwarding them to **GraphGateway**.

- **oauth2-proxy**  
  Authentication proxy that validates Bearer tokens (JWT) issued by Microsoft Entra ID.  
  Only requests with valid tokens are forwarded to **GraphGateway**.

- **RabbitMQ**  
  Message broker for asynchronous communication.

---

## Codebase Architecture

![Codebase Diagram](docs/images/codebase-architecture.jpg)

---

## System Diagram (Simplified)
```text
                     ┌──────────────────────┐
                     │     ClientConsole    │
                     │     (console app)    │
                     └────────┬─────────────┘
      HTTP/WS :8000 (GraphQL) │ ↑ 
                call mutation │ ╎ receive subscription events via WebSocket :8000
       requestAnalysis(input) ↓ ╎ (AnalysisStarted / AnalysisCompleted)
                      ┌─────────┴─────────┐
                      │   Kong Gateway    │
                      │   (API Gateway)   │
                      └───────┬───────────┘
        HTTP :4180 (internal) │ ↑ 
      foward mutation request │ ╎ forward subscription events via WebSocket :8000
       requestAnalysis(input) ↓ ╎ (AnalysisStarted / AnalysisCompleted)
                    ┌───────────┴───────────────┐
                    │       oauth2-proxy        │
                    │ (JWT validation via OIDC) │
                    └─────────┬─────────────────┘
        HTTP :8080 (internal) │ ↑ 
     receive mutation request │ ╎ publish subscription events via WebSocket :8000
       requestAnalysis(input) ↓ ╎ (AnalysisStarted / AnalysisCompleted)
                 ┌──────────────┴────────────────┐
                 │        GraphGateway           │
                 │      (GraphQL Server)         │
                 │  - handles requestAnalysis    │
                 │  - RabbitToSubscriptions      │
                 │    bridges RabbitMQ → GraphQL │
                 └─────────────┬─────────────────┘
                    AMQP :5672 │ △ 
               publish command │ ╎ receive events
             (RequestAnalysis) │ ╎ (AnalysisStarted / AnalysisCompleted)
                               ▼ ╎
         ┌───────────────────────┴────────────────────────────────────────────┐
         │                 RabbitMQ                                           │
         │  Exchanges: analysis.commands   (direct)                           │
         │             analysis.started    (fanout)                           │
         │             analysis.completed  (fanout)                           │
         │  Queues:    orchestrator.analysis.commands ← bind request.analysis │
         │             graph.subs.started             ← bind (fanout)         │
         │             graph.subs.completed           ← bind (fanout)         │
         └────────────────────┬───────────────────────────────────────────────┘
                   AMQP :5672 │ △
              receive command │ ╎ publish events
            (RequestAnalysis) │ ╎ (AnalysisStarted / AnalysisCompleted)
                              ▼ ╎
                 ┌──────────────┴───────────────┐
                 │    SvcAnalysisOrchestrator   │
                 │  - handles RequestAnalysis   │
                 │  - publishes start/completed │
                 └──────────────────────────────┘
```
>Note: Simplified for clarity. Detailed flow described below.

---

## Authentication Flow

1. **Client** authenticates with **MSAL** (device code) and acquires an **access token** for the API:
   - **Scope requested:** `api://{API_CLIENT_ID}/analysis` → token `scp` includes `analysis`.
   - **Token audience:** `api://{API_CLIENT_ID}` or `{API_CLIENT_ID}`
2. **Kong** proxies `/graphql` to **oauth2-proxy**.
3. **oauth2-proxy** validates the Bearer token against Microsoft Entra ID:
   - Verifies issuer and audience
   - If valid, forwards the request upstream
4. **GraphGateway** re-validates the JWT and enforces the GraphQL authorization policy `RequireApiScope` on protected fields.

---

## Analysis Flow
1. **ClientConsole** opens a WebSocket to `/graphql` via Kong Gateway → oauth2-proxy → GraphGateway.  
2. **ClientConsole** starts a subscription filtered by `correlationId` or prepares to resubscribe once it has one.  
3. **ClientConsole** calls the `requestAnalysis` GraphQL mutation, passing an `objectKey`.  
4. **GraphGateway** receives the mutation, generates a `CorrelationId`, and returns it as `AnalysisRequestPayload(correlationId)` in the mutation payload.  
5. **GraphGateway** publishes a `RequestAnalysis` command to RabbitMQ (exchange: `analysis.commands`, routing key: `analysis.request`).  
6. **SvcAnalysisOrchestrator** consumes the `RequestAnalysis` command from its queue (orchestrator.analysis.commands) and simulates processing.  
7. As work progresses, the orchestrator publishes **events**:  
    - `AnalysisStarted` to the **analysis.started** fanout exchange.  
    - `AnalysisCompleted` to the **analysis.completed** fanout exchange.  
8. **GraphGateway** listens to those event exchanges and bridges them to GraphQL subscriptions:  
    - `AnalysisStarted` → triggers `onAnalysisStarted` subscription.  
    - `AnalysisCompleted` → triggers `onAnalysisCompleted` subscription.  
9. **ClientConsole** receives the subscription events over the existing WebSocket and filters by `correlationId` in the event payload to correlate to its request.  

---

## Configuration

All sensitive values come from **environment variables** (via `.env`).  
Copy the provided `.env.template` to `.env` and fill in the values:

```bash
cp .env.template .env
```

---

## Run Locally

1. **Build & start** the stack:
   From the **infra-core** repository root:
   ```bash
   docker compose up -d --build
   ```

2. **Health checks**:
   - **RabbitMQ UI:** http://localhost:15672 (user/pass from `.env`)
   - **Kong:** http://localhost:8000
   - **oauth2-proxy ping:** http://127.0.0.1:4180/ping (should return “OK”)
   - **GraphGateway:**
     - GraphQL: http://localhost:8090/graphql
     - Health: http://localhost:8090/health/ready
   
3. **Run the console client** (from host):
   From the `client-console` repository root:
   ```bash
   dotnet run
   ```
   - Follow the MSAL device-code prompt.
   - Use the console to request an analysis and observe live updates.

---

## Messaging: Exchanges & Routing

- **`analysis.commands` (direct)**
  - **Routing key:** `analysis.request` (`Routes.RequestAnalysis`)
  - **Queue:** `orchestrator.analysis.commands`
  - **Publisher:** GraphGateway (mutation `RequestAnalysisAsync`)
  - **Consumer:** SvcAnalysisOrchestrator (worker `RabbitAnalysisWorker`)

- **`analysis.started` (fanout)**
  - **Publisher:** SvcAnalysisOrchestrator
  - **Consumers:** GraphGateway (bridges to GraphQL subscriptions via `RabbitToSubscriptions`, queue `graph.subs.started`)

- **`analysis.completed` (fanout)**
  - **Publisher:** SvcAnalysisOrchestrator
  - **Consumers:** GraphGateway (bridges to GraphQL subscriptions via `RabbitToSubscriptions`, queue `graph.subs.completed`)

---

## GraphQL Schema (SDL)

```graphql
schema {
  query: Query
  mutation: Mutation
  subscription: Subscription
}

type AnalysisCompleted {
  correlationId: String!
  objectKey: String!
  success: Boolean!
}

type AnalysisRequestPayload {
  correlationId: String!
}

type AnalysisStarted {
  correlationId: String!
  objectKey: String!
}

type Mutation {
  requestAnalysis(input: AnalysisRequestInput!): AnalysisRequestPayload!
    @authorize(policy: "RequireApiScope")
    @cost(weight: "10")
}

type Query {
  ping: String!
}

type Subscription {
  onAnalysisStarted: AnalysisStarted! @authorize(policy: "RequireApiScope")
  onAnalysisCompleted: AnalysisCompleted! @authorize(policy: "RequireApiScope")
}

input AnalysisRequestInput {
  objectKey: String!
}
```

---

## Tech & Libraries

- **.NET 9 / C#**, **HotChocolate GraphQL**
- **RabbitMQ**
- **Kong (OSS)** in **db-less** mode
- **oauth2-proxy** for JWT validation at the edge
- **Microsoft Entra ID** via **MSAL (device code flow)**
- **Docker / Docker Compose** for local orchestration