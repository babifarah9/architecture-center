This example scenario describes an architecture for a multitenant software as a service (SaaS) application hosted in Azure Container Apps and sold as a transactable SaaS offer through Microsoft Marketplace. It combines application authorization with the Microsoft Marketplace subscription lifecycle. The design uses Microsoft Entra ID for authentication, tenant membership checks and database row-level security for isolation, and durable processing of Marketplace lifecycle operations.

The scenario is based on engineering patterns implemented in a SaaS application. It is not a statement that every optional production control or end-to-end commercial certification test has been completed.

## Architecture

:::image type="complex" source="./media/transactable-multitenant-saas-marketplace.svg" alt-text="Architecture diagram of a Microsoft Marketplace SaaS customer onboarding into a multitenant Azure Container Apps application with Microsoft Entra ID, Key Vault, Azure Database for PostgreSQL, and lifecycle reconciliation jobs." border="false" lightbox="./media/transactable-multitenant-saas-marketplace.svg":::
A customer starts a purchase through Microsoft Marketplace and arrives at the SaaS landing page with a purchase token. The application server resolves the token with the Marketplace SaaS Fulfillment APIs. The customer authenticates through Microsoft Entra ID; the application validates membership and provisions an application tenant. After provisioning, the server activates the subscription. The browser talks to a secured API hosted in Azure Container Apps. The API uses managed identity to retrieve secrets from Azure Key Vault and accesses a private Azure Database for PostgreSQL server. A Marketplace webhook delivers subscription lifecycle events. The API persists and verifies the events; a Container Apps job reconciles outstanding operations. GitHub Actions deploys through Azure's federated identity to Azure Container Registry and the Container Apps environment. Azure Monitor and Application Insights collect telemetry.
:::image-end:::

### Data flow

1. A purchaser selects a transactable SaaS plan in Microsoft Marketplace. Marketplace redirects the user to the publisher's landing page with a short-lived purchase identification token.
1. The server resolves the token through the [SaaS Fulfillment subscription APIs](https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api). It validates the expected offer and plan rather than relying on browser-provided subscription data.
1. The beneficiary signs in with Microsoft Entra ID. The service validates the token and links the verified identity to an application tenant and membership.
1. The application provisions the tenant and subscription mapping in PostgreSQL, then requests Marketplace activation. A successful activation request isn't itself proof that the subscription is in the `Subscribed` state; the service reconciles confirmed Marketplace state before granting entitlements.
1. The browser calls the API with an Entra token. The API verifies membership and active Marketplace entitlements for each protected request, then sets the tenant context in the database transaction. PostgreSQL row-level security enforces an additional boundary.
1. Marketplace sends lifecycle notifications such as plan changes, quantity changes, suspension, reinstatement, and cancellation to a webhook. The service validates the webhook, queries the operation from Marketplace, records it durably, and applies verified changes idempotently.
1. A scheduled reconciliation worker checks for outstanding Marketplace operations so an unavailable webhook receiver doesn't permanently prevent subscription state changes.
1. GitHub Actions builds and releases immutable application images through Azure Container Registry and deploys Azure resources and revisions with Bicep. Microsoft Entra workload identity federation avoids a long-lived Azure deployment secret.

### Components

