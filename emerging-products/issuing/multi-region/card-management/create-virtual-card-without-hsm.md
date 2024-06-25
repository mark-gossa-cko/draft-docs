# Streamlined Virtual Card Creation (no HSM)

```mermaid
sequenceDiagram

actor Client
participant Access
participant Config API
participant Config API DynamoDB
participant Controls API
participant Controls API DynamoDB
participant Vault
participant Vault DynamoDB
participant KMS
participant Secrets Manager
participant SNS

Client      ->> Access: Get token
activate Access #000000
Access      ->> Client: Token
deactivate Access

Client->>+Config API: [POST] /cards
Config API->>Config API: Check token

opt Create card with controls
Config API->>Controls API: Validate controls [POST] /can-add
activate Controls API
Controls API->>Config API: 
deactivate Controls API
end

Config API->>Config API DynamoDB: Get Card Product
activate Config API DynamoDB
Config API DynamoDB ->> Config API: Card Product
deactivate Config API DynamoDB
Config API->>Config API: Generate expiry date
Config API->>Config API: Generate revocation date
Config API->>Config API DynamoDB: Get cardholder
activate Config API DynamoDB
Config API DynamoDB ->> Config API: Cardholder
deactivate Config API DynamoDB
Config API->>Config API: Generate PAN

opt Card Product usage milestone reached
Config API->>SNS: Publish card.product.usage.milestone.reached event
end

Config API->>Vault: Store card [POST] /api/v1/cards
activate Vault
Vault->>Vault: Generate hash of PAN
Vault->>Vault DynamoDB: Get card by hash of PAN
activate Vault DynamoDB
Vault DynamoDB->>Vault: Card (if exists)
deactivate Vault DynamoDB

alt Card already exists in Vault
Vault->>Config API: Return Vault ID and hashed PAN
else
Vault->>KMS: Encrypt PAN
activate KMS
KMS->>Vault: Encrypetd PAN
deactivate KMS
Vault->>Vault DynamoDB: Store card with hashed PAN and encrypted PAN
Vault->>Config API: Return Vault ID and hashed PAN
deactivate Vault
end

Config API->>Config API DynamoDB: Store card

opt Create card with controls
Config API->>Controls API: Add controls [POST] /batch
activate Controls API
Controls API->> Controls API DynamoDB: Add controls
Controls API->>Config API: 201 CREATED
deactivate Controls API
end

opt Get credentials
Config API->>Vault: Get credentials (PAN, CVC2)
activate Vault
Vault->>Vault DynamoDB: Get card
activate Vault DynamoDB
Vault DynamoDB->>Vault: Card
deactivate Vault DynamoDB
Vault->>KMS: Decrypt PAN
activate KMS
KMS->>Vault: Decrypted PAN
deactivate KMS

opt Generate CVC2
Vault->>+Secrets Manager: Get encrypted DES key
Secrets Manager->>-Vault: Encrypted DES key
Vault->>+KMS: Decrypt DES key
KMS->>-Vault: Decrypted DES key
Vault->>Vault: Generate CVC2
end
Vault->>Config API: Credentials
deactivate Vault
end

Config API->>SNS: Publish virtual.card.created event

Config API->>-Client: 201 CREATED
```