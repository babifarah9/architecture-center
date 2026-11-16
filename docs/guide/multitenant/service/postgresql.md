---
title: Guidance for Using Azure Database for PostgreSQL in a Multitenant Solution
description: Learn about the Azure Database for PostgreSQL features that are useful when you work with multitenant systems. Explore guidance and examples.
author: PlagueHO
ms.author: dascottr
ms.date: 11/11/2026
ms.update-cycle: 180-days
ms.topic: concept-article
ms.subservice: architecture-guide
ms.custom: arb-saas
ai-usage: ai-assisted
---

# Guidance for using Azure Database for PostgreSQL in a multitenant solution

Many multitenant solutions on Azure use the open-source relational database management system Azure Database for PostgreSQL. This article describes the features of Azure Database for PostgreSQL that are useful when you work with multitenant systems. The article also includes links to guidance and examples for how to use Azure Database for PostgreSQL in a multitenant solution.

## Deployment options

The following options are available for Azure Database for PostgreSQL and are suitable for use with multitenant applications:

- [Azure Database for PostgreSQL](/azure/postgresql/) is a good choice for most multitenant deployments that don't require the horizontal scale-out that elastic clusters provide.

- Azure Database for PostgreSQL [elastic clusters](/azure/postgresql/elastic-clusters/concepts-elastic-clusters) provides horizontal scaling within a managed service. It's suitable for multitenant applications that need to scale from a few tenants to high numbers of tenants.

## Isolation models

Determine the level of isolation that you want when you use Azure Database for PostgreSQL in your multitenant system. The isolation boundary can be the server, the database or schema, or the rows within shared tables.

A system can implement more than one model. Some tenants might share tables, while other tenants have a dedicated server. For more information about selecting a model, see [Tenancy models to consider for a multitenant solution](../considerations/tenancy-models.md) and [Architectural approaches for storage and data in multitenant solutions](../approaches/storage-data.md).

The following table summarizes the differences between the main tenancy isolation models for Azure Database for PostgreSQL.

| Consideration | Server for each tenant | Database or schema for each tenant | Shared tables |
| --- | --- | --- | --- |
| **Data isolation** | High | Medium. Database permissions separate tenants. | Low. Your application and row security policies separate tenants. |
| **Performance isolation** | High. Size each server for the tenant's requirements. | Low. Can have noisy neighbor problems. | Low, with noisy-neighbor risk. Elastic clusters can improve isolation by dedicating nodes to tenants. |
| **Deployment complexity** | Medium. Deploy and configure a server for each tenant. | Medium. Deploy a database or schema for each tenant. | Low |
| **Operational complexity** | High. Monitor each server and coordinate maintenance windows and major-version upgrades. | Medium. Apply schema changes to each tenant's objects. | Low |
| **Resource cost** | High | Low-medium | Low |
| **Example scenario** | Tenants that require their own encryption key or compliance boundary. | Tenants that require a permission boundary but share a server. | Large multitenant solution that has a shared application tier and many small tenants. |

The following sections describe the models in more detail.

### Server for each tenant

This model deploys a server for each tenant. This approach provides the maximum level of isolation, and it's the only model that can apply server-level controls to a single tenant.

Use the following techniques to match a server to a tenant's requirements:

- Deploy the server to a region that's close to the tenant.

- Select a compute tier and storage size that suit the tenant's workload.

