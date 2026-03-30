# Atuin Server on Coolify 4

This repository provides a configuration for running an [Atuin](https://atuin.sh/) server on [Coolify 4](https://coolify.io/).

## Prerequisites

- A running instance of Coolify 4.
- A PostgreSQL database (either managed by Coolify or external).

## Deployment Steps

### 1. Create a New Project
1. Log into your Coolify dashboard.
2. Create a new **Project**.
3. Create a new **Environment** (e.g., `production`).

### 2. Add the Atuin Server
1. Click **+ New Resource** and select **Public Repository** or **Private Repository** (depending on where you host this).
2. Point to this repository.
3. Select **Docker Compose** as the build pack.
4. Coolify will automatically detect the `docker-compose.yaml`.

### 3. Configure PostgreSQL
Atuin requires a PostgreSQL database. You have two options in Coolify:

#### Option A: Use a Coolify-managed PostgreSQL (Recommended)
1. Add a new **PostgreSQL** database service in the same project.
2. Once created, go to the **Service Settings** of the database to find the connection details.
3. Use these details in the environment variables for the Atuin service (see below).

#### Option B: Use an External PostgreSQL (Different Project or External Host)
1. Provide the connection details in the environment variables (see Step 4).
2. **Crucial Networking Step:** If your PostgreSQL is in a *different* Coolify project, you must ensure the Atuin service can reach it:
   - Go to the **Advanced** tab of your Atuin service in Coolify.
   - Look for the **Network** or **Docker Network** section.
   - Select **Connect to predefined network** (usually `coolify`).
   - This allows the containers in this Docker Compose stack to communicate with other services (like PostgreSQL) across different projects on the same Coolify instance using their internal container names or IP addresses.

### 4. Set Environment Variables
In the Atuin service settings in Coolify, go to the **Environment Variables** tab and add the following:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `POSTGRES_USER` | Database username | `atuin` |
| `POSTGRES_PASSWORD` | Database password | `your_secure_password` |
| `POSTGRES_HOST` | Database host (e.g., service name in Coolify) | `postgresql` |
| `POSTGRES_DB` | Database name | `atuin` |

Coolify will use these to construct the `ATUIN_DB_URI` as defined in `docker-compose.yaml`.

### 5. Persistent Storage
The `docker-compose.yaml` mounts `./config:/config`. In Coolify, you should ensure that the `config/` directory is persistent.

1. Go to the **Storage** tab of the Atuin service in Coolify.
2. Add a new **Persistent Volume**.
3. Set the **Mount Path** to `/config`.
4. This ensures that your configuration and database (if using SQLite) are preserved across redeploys.

> **Note:** If you choose to use SQLite instead of PostgreSQL, you must change the `ATUIN_DB_URI` environment variable to `sqlite:///config/atuin.db` and ensure the `/config` volume is correctly mounted.

### 6. Domain Setup
1. In the service settings, find the **Domains** field.
2. Enter your desired domain name (e.g., `https://atuin.example.com`).
3. Coolify will automatically manage SSL certificates and set up a reverse proxy to forward traffic to port `8888`.

### 7. Security (Post-Registration)
Once you have registered your user account, it is recommended to disable open registration:
1. Open the `config/server.toml` file (either locally or via Coolify).
2. Set `open_registration = false`.
3. Redeploy the service.

### 8. Deploy
1. Click **Deploy**.
2. Once the service is running, it will be available on port `8888` (or the domain you've assigned in Coolify).

## Local Development

This project uses [mise](https://mise.jdx.dev/) and Docker Compose for local development.

### Start the Server
```bash
mise run start
```
This will start the Atuin server on `localhost:8888` and a PostgreSQL 18 database.

### Verify the Server
Atuin is an API server and does not have a web UI. Verify it is running with:
```bash
curl http://localhost:8888/
```
You should see a JSON response containing the Atuin version and a quote.

### Stop the Server
```bash
mise run stop
```

## Continuous Integration
This repository includes a GitHub Actions workflow that automatically tests the Docker Compose setup on every pull request and push to the `main` branch.

## Self-Contained Setup (All-in-One)

If you prefer to have the database managed within the same Docker Compose stack in Coolify, you can use the following configuration:

```yaml
services:
  atuin:
    image: ghcr.io/atuinsh/atuin:v18.13.3
    command: start
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      ATUIN_HOST: "0.0.0.0"
      ATUIN_DB_URI: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      ATUIN_CONFIG_DIR: /config
    volumes:
      - ./config:/config
    ports:
      - "8888:8888"

  postgres:
    image: postgres:18-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres-data:/var/lib/postgresql

volumes:
  postgres-data:
```

> **Note for Postgres 18:** The volume mount point has changed to `/var/lib/postgresql` (removing the `/data` suffix) to support the new version-specific directory structure.

---

## Client Configuration

To use your new Atuin server on your local machine:

1. Register an account (if `open_registration = true` in `server.toml`):
   ```bash
   atuin register -u <username> -e <email> -p <password>
   ```
2. Login:
   ```bash
   atuin login -u <username> -p <password>
   ```
3. Update your `~/.config/atuin/config.toml`:
   ```toml
   sync_address = "https://your-atuin-domain.com"
   ```
4. Sync your history:
   ```bash
   atuin sync
   ```

## Troubleshooting
- **Database Connection:**
  - Ensure the PostgreSQL database is reachable from the Atuin container.
  - If they are in the same Coolify project, you can usually use the service name as the host.
  - If they are in **different projects**, remember to select **Connect to predefined network** in the **Advanced** tab of the Atuin service settings.
- **Empty Reply from Server:** Ensure `ATUIN_HOST` is set to `0.0.0.0`.
- **Logs:** Check the **Logs** tab in Coolify or run `mise run logs` locally.
