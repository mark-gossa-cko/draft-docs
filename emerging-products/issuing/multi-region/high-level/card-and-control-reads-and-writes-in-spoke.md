# Card and control reads and writes in the spoke

Spoke (e.g. APAC):
* CMS API
* Controls API
* Vault

Hub (Europe): All other services

```mermaid
sequenceDiagram

participant Client
participant Merchant

box darkgreen Spoke (e.g. APAC)
    participant CMSAPI1 as CMS API
    participant ControlsAPI1 as Controls API
    participant Vault1 as Vault
end

box darkblue Hub (Europe)
    participant MIP
    participant Bifrost
    participant AuthAPI as Authorization API
    participant CAP
    participant CMSAPI2 as CMS API
    participant ControlsAPI2 as Controls API
    participant Vault2 as Vault
    participant ServiceLocator as Service Locator
    participant Finance as Finance Authorization <br>Lambda
    participant HSMProxy as HSM Proxy
    participant ProcessingHSM as Processing HSM
    participant FIAPI as FI API
end

alt Client active in spoke
    Client->>+CMSAPI1: Create card with controls and gets credentials
    CMSAPI1->>+Vault1: Store card and get credentials
    Vault1->>Vault1: Encrypt card (KMS)
    Vault1->>Vault1: Generate card credentials (no HSM)
    Vault1->>Vault1: Store card in DynamoDB
    Vault1->>-CMSAPI1: Card credentials
    CMSAPI1->>CMSAPI1: Store card in DynamoDB
    CMSAPI1->>+ControlsAPI1: Create controls
    ControlsAPI1->>ControlsAPI1: Store controls in DynamoDB
    ControlsAPI1->>-CMSAPI1: 201 Created
    CMSAPI1->>-Client: Card details
else Client active in hub (failover)
    Client->>+CMSAPI2: Create card with controls and gets credentials
    CMSAPI2->>+Vault2: Store card and get credentials
    Vault2->>Vault2: Encrypt card (KMS)
    Vault2->>Vault2: Generate card credentials (no HSM)
    Vault2->>Vault2: Store card in DynamoDB
    Vault2->>-CMSAPI2: Card credentials
    CMSAPI2->>CMSAPI2: Store card in DynamoDB
    CMSAPI2->>+ControlsAPI2: Create controls
    ControlsAPI2->>ControlsAPI2: Store controls in DynamoDB
    ControlsAPI2->>-CMSAPI2: 201 Created
    CMSAPI2->>-Client: Card details
end

Client->>Merchant: Sends card details
Merchant->>MIP: Performs authorization
MIP->>+Bifrost: Process authorization

Bifrost->>+ServiceLocator: Get authoritative vault URL
ServiceLocator->>-Bifrost: Authoritative vault URL
alt Client active in spoke
    Bifrost->>+Vault1: Verify card
    Vault1->>-Bifrost: Card verification response
else Client active in hub (failover)
    Bifrost->>+Vault2: Verify card
    Vault2->>-Bifrost: Card verification response
end

Bifrost->>+AuthAPI: Process authorization
AuthAPI->>+ServiceLocator: Get authoritative CMS API URL
ServiceLocator->>-AuthAPI: Authoritative CMS API URL
alt Client active in spoke
    AuthAPI->>+CMSAPI1: Get card
    CMSAPI1->>-AuthAPI: Card data
else Client active in hub (failover)
    AuthAPI->>+CMSAPI2: Get card
    CMSAPI2->>-AuthAPI: Card data
end

AuthAPI->>+HSMProxy: Generate authorization code
HSMProxy->>+ProcessingHSM: Generate authorization code
ProcessingHSM->>-HSMProxy: Authorization code
HSMProxy->>-AuthAPI: Authorization code
AuthAPI->>+CAP: Assess authorization

CAP->>+ServiceLocator: Get authoritative CMS API URL
ServiceLocator->>-CAP: Authoritative CMS API URL
alt Client active in spoke
    CAP->>+CMSAPI1: Get card
    CMSAPI1->>-CAP: Card data
else Client active in hub (failover)
    CAP->>+CMSAPI2: Get card
    CMSAPI2->>-CAP: Card data
end

CAP->>+ServiceLocator: Get authoritative Controls API URL
ServiceLocator->>-CAP: Authoritative Controls API URL
alt Client active in spoke
    CAP->>+ControlsAPI1: Get Controls and control profile targets
    ControlsAPI1->>-CAP: Controls and control profile targets
else Client active in hub (failover)
    CAP->>+ControlsAPI2: Get Controls and control profile targets
    ControlsAPI2->>-CAP: Controls and control profile targets
end

CAP->>CAP: Assess authorization
CAP->>-AuthAPI: Assessment result
AuthAPI->>+Finance: Balance basic checks and update balance
Finance->>+FIAPI: Balance basic checks
FIAPI->>-Finance: Check result
Finance->>FIAPI: Async balance update
Finance->>-AuthAPI: Check result
AuthAPI->>-Bifrost: Response
Bifrost->>-MIP: Response
```