- [Azure Container Apps](https://learn.microsoft.com/azure/container-apps/overview) hosts the public web/API service, a one-shot database migration job, and the scheduled Marketplace recovery job.
- [Azure Database for PostgreSQL flexible server](https://learn.microsoft.com/azure/postgresql/flexible-server/overview) stores business tenants, membership, subscription mappings, and lifecycle events. The database uses private network access and PostgreSQL row-level security.
- [Microsoft Entra ID](https://learn.microsoft.com/entra/identity-platform/) authenticates users and issues API access tokens. Authentication alone doesn't authorize access to a tenant.
- [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/general/overview) stores database connection details and Marketplace credentials. Separate user-assigned managed identities scope access for the running application and database migrations.
- [Azure Container Registry](https://learn.microsoft.com/azure/container-registry/container-registry-intro) stores the immutable container release image.
- [Azure Virtual Network](https://learn.microsoft.com/azure/virtual-network/virtual-networks-overview) provides delegated subnets for the Container Apps environment and private PostgreSQL access, alongside private DNS.
- [Azure Monitor](https://learn.microsoft.com/azure/azure-monitor/overview), Application Insights, and Log Analytics provide application and platform observability.
- [Microsoft Marketplace SaaS Fulfillment APIs](https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api) provide the authoritative purchase, activation, subscription, and operation lifecycle.
- [GitHub Actions](https://docs.github.com/actions) deploys with protected environments and OpenID Connect federation to Azure.

## Scenario details

The solution separates three independent concepts: a user's Entra identity, the application's tenant membership, and the customer's Marketplace subscription entitlement. None of these should be substituted for another. The API derives tenant context from verified membership, not from an untrusted tenant identifier supplied by a browser. The data tier applies forced row-level security to tenant-owned tables through an application role that neither owns those tables nor bypasses RLS.

Marketplace lifecycle handling should be resilient to duplicate notifications and missed callbacks. Record the Marketplace activity or operation identifier durably, validate the operation with Marketplace before changing access, and implement an idempotent state transition. Acknowledge webhook delivery promptly and process business changes through durable storage. An independent reconciliation process covers operations that weren't delivered or processed successfully.

### Purchase and activation

The landing page resolves the purchase token on the server. The publisher provisions an application tenant and records the subscription mapping before calling the activation API. The application must not infer a final active entitlement solely from a successful activation request. It confirms the state using the Marketplace subscription APIs. Duplicate landing-page requests must return the existing tenant/subscription association instead of provisioning a second workspace.

### Plan and subscription changes

The application maps only supported Marketplace offer and plan identifiers to server-side feature entitlements. It verifies incoming operations via the Fulfillment operations API before applying plan, quantity, suspension, reinstatement, or cancellation changes. A suspended or unsubscribed customer loses protected application access, while data retention follows the publisher's stated policy. Keep entitlement checks on the server, not only in disabled UI controls.

### Security boundaries

- **Identity:** Validate Entra issuer, audience, signature, token version, and tenant policies. Check application membership independently of sign-in.
- **Tenant isolation:** Set tenant context per database transaction; enforce RLS and test cross-tenant reads and writes. Use separate runtime and migration database roles.
- **Secrets:** Grant the web identity access only to runtime secrets and the migration identity access only to migration credentials. Keep credentials out of source control and browser responses.
- **Network:** Place the PostgreSQL server in a delegated private subnet. Expose only the intended HTTPS application endpoint.
- **Marketplace notifications:** Validate incoming credentials or tokens and confirm the operation directly with Marketplace before changing entitlements.
- **Delivery:** Use a protected production GitHub environment, immutable image tags, deployment health checks, and separate testing and production configurations.

## Design considerations

The following considerations align with the [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/).

### Reliability

Make purchase resolution and provisioning idempotent. Persist lifecycle event identifiers and retries, and reconcile outstanding operations on a schedule. Configure application readiness probes and a failed-release gate. Plan and test database restoration and rollback before production rollout.

### Security

Apply both membership authorization and database-enforced tenant isolation. Managed identities and Key Vault reduce application secret exposure, but Marketplace publisher credentials and database passwords still require rotation policies. Consider additional Key Vault network restrictions and edge protection depending on deployment requirements.

### Operational excellence

Treat foundation and application deployment as separate infrastructure-as-code stages. Build an immutable container image and verify the deployed application's readiness before treating a release as successful. Use structured logging with request correlation IDs; avoid logging Marketplace purchase tokens, customer data, or credentials.

### Performance efficiency

Container Apps can scale application replicas independently of database resources. Size PostgreSQL for concurrency, connection limits, and workload behavior. Apply connection pooling and performance testing before increasing replica counts. Schedule reconciliation work according to Marketplace requirements and expected event volume.

### Cost optimization

A shared multitenant app and database can reduce per-customer infrastructure costs, but require strong isolation controls. Use an appropriately sized Container Apps environment and PostgreSQL tier for development, and account for database high availability, log ingestion, backup retention, and idle resources in production estimates.

## Alternatives

- Use [Azure App Service](https://learn.microsoft.com/azure/app-service/overview) for a simpler web-hosting model if separate Container Apps jobs or container deployment aren't needed.
- Use [Azure Kubernetes Service](https://learn.microsoft.com/azure/aks/intro-kubernetes) when container orchestration requirements justify additional management complexity.
- Use a dedicated database or deployment per customer when required by strict isolation, compliance, or customer-specific lifecycle constraints, at a higher operating cost.
- Use a managed queue for lifecycle-event buffering if webhook throughput or processing time exceeds what a database-backed inbox can comfortably support.

## Deploy this scenario

A production implementation needs an Entra multitenant application, a Partner Center transactable SaaS offer, a landing page, a webhook endpoint, Azure hosting, a secure data store, and monitored subscription state transitions. Configure separate development/preview and production Marketplace offers and publisher credentials. Complete end-to-end testing of initial purchase, activation, duplicate events, plan and quantity changes, suspension, reinstatement, cancellation, and missed-operation recovery before production release.

The architecture diagram is illustrative. Adapt the identity, network, operational, and entitlement configurations to your organization's requirements, and validate the current Partner Center integration contract.

## Contributors

This architecture scenario was contributed by Bassem Abi-Farah, Converture Inc.

## Related resources

- [Create a SaaS offer in Microsoft Marketplace](https://learn.microsoft.com/partner-center/marketplace-offers/create-new-saas-offer)
- [SaaS Fulfillment subscription APIs](https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api)
- [SaaS Fulfillment operations APIs](https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-operations-api)
- [SaaS webhook reference](https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-webhook)
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
