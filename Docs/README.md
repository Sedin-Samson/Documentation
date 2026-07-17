# Bizweave & Zapstro Deployment & Setup Guide

This guide provides step-by-step instructions to set up, configure, and run the complete ecosystem containing **Bizweave Backend** (Python ETL engine) and **Zapstro** (Next.js ETL management platform).

---

## 📋 System Prerequisites

Ensure you have the following installed on your machine:

1. **Docker & Docker Compose**: Essential for containerizing databases, backend, and the Zapstro frontend.
   - [Install Docker Desktop](https://docs.docker.com/get-docker/)
2. **Python 3.10**: Required to run the Bizweave backend locally and execute database seed/admin scripts.
   - [Install Python 3.10](https://www.python.org/downloads/release/python-3100/)

---

## 🚀 Setup Steps

### 1. Clone the Repositories
Clone both repositories into the same parent directory (`bizweave`) to maintain the expected folder structure:
```bash
# Create directory and enter it
mkdir bizweave && cd bizweave

# Clone Bizweave Backend
git clone <bizweave-backend-repo-url> bizweave_backend

# Clone Zapstro
git clone <zapstro-repo-url> zapstro
```

Your directory structure should look like this:
```text
bizweave/
├── bizweave_backend/      # Python backend application
├── zapstro/               # Next.js frontend application
└── docker-compose.yaml    # Root docker-compose file for databases and applications
```

---

### 2. Create the Docker External Network
The Docker configuration uses an external network named `zapstro-network` to link containers across both services. Create it by running:
```bash
docker network create zapstro-network
```

---

### 3. Environment & Configuration Files

#### A. Configure Zapstro Environment File
1. Navigate to the `zapstro` folder:
   ```bash
   cd zapstro
   ```
2. Copy the example `.env` template:
   ```bash
   cp .env.example .env
   ```
3. Open [.env](file:///Users/samson/Documents/Datakulture/bizweave/zapstro/.env) and populate the values:
   - For **Docker Setup**, point the database host to the Docker service name (`postgresql`):
     ```env
     NODE_ENV=development
     DB_HOST=postgresql
     DB_PORT=5432
     DB_USER=postgres
     DB_PASSWORD=datakulturedk123
     DB_NAME=etl_db
     DATABASE_URL=postgresql://postgres:datakulturedk123@postgresql:5432/etl_db
     NEXTAUTH_URL=http://localhost:3000
     NEXTAUTH_SECRET=your_nextauth_secret_here # Generate with: openssl rand -base64 32
     NEXT_PUBLIC_MY_SECRET=your_jwt_secret_here
     BACKEND_SERVICE_KEY=2b10QLbPEZdbLskK1JRpHWfz4uaWjHCWDhu0iO8ECCq8RnVs7nclFE5Fa
     EXTERNAL_SERVICE_URL=http://localhost:3000/accelerator
     ```
   - For **Local Setup (running apps natively)**, change `DB_HOST` to `localhost`:
     ```env
     DB_HOST=localhost
     DATABASE_URL=postgresql://postgres:datakulturedk123@localhost:5432/etl_db
     ```

#### B. Configure Bizweave Backend Environment File
1. Navigate to the `bizweave_backend` folder:
   ```bash
   cd ../bizweave_backend
   ```
2. Create or verify the [.env](file:///Users/samson/Documents/Datakulture/bizweave/bizweave_backend/.env) file:
   ```env
   shared-secret-key = "2b10QLbPEZdbLskK1JRpHWfz4uaWjHCWDhu0iO8ECCq8RnVs7nclFE5Fa"
   ```
3. Verify the credentials inside [config.yaml](file:///Users/samson/Documents/Datakulture/bizweave/bizweave_backend/config.yaml) for cloud integrations and API keys.

---

## 🛠️ How to Run the Applications

### Option A: Running the Entire Stack via Docker (Recommended)
You can start all databases (Postgres, MySQL, ClickHouse, MSSQL, Redis, Portainer) along with the backend and frontend services using the root [docker-compose.yaml](file:///Users/samson/Documents/Datakulture/bizweave/docker-compose.yaml):

1. Go to the root workspace directory (`bizweave/`):
   ```bash
   cd ..
   ```
2. Launch all services:
   ```bash
   docker compose up -d --build
   ```
3. Access services:
   - **Zapstro Web Dashboard**: [http://localhost:3001](http://localhost:3001)
   - **Bizweave Backend API**: [http://localhost:4060](http://localhost:4060)
   - **Portainer Docker GUI**: [https://localhost:9443](https://localhost:9443)

---

### Option B: Running Applications Locally (Development Mode)
If you are actively developing code and want hot-reloading:

#### 1. Start the Databases Container Only
In the root directory, start the databases first:
```bash
docker compose up -d postgresql mysql clickhouse mssql redis
```

#### 2. Run Bizweave Backend Locally
1. Navigate to `bizweave_backend`:
   ```bash
   cd bizweave_backend
   ```
2. Create a virtual environment and activate it:
   ```bash
   python3.10 -m venv venv
   source venv/bin/activate
   ```
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
4. Start the backend server:
   ```bash
   python3 -m uvicorn app.__main__:app --host 0.0.0.0 --port 4060 --reload
   ```

#### 3. Run Zapstro Frontend Container
The Zapstro frontend runs inside a Docker container using the configuration in the `zapstro` directory.

1. Start the databases in the root directory:
   ```bash
   docker compose up -d postgresql mysql clickhouse mssql redis
   ```

2. Navigate into the `zapstro` directory and start the frontend container:
   ```bash
   cd zapstro
   docker compose up -d --build
   ```
   *(Note: The container uses `zapstro/docker-compose.yaml` and will automatically run database migrations on startup).*

3. Create the admin user:
   - In the `zapstro` directory, install the required packages for the script on your host machine:
     ```bash
     pip install -r scripts/requirements.txt
     ```
   - Run the script on your host machine to seed the admin user (ensure the `DB_HOST` variable in `zapstro/.env` is set to `localhost` so the script can connect to Postgres from the host):
     ```bash
     python3 scripts/create-dk-admin.py
     ```

4. Visit the dashboard at [http://localhost:3001](http://localhost:3001).

---

## 🛟 Troubleshooting

- **Docker Network Error**: If you see `network "zapstro-network" declared as external, but could not be found`, make sure you run `docker network create zapstro-network`.
- **Prisma Client Issues**: Run `pnpm prisma generate` inside the `zapstro/` folder to rebuild the database client.
- **Port Conflicts**: Ensure ports `3000`, `3001`, `4060`, `5432`, `3306`, `9000`, `8123`, `1433`, `6379`, and `9443` are free on your host machine.