- Use tenant-specific encryption keys. Customer-managed keys apply at the server level, so a dedicated server is the only way to give a tenant its own key. For more information, see [Encryption](#encryption).

- If you need to restore a tenant to a point in time, you can do so without affecting other tenants. Backup and restore operate at the server level.

This isolation model becomes more complex to manage as the number of tenants increases. It also increases costs because you pay for each server, and small tenants rarely use a full server's resources.

### Database or schema for each tenant

If multiple tenants share a server, you can isolate each tenant in a dedicated database or schema. To control access, grant each tenant's role permissions on only that tenant's database or schema. Your application doesn't filter rows, which removes the risk that an application bug, like a missing query filter, exposes another tenant's data.

As your system adds more tenants, the number of databases or schemas increases. You then apply schema changes and maintenance to each tenant's objects, which raises operational cost. [Elastic clusters](#elastic-clusters) distribute tenant schemas across nodes. With transaction pooling, your application sets the tenant's `search_path` within every transaction by using `SET LOCAL search_path`. For session pooling, reset the `search_path` before reusing a connection for another tenant. For more information, see [Connection pooling](#connection-pooling).

Shared servers increase the risk of [noisy neighbor problems](../../../antipatterns/noisy-neighbor/noisy-neighbor.yml). Tenants share the server's compute, memory, and storage, so a tenant that runs expensive queries or holds long transactions can degrade performance for other tenants. PgBouncer creates a separate pool for each user and database pair, so separate databases or roles limit how many server connections a single tenant consumes.

Tenants share server resources, so Azure billing data doesn't show the cost of each tenant. Measure the resources that each tenant consumes in your application. For more information, see [Measure the consumption of each tenant](../considerations/measure-consumption.md).

Backups apply to the whole server. To recover one tenant, you need to restore the server's backup to a new server, and then copy that tenant's data back.

### Shared tables

All your tenants can share tables, and a tenant identifier column separates their rows. This model decreases operational complexity and resource costs, and it supports the largest number of tenants. It's common in models like business-to-consumer (B2C) software as a service (SaaS), where tenant density is important and isolation requirements are comparatively less strict than other models.

But shared tables don't provide data isolation between tenants, so you're responsible for satisfying data isolation requirements in your application logic and in row security policies. Set the tenant context within each transaction, and review which roles can bypass row security policies. For more information, see [Row-level security](#row-level-security).

A shared server can also lead to [noisy neighbor problems](../../../antipatterns/noisy-neighbor/noisy-neighbor.yml) for some tenants. [Elastic clusters](#elastic-clusters) distribute rows by tenant ID across nodes and support more tenants for each node than schema-based sharding. You can move a tenant that consumes too many resources to its own shard group, and rebalancing runs online.

Tenants share server resources, so Azure billing data doesn't show the cost of each tenant. Measure the resources that each tenant consumes in your application. For more information, see [Measure the consumption of each tenant](../considerations/measure-consumption.md).

Backups apply to the whole server. To recover one tenant, you need to restore the server's backup to a new server, and then copy that tenant's data back.

## Azure Database for PostgreSQL features that support multitenancy

When you use Azure Database for PostgreSQL to build a multitenant application, the following features can enhance your solution.

> [!NOTE]
> Some features are only available in specific [deployment options](#deployment-options). The following guidance describes which features are available.

### Row-level security

Row-level security is useful for enforcing tenant-level isolation when you use shared tables. In PostgreSQL, you implement row-level security by applying *row security policies* to tables to restrict access to rows by tenant.

Implementing row-level security on a table might affect performance. You might need to create other indexes on tables that have row-level security enabled to ensure that performance isn't affected. When you use row-level security, it's important to use performance testing techniques to validate that your workload meets your baseline performance requirements.

Row security policies don't apply to every role. A role that has the `BYPASSRLS` attribute reads all rows in a table, and PostgreSQL exempts a table's owner from the policies on that table unless you force row-level security on the table. On servers that run PostgreSQL 16 and later, an administrator can create nonadministrative roles that have the `BYPASSRLS` attribute. Audit which roles hold that attribute, and don't let your application roles own the tables that they query. Otherwise, a routine permission change can remove your tenant isolation boundary without any visible error.

For more information, see [Access management for Azure Database for PostgreSQL](/azure/postgresql/security/security-access-control#row-level-security) and [Row security policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) in the PostgreSQL documentation.

#### Tenant context for row-level security

Row-level security policies require access to the current tenant identifier. Azure Database for PostgreSQL provides the **session_variable** extension, which allows for session-scoped variables that you can use to store and retrieve the tenant identifier within a session. The extension uses functions such as `session_variable.set(...)` to change a variable, not `SET LOCAL`. For API details, see the [session_variable documentation](https://github.com/splendiddata/session_variable).

> [!WARNING]
> Session-scoped tenant context belongs to a server connection rather than to a logical client. If connection pooling sits between your application and the database, a pooled connection can carry the tenant context of a previous tenant into the next unit of work. Don't rely on the session_variable extension to provide transaction-local tenant context.

With transaction pooling, use a transaction-local PostgreSQL configuration setting for the tenant identifier, and have your row security policies read that setting. Set it within every transaction before accessing tenant data. The PostgreSQL [`SET LOCAL` command](https://www.postgresql.org/docs/current/sql-set.html) limits a configuration setting to the current transaction and restores the previous value at commit or rollback. It doesn't change variables managed by the `session_variable` extension. For more information about session-scoped context and pooling modes, see [Connection pooling](#connection-pooling).

For more information, see [Extensions and extension versions for Azure Database for PostgreSQL](/azure/postgresql/extensions/concepts-extensions-versions).

### Elastic clusters

Elastic clusters are a feature of Azure Database for PostgreSQL. They provide horizontal scaling capabilities within a single managed service. This deployment option uses distributed table functionality for multitenant workloads that require scale-out capabilities.

Elastic clusters currently support only one database per cluster, so you can't create a database for each tenant within a cluster. You also can't reduce the node count or perform major-version upgrades. Review these [limitations](/azure/postgresql/elastic-clusters/concepts-elastic-clusters-limitations) before choosing this option.

The [Sharding pattern](../../../patterns/sharding.md) enables you to scale your workload across multiple databases or database servers. Elastic clusters support two sharding models for multitenant workloads:

- **Row-based sharding**: Distribute rows by tenant ID to colocate each tenant's data on a node. Include the distribution column in queries so the cluster can route them to the appropriate node. This approach can improve query performance for tenant-specific queries, and it supports the highest number of tenants for each node.
- **Schema-based sharding**: Use a separate schema for each tenant. Queries don't need to change, but the application must set the appropriate `search_path`. With transaction pooling, use `SET LOCAL search_path` within every transaction. With session pooling, reset the `search_path` before reusing a connection for another tenant. For more information, see [Connection pooling](#connection-pooling).

    This approach suits applications that can't be modified to support row-based sharding. Microsoft's sharding guidance suggests it for workloads that have up to approximately 10,000 tenants, and it supports fewer tenants for each node than row-based sharding.

For more information, see the following articles:

- [Elastic clusters in Azure Database for PostgreSQL](/azure/postgresql/elastic-clusters/concepts-elastic-clusters)
- [Sharding models for elastic clusters](/azure/postgresql/elastic-clusters/concepts-elastic-clusters-sharding-models)

### Connection pooling

Postgres uses a process-based model for connections. This model makes it inefficient to maintain large numbers of idle connections. Some multitenant architectures require many active connections, which negatively affect the performance of the Postgres server.

Azure Database for PostgreSQL includes [PgBouncer](/azure/postgresql/connectivity/concepts-pgbouncer) as a built-in connection pooling solution that you enable for each server. Built-in PgBouncer isn't supported on the Burstable compute tier. If your workload on that tier needs pooling, use application-level pooling or an external pooler. Workloads with few connections might not need pooling. In an elastic cluster, each node runs its own PgBouncer instance.

The pooling mode determines whether session state follows a logical client. The built-in PgBouncer uses transaction pooling by default, which returns the server connection to the pool when each transaction completes. A connection can serve a different tenant on its next transaction, so any tenant context that your application sets at authentication or for the duration of a session isn't guaranteed to apply. Choose one of the following approaches for multitenant workloads:

- Keep transaction pooling and set the tenant context within each transaction. This approach preserves the connection density that makes pooling valuable when many tenants share a server.
- Configure [session pooling](/azure/postgresql/connectivity/concepts-connection-pooling-best-practices) so that a server connection stays assigned to a client for the life of its connection. Session pooling supports session-scoped tenant context, but it holds server connections longer and reduces the number of clients that a server can support. Reset the tenant context before your application returns a connection to its own pool.

For more information, see the following articles:

- [PgBouncer in Azure Database for PostgreSQL](/azure/postgresql/connectivity/concepts-pgbouncer)
- [Connection pooling strategy for Azure Database for PostgreSQL by using PgBouncer](/azure/postgresql/connectivity/concepts-connection-pooling-best-practices)

### Microsoft Entra authentication

Azure Database for PostgreSQL supports authentication by using Microsoft Entra ID. Your application connects with a managed identity or a service principal instead of a password, which keeps database credentials out of your configuration.

In a multitenant solution, you can give each tenant-specific component its own workload identity and map that identity to a PostgreSQL role that holds only the permissions that the tenant's data requires.

Authentication establishes which identity connects to the database. It doesn't isolate tenant data on its own. Combine Microsoft Entra authentication with database permissions and row security policies so that a connection reaches only the data that belongs to its tenant. Azure Database for PostgreSQL matches access tokens to database roles by using the identity's object ID rather than its name, so deleting an identity and creating another one that has the same name doesn't restore database access. Account for this behavior when you offboard a tenant or rotate the identities that a tenant uses.

For more information, see the following articles:

- [Microsoft Entra authentication in Azure Database for PostgreSQL](/azure/postgresql/security/security-entra-concepts)
- [Connect with managed identity to Azure Database for PostgreSQL](/azure/postgresql/security/security-connect-with-managed-identity)

### Azure confidential computing

Azure Database for PostgreSQL supports Azure confidential computing through trusted execution environments (TEEs), which provide hardware-based protection for data in use. This feature protects tenant data from unauthorized access by the operating system, hypervisor, or other applications.

For multitenant solutions that handle sensitive data, use confidential computing when your tenants have regulatory or contractual requirements to protect data in use. Confidential computing doesn't restrict access by authorized database roles or by your application code, so it isn't a substitute for tenant isolation controls such as row security policies and scoped database permissions.

For more information, see [Azure confidential computing for Azure Database for PostgreSQL](/azure/postgresql/security/security-confidential-computing).

### Encryption

Azure Database for PostgreSQL encrypts data stored in the service at rest by default by using Microsoft-managed keys. You can use customer-managed keys (CMKs) to control the server's encryption key. To give each tenant a distinct server encryption key, deploy a separate server for each tenant.

When you use [CMKs](/azure/postgresql/security/security-data-encryption), you can store a server's encryption key in [Azure Key Vault](/azure/key-vault/general/overview) or [Azure Key Vault Managed HSM](/azure/key-vault/managed-hsm/overview). CMKs apply at the server level, so tenants that share a server can't use different server encryption keys.

For separation of duties, Azure Database for PostgreSQL flexible server supports [cross-tenant CMKs](/azure/postgresql/security/security-data-encryption#cross-tenant-customer-managed-keys-cmk), where the key store resides in a different Microsoft Entra tenant than the server. In a SaaS solution, this option lets a customer own and administer the encryption key while you operate the database server in your own Microsoft Entra tenant. Cross-tenant CMKs separate key ownership and administration from server administration. They don't provide different encryption keys for tenants that share a server. Review the [CMK limitations](/azure/postgresql/security/security-data-encryption#limitations-of-customer-managed-keys-cmk) before choosing this option. For example, long-term retention backups aren't supported on servers that use cross-tenant CMKs. Confirm that your backup commitments don't depend on long-term retention.

To use [automatic key version updates](/azure/postgresql/security/security-data-encryption#cmk-key-version-updates), configure the server with a versionless key URI, which omits the version identifier. The service then adopts new key versions after each rotation in Key Vault, so you don't maintain the key version reference manually. If you configure the server with a versioned key URI, the server continues to use that version until you update the reference yourself. Keep the previous key version available for at least two hours after each rotation so that re-encryption can complete. If you disable it sooner, the server can become inaccessible.

## Contributors

*Microsoft maintains this article. The following contributors wrote this article.*

Principal author:

- [Daniel Scott-Raynsford](https://www.linkedin.com/in/dscottraynsford) | Partner Solution Architect, Data & AI

Other contributors:

- [John Downs](https://www.linkedin.com/in/john-downs/) | Principal Software Engineer, Azure Patterns & Practices
- [Arsen Vladimirskiy](https://www.linkedin.com/in/arsenv) | Principal Customer Engineer, FastTrack for Azure
- [Paul Burpo](https://www.linkedin.com/in/paul-burpo/) | Principal Customer Engineer, FastTrack for Azure ISVs
- [Assaf Fraenkel](https://www.linkedin.com/in/assaf-fraenkel/) | Senior Engineer/Data Architect, Azure FastTrack for ISVs and Start-ups

*To see nonpublic LinkedIn profiles, sign in to LinkedIn.*

## Related resources

- [Tenancy models to consider for a multitenant solution](../considerations/tenancy-models.md)
- [Architectural approaches for storage and data in multitenant solutions](../approaches/storage-data.md)
- [Architecture best practices for Azure Database for PostgreSQL](/azure/well-architected/service-guides/postgresql)
