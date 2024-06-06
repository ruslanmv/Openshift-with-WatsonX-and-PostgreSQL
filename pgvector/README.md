**PostgreSQL + pgvector Setup**
================================

**About pgvector**

pgvector is an open-source vector similarity search for Postgres, allowing you to store and query vectors alongside your relational data. It supports exact and approximate nearest neighbor search, L2 distance, inner product, and cosine distance, and is compatible with any language that has a Postgres client.

**Container Image**

The provided `Containerfile` builds a PostgreSQL 15 + pgvector image, with pgvector built from source. You can deploy this container like any other PostgreSQL image.

Alternatively, you can use the prebuilt image available at [https://quay.io/repository/rh-aiservices-bu/postgresql-15-pgvector-c9s](https://quay.io/repository/rh-aiservices-bu/postgresql-15-pgvector-c9s).

**Deployment**

To deploy a PostgreSQL+pgvector instance, follow these steps:

1. Create a secret using `01_db_secret.yaml`, which defines the database name, username, and password.
2. Create a Persistent Volume Claim (PVC) using `02_pvc.yaml`, which persists the database data. Make sure to add a default storage class if necessary.
3. Deploy the PostgreSQL server using `03_deployment.yaml`.
4. Expose the PostgreSQL server using `04_services.yaml`.

After applying these files, you should have a running PostgreSQL+pgvector server accessible at `postgresql.your-project.svc.cluster.local:5432`.

**Enabling PgVector Extension**

To enable the PgVector extension, you need to connect to the running server Pod as a Superuser and execute the following command:
```
psql -d vectordb -c "CREATE EXTENSION vector;"
```
Replace `vectordb` with the name of your database if you changed it in the Secret.

If the command succeeds, it will print `CREATE EXTENSION`.

**Troubleshooting**

If you encounter any issues, make sure to check the pod logs and verify that the PgVector extension is enabled.