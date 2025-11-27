flowchart TB
    subgraph "User Channels"
        PMS[Practice Management Software]
        PatientApp[Patient App / myGov]
    end

    subgraph "Azure Secure Boundary (DTS)"
        direction TB
        
        %% Gateway Layer
        APIM[Azure API Management <br/>(Gateway & WAF)]
        
        %% Identity Layer
        Auth[Azure AD B2C <br/>(Identity Broker)]

        %% Compute Layer (Microservices)
        subgraph "Core Services"
            TokenService[Token Engine <br/>(Azure Functions)]
            NotifyService[Notification Engine <br/>(Logic Apps)]
            ConsentService[Consent Registry <br/>(Azure Functions)]
        end

        %% Data Layer
        subgraph "Data & Security"
            Redis[(Azure Redis Cache <br/> Hot Storage)]
            Cosmos[(Azure Cosmos DB <br/> Consent Store)]
            HSM[Azure Cloud HSM <br/> Key Management]
            Ledger[(Azure SQL Ledger <br/> Immutable Audit)]
            ServiceBus[Azure Service Bus <br/> Async Messaging]
        end
    end

    subgraph "External Systems"
        PRODA[PRODA Identity Provider]
        MBS[Services Australia Payment Engine <br/> (ECLIPSE)]
        OrgReg[Organisation Register <br/> (MyMedicare)]
    end

    %% Connections
    PMS -->|HTTPS/API| APIM
    PatientApp -->|HTTPS/API| APIM
    
    APIM -->|Validate Auth| Auth
    Auth <-->|Federate| PRODA
    
    APIM -->|Route Request| TokenService
    APIM -->|Route Request| ConsentService
    
    %% Token Logic
    TokenService -->|Sign/Verify| HSM
    TokenService -->|Log Transaction| Ledger
    TokenService -->|Check Status| Redis
    
    %% Consent Logic
    ConsentService -->|Read/Write| Cosmos
    ConsentService -->|Cache Hot Data| Redis
    ConsentService <-->|Verify Eligibility| OrgReg
    
    %% Notification Logic
    TokenService -->|Publish Event| ServiceBus
    ServiceBus -->|Trigger| NotifyService
    NotifyService -.->|SMS/Push| PatientApp
    
    %% Payment Integration
    MBS -.->|Pre-Payment Check| APIM
