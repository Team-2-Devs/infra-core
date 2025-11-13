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

<!-- ---

## Codebase Architecture

![Codebase Diagram](docs/images/codebase-architecture.jpg) -->

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
7. As work progresses, the orchestrator publishes events:  
    - `AnalysisStarted` to the **analysis.started** fanout exchange.  
    - `AnalysisCompleted` to the **analysis.completed** fanout exchange.  
8. **GraphGateway** listens to those event exchanges and bridges them to GraphQL subscriptions:  
    - `AnalysisStarted` → triggers `onAnalysisStarted` subscription.  
    - `AnalysisCompleted` → triggers `onAnalysisCompleted` subscription.  
9. **ClientConsole** receives the subscription events over the existing WebSocket and filters by `correlationId` in the event payload to correlate to its request.  

---

## Configuration
If values and secrets are not yet created:
1. Navigate to `infra-core/k8s`.  
2. Download `generate-secrets-and-values.sh` to that directory.  
3. Run:
   ```bash
   bash generate-secrets-and-values.sh
   ```

## Environment Setup
**Windows:**
1. Install WSL (if you don't already have it):
   ```bash
   wsl --install -d Ubuntu
   ```
   Run all following commands inside WSL (Ubuntu). Start it by running wsl in PowerShell.
2. Install kind, kubectl, and helm:
   ```bash
   curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.30.0/kind-linux-amd64
   chmod +x ./kind
   sudo mv ./kind /usr/local/bin/kind
   ```
   ```bash
   curl -LO "https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/kubectl
   ```
   ```bash
   curl -LO https://get.helm.sh/helm-v3.15.2-linux-amd64.tar.gz
   tar -xf helm-v3.15.2-linux-amd64.tar.gz
   sudo mv linux-amd64/helm /usr/local/bin/helm
   ```
3. Add kong and oauth2-proxy helm charts to helm repo:
   ```bash
   helm repo add kong https://charts.konghq.com
   ```
   ```bash
   helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
   ```

**MacOS:**
1. Install Homebrew (if you don't already have it):
2. Install kind, kubectl, and helm:
   ```bash
   brew install kind
   brew install kubectl
   brew install helm
   ```
3. Add kong and oauth2-proxy helm charts to helm repo:
   ```bash
   helm repo add kong https://charts.konghq.com
   ```
   ```bash
   helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
   ```
4. Install Go:
   ```bash
   brew install go
   ```
5. Install cloud-provider-kind:
   ```bash
   go install sigs.k8s.io/cloud-provider-kind@latest
   ```
6. Move cloud-provider-kind to /usr/local/bin for global use:
   ```bash
   sudo mv ~/go/bin/cloud-provider-kind /usr/local/bin/cloud-provider-kind
   ```

---

## Run Kubernetes Cluster Locally
**Windows:**
1. Ensure Docker Desktop is running.  
2. Determine the path to `infra-core/k8s`. You will need this when switching into WSL.  
3. Open PowerShell and start WASL:
   ```bash
   wsl
   ```
   All remaining commands must be executed inside the WSL terminal.  
4. Navigate to the `infra-core/k8s` directory inside WSL:
   ```bash
   cd "/mnt/<drive>/<path>/infra-core/k8s"
   ```
5. Create the Kubernetes cluster:
   ```bash
   kind create cluster --config kind-calico.yaml
   ```
6. Apply the calico CNI manifest:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml
   ```
7. Install required Helm charts:
   ```bash
   helm install oauth2-proxy oauth2-proxy/oauth2-proxy --namespace api-gateway --create-namespace -f helm/oauth2-proxy/values.yaml  
   helm install kong kong/kong --namespace api-gateway --create-namespace -f helm/kong/values.yaml  
   helm install rabbitmq oci://registry-1.docker.io/cloudpirates/rabbitmq --namespace messaging --create-namespace -f helm/rabbitmq/values.yaml  
   ```
8. Apply local manifests:
   ```bash
   kubectl apply -f helm/oauth2-proxy/secret.yaml  
   kubectl apply -f helm/rabbitmq/network-policy.yaml  
   kubectl apply -f helm/oauth2-proxy/network-policy.yaml  
   kubectl apply -f helm/kong/network-policy.yaml  
   kubectl apply -f graph-gateway
   kubectl apply -f svc-analysis-orchestrator
   ```
   *If errors occur in the last two commands, simply run them again. Sometimes manifests are applied out of order, causing temporary startup errors.*
9. Verify that all pods are running:
   ```bash
   kubectl get pods -A
   ```
10. Port-forward Kong to your local machine:
    ```bash
    kubectl port-forward -n api-gateway svc/kong-proxy 8080:80
    ```
    This makes Kong accessible at `http://localhost:8080` on your Windows machine, even though Kong is running inside the Kubernetes cluster.

**MacOS:**
1. Ensure Docker Desktop is running.  
2. Navigate to the `infra-core/k8s` directory.  
3. Create the Kubernetes cluster:
   ```bash
   kind create cluster --config kind-calico.yaml
   ```
4. Apply the calico CNI manifest:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml
   ```
7. Install required Helm charts:
   ```bash
   helm install oauth2-proxy oauth2-proxy/oauth2-proxy --namespace api-gateway --create-namespace -f helm/oauth2-proxy/values.yaml  
   helm install kong kong/kong --namespace api-gateway --create-namespace -f helm/kong/values.yaml  
   helm install rabbitmq oci://registry-1.docker.io/cloudpirates/rabbitmq --namespace messaging --create-namespace -f helm/rabbitmq/values.yaml  
   ```
8. Apply local manifests:
   ```bash
   kubectl apply -f helm/oauth2-proxy/secret.yaml  
   kubectl apply -f helm/rabbitmq/network-policy.yaml  
   kubectl apply -f helm/oauth2-proxy/network-policy.yaml  
   kubectl apply -f helm/kong/network-policy.yaml  
   kubectl apply -f graph-gateway
   kubectl apply -f svc-analysis-orchestrator
   ```
   *If errors occur in the last two commands, simply run them again. Sometimes manifests are applied out of order, causing temporary startup errors.*
9. Verify that all pods are running:
   ```bash
   kubectl get pods -A
   ```
10. Open a new terminal at the project root and start cloud-provider-kind:
    ```bash
    sudo go/bin/cloud-provider-kind
    ```
    This assigns an external IP to the Kong LoadBalancer service that allows it to be accessed from outside the Kubernetes cluster.

**Cleanup:**
Remove the kind cluster:
```bash
kind delete cluster
```

---

## Run Console Client
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
<!-- - **Docker / Docker Compose** for local orchestration -->