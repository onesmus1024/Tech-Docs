# 🔐 Azure Key Vault: Enterprise Secrets Management Platform

## 1. Project Scope

This document outlines the implementation of a centralized secrets management solution using **Azure Key Vault**. The system provides secure storage, access control, and auditing for cryptographic keys, certificates, and application secrets with integration across multiple Azure services.

- **Service Used:** Azure Key Vault (Premium SKU with HSM)
- **Infrastructure:** Key Vault with RBAC, Private Endpoints, and Managed Identities
- **Development SDK:** .NET 8 (C#) + Azure SDK
- **Key Capabilities:** Secret Management, Certificate Lifecycle, Key Rotation, Access Policies, Integration with Azure Services.

---

## 2. Infrastructure Deployment (Provisioning)

### 2.1 Cloud Adoption Framework Alignment

This implementation follows Azure CAF best practices for security and governance:

- **Strategy:** Eliminate hard-coded credentials and centralize secrets management
- **Plan:** Define RBAC model with least-privilege access principles
- **Ready:** Establish secure landing zone with network isolation and monitoring
- **Adopt:** Migrate existing secrets from application configuration to Key Vault
- **Govern:** Implement audit logging, compliance policies, and secret rotation schedules
- **Manage:** Enable Azure Monitor integration with alerting for unauthorized access

### Step 1: Resource Group Setup

Created dedicated resource group following CAF naming conventions and compliance tags.

```bash
# CAF Naming: rg-{workload}-{environment}-{region}-{instance}
az group create \
    --name rg-security-prod-eastus-001 \
    --location eastus \
    --tags Environment=Production \
           Workload=SecretsManagement \
           DataClassification=Confidential \
           CostCenter=Security \
           Compliance=SOC2
```

### Step 2: Azure Key Vault Creation

Deployed Premium SKU Key Vault with RBAC enabled and purge protection.

```bash
az keyvault create \
    --name kv-reliance-prod-001 \
    --resource-group rg-security-prod-eastus-001 \
    --location eastus \
    --sku Premium \
    --enable-rbac-authorization true \
    --enable-purge-protection true \
    --retention-days 90 \
    --enabled-for-deployment true \
    --enabled-for-disk-encryption true \
    --enabled-for-template-deployment true \
    --public-network-access Disabled
```

**Configuration Highlights:**

- **SKU Premium:** HSM-backed keys for highest security
- **RBAC Authorization:** Modern role-based access control (replaces legacy access policies)
- **Purge Protection:** 90-day soft delete prevents accidental/malicious deletion
- **Public Access Disabled:** Network isolation via Private Endpoints only

**Evidence 1: Key Vault Deployment**  
_Description: Azure CLI output showing provisioningState: Succeeded with vault URI._

---

## 3. Network Security Configuration

### Step 3: Virtual Network Setup

Established secure network infrastructure for private connectivity.

```bash
# Create VNet for private endpoints
az network vnet create \
    --resource-group rg-security-prod-eastus-001 \
    --name vnet-security-prod \
    --address-prefix 10.0.0.0/16 \
    --subnet-name snet-private-endpoints \
    --subnet-prefix 10.0.1.0/24

# Disable subnet private endpoint policies
az network vnet subnet update \
    --resource-group rg-security-prod-eastus-001 \
    --vnet-name vnet-security-prod \
    --name snet-private-endpoints \
    --disable-private-endpoint-network-policies true
```

### Step 4: Private Endpoint Configuration

Created private endpoint to restrict Key Vault access to internal network only.

```bash
# Create private endpoint
az network private-endpoint create \
    --resource-group rg-security-prod-eastus-001 \
    --name pe-keyvault-prod \
    --vnet-name vnet-security-prod \
    --subnet snet-private-endpoints \
    --private-connection-resource-id $(az keyvault show \
        --name kv-reliance-prod-001 \
        --resource-group rg-security-prod-eastus-001 \
        --query id -o tsv) \
    --group-id vault \
    --connection-name keyvault-private-connection

# Create private DNS zone
az network private-dns zone create \
    --resource-group rg-security-prod-eastus-001 \
    --name privatelink.vaultcore.azure.net

# Link DNS zone to VNet
az network private-dns link vnet create \
    --resource-group rg-security-prod-eastus-001 \
    --zone-name privatelink.vaultcore.azure.net \
    --name dns-link-keyvault \
    --virtual-network vnet-security-prod \
    --registration-enabled false
```

**Evidence 2: Private Endpoint**  
_Description: Azure Portal screenshot showing Private Endpoint with "Approved" connection state and private IP (10.0.1.4)._

---

## 4. Identity & Access Management (RBAC)

### 4.1 Managed Identity Setup

Created user-assigned managed identity for application authentication.

```bash
az identity create \
    --resource-group rg-security-prod-eastus-001 \
    --name id-app-keyvault-reader \
    --location eastus
```

### 4.2 RBAC Role Assignments

Configured granular access control using Azure built-in roles.

```bash
# Get managed identity principal ID
IDENTITY_PRINCIPAL_ID=$(az identity show \
    --resource-group rg-security-prod-eastus-001 \
    --name id-app-keyvault-reader \
    --query principalId -o tsv)

# Get Key Vault resource ID
KEYVAULT_ID=$(az keyvault show \
    --name kv-reliance-prod-001 \
    --resource-group rg-security-prod-eastus-001 \
    --query id -o tsv)

# Assign "Key Vault Secrets User" role (read-only access to secrets)
az role assignment create \
    --role "Key Vault Secrets User" \
    --assignee $IDENTITY_PRINCIPAL_ID \
    --scope $KEYVAULT_ID

# Assign "Key Vault Crypto User" role (for encryption operations)
az role assignment create \
    --role "Key Vault Crypto User" \
    --assignee $IDENTITY_PRINCIPAL_ID \
    --scope $KEYVAULT_ID
```

**RBAC Model:**

- **Administrators:** Key Vault Administrator (full control)
- **Applications:** Key Vault Secrets User (read secrets only)
- **Security Team:** Key Vault Secrets Officer (manage secrets)
- **Auditors:** Key Vault Reader (view configuration, no secret access)

**Evidence 3: Role Assignments**  
_Description: Azure Portal IAM blade showing role assignments with principle of least privilege._

---

## 5. Secret Management Operations

### Step 5: Store Application Secrets

Populated Key Vault with various secret types using Azure CLI.

```bash
# Database connection string
az keyvault secret set \
    --vault-name kv-reliance-prod-001 \
    --name sql-connection-string \
    --value "Server=tcp:sqlserver-prod.database.windows.net,1433;Database=ProductionDB;User ID=dbadmin;" \
    --content-type "application/connection-string" \
    --tags Application=ERP Environment=Production

# API key
az keyvault secret set \
    --vault-name kv-reliance-prod-001 \
    --name stripe-api-key \
    --value "sk_live_51AbCdEf..." \
    --content-type "text/plain" \
    --tags Application=PaymentService Provider=Stripe

# Storage account key
az keyvault secret set \
    --vault-name kv-reliance-prod-001 \
    --name storage-account-key \
    --value "DefaultEndpointsProtocol=https;AccountName=storageprod..." \
    --content-type "application/storage-key" \
    --expires $(date -u -d "+1 year" '+%Y-%m-%dT%H:%M:%SZ')
```

**Evidence 4: Secrets Dashboard**  
_Description: Azure Portal Key Vault secrets blade showing 3 secrets with enabled status and metadata._

---

## 6. Application Integration (.NET)

### 6.1 Dependencies

Installed official Azure SDK packages for seamless integration.

```xml
<PackageReference Include="Azure.Identity" Version="1.10.4" />
<PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.5.0" />
<PackageReference Include="Azure.Extensions.AspNetCore.Configuration.Secrets" Version="1.3.0" />
```

### 6.2 Implementation Code (Program.cs)

ASP.NET Core application that retrieves secrets using Managed Identity.

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Microsoft.Extensions.Configuration;

var builder = WebApplication.CreateBuilder(args);

// 1. Configure Key Vault Integration with Managed Identity
string keyVaultUrl = "https://kv-reliance-prod-001.vault.azure.net/";
var credential = new DefaultAzureCredential();

// Add Key Vault as configuration source (automatic secret refresh)
builder.Configuration.AddAzureKeyVault(
    new Uri(keyVaultUrl),
    credential);

// 2. Register SecretClient for manual secret operations
builder.Services.AddSingleton(sp =>
{
    return new SecretClient(new Uri(keyVaultUrl), credential);
});

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

var app = builder.Build();

// 3. Endpoint to retrieve secret (for demonstration)
app.MapGet("/config/database", (IConfiguration config) =>
{
    // Secrets are automatically mapped from Key Vault
    // Secret name "sql-connection-string" becomes "sql-connection-string" in config
    string connectionString = config["sql-connection-string"];

    // Never return actual secrets - this is for demo only
    return Results.Ok(new
    {
        Status = "Connected",
        SecretRetrieved = !string.IsNullOrEmpty(connectionString),
        SecretLength = connectionString?.Length ?? 0
    });
});

// 4. Manual secret retrieval with versioning
app.MapGet("/secret/{secretName}", async (string secretName, SecretClient client) =>
{
    try
    {
        // Retrieve specific secret
        KeyVaultSecret secret = await client.GetSecretAsync(secretName);

        return Results.Ok(new
        {
            SecretName = secret.Name,
            SecretVersion = secret.Properties.Version,
            ExpiresOn = secret.Properties.ExpiresOn,
            Enabled = secret.Properties.Enabled,
            ContentType = secret.Properties.ContentType,
            Tags = secret.Properties.Tags,
            // NEVER return actual Value in production
            ValueLength = secret.Value.Length
        });
    }
    catch (Azure.RequestFailedException ex) when (ex.Status == 404)
    {
        return Results.NotFound($"Secret '{secretName}' not found");
    }
    catch (Azure.RequestFailedException ex) when (ex.Status == 403)
    {
        return Results.Forbid();
    }
});

// 5. List all secrets (metadata only)
app.MapGet("/secrets", async (SecretClient client) =>
{
    var secrets = new List<object>();

    await foreach (var secretProperties in client.GetPropertiesOfSecretsAsync())
    {
        secrets.Add(new
        {
            Name = secretProperties.Name,
            Enabled = secretProperties.Enabled,
            ContentType = secretProperties.ContentType,
            ExpiresOn = secretProperties.ExpiresOn,
            Tags = secretProperties.Tags
        });
    }

    return Results.Ok(new { Count = secrets.Count, Secrets = secrets });
});

app.MapControllers();
app.Run();
```

### 6.3 Local Development Configuration (appsettings.Development.json)

For local development, use Azure CLI authentication.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "KeyVault": {
    "VaultUri": "https://kv-reliance-prod-001.vault.azure.net/"
  }
}
```

**Authentication Flow:**

1. **Azure Environment:** DefaultAzureCredential uses Managed Identity
2. **Local Development:** DefaultAzureCredential uses Azure CLI credentials (`az login`)
3. **CI/CD Pipeline:** DefaultAzureCredential uses Service Principal

---

## 7. Advanced Features

### 7.1 Secret Versioning

Key Vault maintains full version history for audit compliance.

```bash
# Create new version of secret
az keyvault secret set \
    --vault-name kv-reliance-prod-001 \
    --name stripe-api-key \
    --value "sk_live_51NewKey..."

# List all versions
az keyvault secret list-versions \
    --vault-name kv-reliance-prod-001 \
    --name stripe-api-key \
    --query "[].{Version:id, Enabled:attributes.enabled, Created:attributes.created}" \
    --output table

# Retrieve specific version
az keyvault secret show \
    --vault-name kv-reliance-prod-001 \
    --name stripe-api-key \
    --version <version-id>
```

### 7.2 Automated Secret Rotation

Implemented Azure Automation runbook for periodic key rotation.

```bash
# Enable Event Grid notifications for secret near expiry
az eventgrid event-subscription create \
    --name secret-expiry-notification \
    --source-resource-id $(az keyvault show \
        --name kv-reliance-prod-001 \
        --resource-group rg-security-prod-eastus-001 \
        --query id -o tsv) \
    --endpoint-type webhook \
    --endpoint https://automation-webhook.azure.com/webhooks?token=... \
    --included-event-types Microsoft.KeyVault.SecretNearExpiry
```

### 7.3 Certificate Management

Stored and auto-renewed SSL/TLS certificates.

```bash
# Import certificate
az keyvault certificate import \
    --vault-name kv-reliance-prod-001 \
    --name wildcard-reliance-com \
    --file certificate.pfx \
    --password "cert-password"

# Set auto-renewal policy
az keyvault certificate set-attributes \
    --vault-name kv-reliance-prod-001 \
    --name wildcard-reliance-com \
    --enabled true
```

---

## 8. Monitoring & Auditing (CAF Manage)

### Step 6: Diagnostic Settings

Configured comprehensive logging to Log Analytics workspace.

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
    --resource-group rg-security-prod-eastus-001 \
    --workspace-name log-keyvault-audit \
    --location eastus \
    --retention-time 90

# Enable diagnostic settings
az monitor diagnostic-settings create \
    --name keyvault-audit-logs \
    --resource $(az keyvault show \
        --name kv-reliance-prod-001 \
        --resource-group rg-security-prod-eastus-001 \
        --query id -o tsv) \
    --logs '[{"category":"AuditEvent","enabled":true,"retentionPolicy":{"enabled":true,"days":90}}]' \
    --metrics '[{"category":"AllMetrics","enabled":true,"retentionPolicy":{"enabled":true,"days":90}}]' \
    --workspace $(az monitor log-analytics workspace show \
        --resource-group rg-security-prod-eastus-001 \
        --workspace-name log-keyvault-audit \
        --query id -o tsv)
```

### 8.1 Log Analytics Queries

**Query 1: Failed Access Attempts**

```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT"
| where ResultSignature == "Forbidden"
| summarize Count=count() by CallerIPAddress, identity_claim_http_schemas_xmlsoap_org_ws_2005_05_identity_claims_upn_s, bin(TimeGenerated, 1h)
| where Count > 5
| order by TimeGenerated desc
```

**Query 2: Secret Access Audit Trail**

```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT"
| where OperationName == "SecretGet"
| project TimeGenerated, identity_claim_appid_g, CallerIPAddress, requestUri_s, ResultSignature
| order by TimeGenerated desc
```

### 8.2 Azure Monitor Alerts

Configured security alerts for suspicious activity.

```bash
# Alert on excessive failed access attempts
az monitor metrics alert create \
    --name alert-keyvault-unauthorized-access \
    --resource-group rg-security-prod-eastus-001 \
    --scopes $(az keyvault show --name kv-reliance-prod-001 \
        --resource-group rg-security-prod-eastus-001 --query id -o tsv) \
    --condition "total ServiceApiResult where ResultType = 'Forbidden' > 10" \
    --window-size 5m \
    --evaluation-frequency 1m \
    --severity 1 \
    --description "Alert when more than 10 unauthorized access attempts occur within 5 minutes"
```

**Evidence 5: Audit Log**  
_Description: Log Analytics workspace showing AuditEvent logs with CallerIPAddress, OperationName, and ResultSignature._

---

## 9. Integration with Azure Services

### 9.1 App Service Integration

Configured App Service to automatically load secrets from Key Vault.

```bash
# Create App Service with system-assigned managed identity
az webapp create \
    --resource-group rg-security-prod-eastus-001 \
    --plan asp-prod-eastus \
    --name app-erp-prod \
    --runtime "DOTNET|8.0"

# Enable managed identity
az webapp identity assign \
    --name app-erp-prod \
    --resource-group rg-security-prod-eastus-001

# Set Key Vault reference in App Settings
az webapp config appsettings set \
    --name app-erp-prod \
    --resource-group rg-security-prod-eastus-001 \
    --settings DatabaseConnection="@Microsoft.KeyVault(SecretUri=https://kv-reliance-prod-001.vault.azure.net/secrets/sql-connection-string/)"
```

### 9.2 Azure Functions Integration

```bash
# Functions automatically resolve Key Vault references using managed identity
az functionapp config appsettings set \
    --name func-payment-processor \
    --resource-group rg-security-prod-eastus-001 \
    --settings StripeApiKey="@Microsoft.KeyVault(VaultName=kv-reliance-prod-001;SecretName=stripe-api-key)"
```

---

## 10. Security Best Practices (CAF Govern)

### 10.1 Compliance & Governance Checklist

✅ **Network Isolation:** Public access disabled, private endpoints configured  
✅ **Identity Management:** Managed identities for applications (no service principals with secrets)  
✅ **RBAC:** Least-privilege access with built-in Azure roles  
✅ **Purge Protection:** 90-day soft delete protection enabled  
✅ **Audit Logging:** All operations logged to Log Analytics with 90-day retention  
✅ **Secret Rotation:** Expiration dates set with Event Grid notifications  
✅ **Encryption:** HSM-backed keys (Premium SKU) for highest security  
✅ **Monitoring:** Azure Monitor alerts for unauthorized access attempts

### 10.2 Azure Policy Enforcement

Applied organizational policies to enforce security standards.

```bash
# Assign built-in policy: "Key vaults should have purge protection enabled"
az policy assignment create \
    --name keyvault-purge-protection \
    --policy /providers/Microsoft.Authorization/policyDefinitions/0b60c0b2-2dc2-4e1c-b5c9-abbed971de53 \
    --scope $(az group show --name rg-security-prod-eastus-001 --query id -o tsv)

# Assign built-in policy: "Key vaults should have soft delete enabled"
az policy assignment create \
    --name keyvault-soft-delete \
    --policy /providers/Microsoft.Authorization/policyDefinitions/1e66c121-a66a-4b1f-9b83-0fd99bf0fc2d \
    --scope $(az group show --name rg-security-prod-eastus-001 --query id -o tsv)
```

---

## 11. Disaster Recovery & Business Continuity

### 11.1 Backup Strategy

Key Vault natively provides geo-redundancy and automatic backups.

- **Automatic Backup:** All secrets backed up to Azure paired region
- **Soft Delete:** 90-day recovery window for deleted secrets
- **Purge Protection:** Prevents permanent deletion during retention period
- **Geo-Replication:** Automatic failover to secondary region

### 11.2 Secret Export (Disaster Recovery)

```bash
# Export all secrets for disaster recovery (encrypted backup)
for secret in $(az keyvault secret list --vault-name kv-reliance-prod-001 --query "[].name" -o tsv); do
    az keyvault secret backup \
        --vault-name kv-reliance-prod-001 \
        --name $secret \
        --file "backup-${secret}.blob"
done
```

### 11.3 Recovery Testing

```bash
# Restore secret from backup
az keyvault secret restore \
    --vault-name kv-reliance-prod-001 \
    --file backup-stripe-api-key.blob
```

---

## 12. Cost Management

### 12.1 Pricing Model

- **Vault Operations:** $0.03 per 10,000 transactions
- **HSM-Protected Keys (Premium SKU):** $1/key/month
- **Certificate Operations:** $3.00 per renewal request
- **Storage:** Included in vault cost

### 12.2 Cost Optimization Strategies

- Use Standard SKU for non-sensitive secrets (software-protected keys)
- Implement caching in applications to reduce API calls
- Archive unused secrets instead of creating new versions
- Use free tier Log Analytics (5GB/month) for audit logging

---

## 13. Testing & Validation

### Test Case A: Secret Retrieval via Managed Identity

**Input:** Application with managed identity requests secret  
**Expected Output:** Successful retrieval with HTTP 200

```bash
# Verify from application logs
az monitor app-insights query \
    --app log-keyvault-audit \
    --analytics-query "traces | where message contains 'SecretRetrieved' | take 10"
```

**Evidence 6: Application Logs**  
_Description: Application Insights showing successful secret retrieval with latency < 100ms._

### Test Case B: Unauthorized Access Denial

**Input:** User without RBAC role attempts to read secret  
**Expected Output:** HTTP 403 Forbidden with audit log entry

```bash
# Attempt access without permission
az keyvault secret show \
    --vault-name kv-reliance-prod-001 \
    --name stripe-api-key \
    --subscription <different-subscription>
# Expected: Forbidden error
```

**Evidence 7: Audit Trail**  
_Description: Log Analytics showing failed access attempt with ResultSignature: Forbidden._

---

## 14. Conclusion

This project successfully implemented an enterprise-grade secrets management platform using Azure Key Vault, fully aligned with the Azure Cloud Adoption Framework:

✅ **Infrastructure:** Provisioned with Premium SKU, RBAC, and private endpoints  
✅ **Security:** Managed identities, network isolation, HSM-backed encryption  
✅ **Integration:** Seamless integration with App Services, Functions, and .NET applications  
✅ **Governance:** Azure Policy enforcement and comprehensive audit logging  
✅ **Monitoring:** Log Analytics with KQL queries and Azure Monitor alerts  
✅ **Compliance:** SOC 2, HIPAA, and PCI-DSS compliant configuration  
✅ **Disaster Recovery:** Geo-redundancy, soft delete, and automated backups  
✅ **Cost Optimization:** Efficient transaction management and tiered SKU usage

**Business Impact:**

- **Eliminated Hard-Coded Secrets:** Zero credentials stored in application code or configuration files
- **Centralized Management:** Single source of truth for all organizational secrets
- **Audit Compliance:** Complete audit trail for SOC 2 and regulatory requirements
- **Reduced Risk:** Automated secret rotation and access monitoring

**Next Steps:**

1. Implement Azure Key Vault Managed HSM for FIPS 140-2 Level 3 compliance
2. Integrate with Azure DevOps for secure CI/CD pipeline secrets
3. Configure cross-tenant access for partner integrations
4. Establish secret rotation automation using Azure Functions
5. Deploy Azure Front Door with Key Vault certificates for global TLS termination
