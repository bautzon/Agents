# Copilot Studio Client - Authentication & Message Flow

This document explains the traffic flow for authentication and message exchange in the Copilot Studio .NET client application.

## Architecture Overview

```mermaid
graph TB
    subgraph "Client Application"
        A[.NET Console App] --> B[ChatConsoleService]
        A --> C[Program.cs]
        C --> D[SampleConnectionSettings]
        C --> E[HttpClient Factory]
        E --> F[AddTokenHandler]
        B --> G[CopilotClient]
    end
    
    subgraph "Authentication Layer"
        F --> H[MSAL PublicClientApplication]
        H --> I[Token Cache]
        H --> J[Interactive Auth Browser]
    end
    
    subgraph "Microsoft Services"
        K[Azure AD/Entra ID]
        L[Power Platform API]
        M[Copilot Studio Agent]
    end
    
    F --> K
    J --> K
    G --> L
    L --> M
```

## Detailed Authentication Flow

```mermaid
sequenceDiagram
    participant App as .NET Console App
    participant Handler as AddTokenHandler
    participant MSAL as MSAL Library
    participant Cache as Token Cache
    participant Browser as User Browser
    participant AAD as Azure AD/Entra ID
    participant PP as Power Platform API
    participant CS as Copilot Studio Agent

    Note over App,CS: Application Startup & Authentication
    
    App->>Handler: Initialize with settings
    Note right of Handler: AppClientId: 54a843e4-d4e7-4766-ad57-73c12b7ccd5b<br/>TenantId: 2db89727-a51e-4d8d-9f51-8a68603cf7c0<br/>UseS2SConnection: false
    
    App->>MSAL: Create PublicClientApplication
    Note right of MSAL: Authority: AzureAdMyOrg<br/>RedirectUri: http://localhost<br/>Scopes: https://api.powerplatform.com/.default
    
    MSAL->>Cache: Check for cached token
    Cache-->>MSAL: No valid token found
    
    MSAL->>Browser: Launch interactive authentication
    Browser->>AAD: User credentials + App consent
    AAD-->>Browser: Authorization code
    Browser-->>MSAL: Authorization code
    
    MSAL->>AAD: Exchange code for tokens
    AAD-->>MSAL: Access token + Refresh token
    
    MSAL->>Cache: Store tokens securely
    Note right of Cache: Location: mcs_client_console/TokenCache
    
    Handler-->>App: Authentication complete
```

## Message Exchange Flow

```mermaid
sequenceDiagram
    participant User as Console User
    participant App as ChatConsoleService
    participant Client as CopilotClient
    participant Handler as AddTokenHandler
    participant Cache as Token Cache
    participant PP as Power Platform API
    participant CS as Copilot Studio Agent

    Note over User,CS: Message Exchange Process
    
    App->>Client: StartConversationAsync()
    Client->>Handler: Send HTTP Request
    
    Handler->>Cache: Get cached access token
    Cache-->>Handler: Valid access token
    
    Handler->>PP: HTTPS Request with Bearer token
    Note right of PP: Environment: dc65cbb8-d34e-e4fa-8fd3-03f79392775a<br/>Schema: cre49_agent1_rdmyhe
    
    PP->>CS: Route to agent instance
    CS-->>PP: Initial conversation setup
    PP-->>Client: Activities stream
    
    Client-->>App: Yield activities
    App-->>User: Display agent response
    
    loop Message Loop
        User->>App: Type message
        App->>Client: SendMessageAsync(message)
        Client->>Handler: Send message request
        Handler->>Cache: Get token (refresh if needed)
        Cache-->>Handler: Valid token
        Handler->>PP: Message with Bearer auth
        PP->>CS: Forward message to agent
        CS-->>PP: Agent response activities
        PP-->>Client: Response activities
        Client-->>App: Yield response activities
        App-->>User: Display agent response
    end
```

## Configuration Flow

```mermaid
flowchart TD
    A[appsettings.json] --> B{UseS2SConnection?}
    B -->|false| C[User Interactive Auth]
    B -->|true| D[Service-to-Service Auth]
    
    C --> E[AddTokenHandler]
    D --> F[AddTokenHandlerS2S]
    
    E --> G[MSAL PublicClient]
    F --> H[MSAL ConfidentialClient]
    
    G --> I[Interactive Browser Login]
    H --> J[Client Credentials Flow]
    
    subgraph "Your Working Configuration"
        K[UseS2SConnection: false<br/>AppClientId: 54a843e4-...<br/>TenantId: 2db89727-...<br/>EnvironmentId: dc65cbb8-...<br/>SchemaName: cre49_agent1_rdmyhe]
    end
    
    K --> C
```

## Network Traffic Overview

```mermaid
graph LR
    subgraph "Authentication Traffic"
        A1[HTTPS to login.microsoftonline.com] --> A2[OAuth2/OIDC Flow]
        A2 --> A3[Access Token Response]
    end
    
    subgraph "API Traffic"
        B1[HTTPS to api.powerplatform.com] --> B2[Bearer Token Auth]
        B2 --> B3[Message Exchange]
        B3 --> B4[Activity Responses]
    end
    
    subgraph "Local Storage"
        C1[Token Cache File] --> C2[Encrypted Token Storage]
        C2 --> C3[Token Refresh Logic]
    end
    
    A3 --> B1
    A3 --> C1
    C3 --> B1
```

## Key Components

### Current Working Configuration:
- **Authentication Mode**: User Interactive (`UseS2SConnection: false`)
- **App Registration**: `54a843e4-d4e7-4766-ad57-73c12b7ccd5b`
- **Tenant**: `2db89727-a51e-4d8d-9f51-8a68603cf7c0`
- **Environment**: `dc65cbb8-d34e-e4fa-8fd3-03f79392775a`
- **Agent Schema**: `cre49_agent1_rdmyhe`

### Authentication Flow:
1. **MSAL Setup**: Creates PublicClientApplication with app registration details
2. **Token Cache**: Checks for existing valid tokens in local encrypted cache
3. **Interactive Auth**: Opens browser for user login if no valid token exists
4. **Token Storage**: Securely stores access and refresh tokens locally
5. **Request Authentication**: Adds Bearer token to all API requests

### Message Flow:
1. **Connection**: Establishes connection to Copilot Studio agent via Power Platform API
2. **Conversation Start**: Initiates conversation and receives initial activities
3. **Message Loop**: Continuous exchange of user messages and agent responses
4. **Activity Processing**: Handles various activity types (text, cards, etc.)

### Security:
- **Token Encryption**: Tokens stored encrypted in local file system
- **HTTPS Only**: All network communication over secure HTTPS
- **Token Refresh**: Automatic token refresh using stored refresh tokens
- **Scope Limitation**: Tokens scoped specifically to Power Platform API access
