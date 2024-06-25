# Card and control reads and writes in the spoke

Spoke (e.g. APAC):
* CMS API
* Controls API
* Vault

Hub (Europe): All other services

Only normal operation shown (with CMS services in the spoke region, i.e. not a failover scenario)

```mermaid
sequenceDiagram

participant Client
participant Merchant

box darkgreen Spoke (e.g. APAC)
    participant CMSAPIInSpoke as CMS API
    participant ControlsAPIInSpoke as Controls API
    participant VaultInSpoke as Vault
    participant CMSDataAggregatorInSpoke as CMS Data Aggregator
end

box darkblue Hub (Europe)
    participant CMSDataAggregatorInHub as CMS Data Aggregator
    participant MIP
    participant Bifrost
    participant AuthAPI as Authorization API
    participant CAP
    participant CMSAPIInHub as CMS API
    participant ControlsAPIInHub as Controls API
    participant VaultInHub as Vault
    participant ServiceLocator as Service Locator
    participant Finance as Finance Authorization <br>Lambda
    participant HSMProxy as HSM Proxy
    participant ProcessingHSM as Processing HSM
    participant FIAPI as FI API
end

Client->>+CMSAPIInSpoke: Create card with controls and gets credentials
CMSAPIInSpoke->>+VaultInSpoke: Store card and get credentials
VaultInSpoke->>VaultInSpoke: Encrypt card (KMS)
VaultInSpoke->>VaultInSpoke: Generate card credentials (no HSM)
VaultInSpoke->>VaultInSpoke: Store card in DynamoDB
VaultInSpoke->>-CMSAPIInSpoke: Card credentials
CMSAPIInSpoke->>CMSAPIInSpoke: Store card in DynamoDB
CMSAPIInSpoke->>+ControlsAPIInSpoke: Create controls
ControlsAPIInSpoke->>ControlsAPIInSpoke: Store controls in DynamoDB
ControlsAPIInSpoke->>-CMSAPIInSpoke: InHub0InSpoke Created
CMSAPIInSpoke->>-Client: Card details

Client->>Merchant: Sends card details
Merchant->>MIP: Performs authorization
MIP->>+Bifrost: Process authorization

Bifrost->>+ServiceLocator: Get authoritative vault URL (OnlineAuthorizationFlow: true)
ServiceLocator->>-Bifrost: Authoritative vault URL (points to CMS Data Aggregator)

Bifrost->>+CMSDataAggregatorInHub: Verify card
# Todo: CMS DA in spoke gets card verification result, cards data and control data
CMSDataAggregatorInHub->>+CMSDataAggregatorInSpoke: Verify card, get card and control data
CMSDataAggregatorInSpoke->>+VaultInSpoke: Verify card
VaultInSpoke->>-CMSDataAggregatorInSpoke: Verify card response
CMSDataAggregatorInSpoke->>+CMSAPIInSpoke: Get card
CMSAPIInSpoke->>-CMSDataAggregatorInSpoke: Card data
CMSDataAggregatorInSpoke->>+ControlsAPIInSpoke: Get controls for card (card, card product, control profiles)
ControlsAPIInSpoke->>-CMSDataAggregatorInSpoke: Control data
CMSDataAggregatorInSpoke->>-CMSDataAggregatorInHub: Card verification, card data, control data
CMSDataAggregatorInHub->>CMSDataAggregatorInHub: Caches data
CMSDataAggregatorInHub->>-Bifrost: Card verification response

Bifrost->>+AuthAPI: Process authorization
AuthAPI->>+ServiceLocator: Get authoritative CMS API URL (OnlineAuthorizationFlow: true)
ServiceLocator->>-AuthAPI: Authoritative CMS API URL (points to CMS Data Aggregator)
AuthAPI->>+CMSDataAggregatorInHub: Get card
CMSDataAggregatorInHub->>-AuthAPI: Card data

AuthAPI->>+HSMProxy: Generate authorization code
HSMProxy->>+ProcessingHSM: Generate authorization code
ProcessingHSM->>-HSMProxy: Authorization code
HSMProxy->>-AuthAPI: Authorization code
AuthAPI->>+CAP: Assess authorization

CAP->>+ServiceLocator: Get authoritative CMS API URL (OnlineAuthorizationFlow: true)
ServiceLocator->>-CAP: Authoritative CMS API URL (points to CMS Data Aggregator)
CAP->>+CMSDataAggregatorInHub: Get card
CMSDataAggregatorInHub->>-CAP: Card data

CAP->>+ServiceLocator: Get authoritative Controls API URL (OnlineAuthorizationFlow: true)
ServiceLocator->>-CAP: Authoritative Controls API URL (points to CMS Data Aggregator)
CAP->>+CMSDataAggregatorInHub: Get Controls and control profile targets
CMSDataAggregatorInHub->>-CAP: Controls and control profile targets

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