Snippet Overview
====================

- [riggering an Azure Data Factory Pipeline](#Triggering-an-Azure-Data-Factory-Pipeline)

## Triggering an Azure Data Factory Pipeline
**Maintainer:** @tomkerkhove

Provides the capability to trigger a specific Azure Data Factory Pipeline with or without parameters, in this example `UserEmailAddress`. The authentication handshake with Azure Management REST API is handled in the policy itself so that consumers do not need to manage this.

Required Named Values:
- `ActiveDirectory-TenantId` - Tenant id of the subscription
- `ServicePrinciple-ClientId` - Client id of the service principle which will be used by Azure API Management to authenticate with Azure Management REST API.  which has te required permissions ([docs](https://docs.microsoft.com/en-us/azure/data-factory/quickstart-create-data-factory-rest-api))
- `ServicePrinciple-ClientSecret` - Client secret of the service principle
- `Subscription-Id` - Id of the subscription
- `ResourceGroup-Name` - Name of the resource group
- `DataFactory-Name` - Name of the Azure Data Factory instance
- `Pipeline-Name` - Name of the Azure Data Factory pipeline to trigger
