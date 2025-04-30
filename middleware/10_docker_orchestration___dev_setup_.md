# Chapter 10: Docker Orchestration & Dev Setup

Welcome to the final chapter! In [Chapter 9: Redux State Management (`web-server/src/slices` & `store`)](09_redux_state_management___web_server_src_slices_____store___.md), we saw how the frontend application keeps its data organized using Redux. We've covered the database, backend API, frontend UI, and everything in between. Now, how do we take all these different pieces – the Python backend, the React frontend, the PostgreSQL database, the Redis cache – and make them run together smoothly, especially when setting up a development environment?

Imagine you just joined the team and want to start coding on the `middleware` project. You'd normally need to install the correct versions of Python, Node.js, PostgreSQL, Redis, run various setup commands, and manage all their configurations just to get the app running. This can be complicated, time-consuming, and different for every developer's computer!

This is the problem that **Docker Orchestration & Dev Setup** solves. Think of it as providing a **pre-packaged workshop-in-a-box** for our application. It uses tools like Docker and Docker Compose to ensure that the application and all its dependencies are bundled together and can be started with simple commands, making the setup process incredibly easy and consistent across different machines.

**Use Case:** A new developer wants to get the `middleware` application running on their local machine for development in just a few minutes, without manually installing or configuring the database, Redis, or specific versions of Python and Node.js.

## Key Concepts

1. **What is Docker? (The Shipping Container)**
    * **Analogy:** Imagine you need to ship a complex machine overseas. Instead of sending all the parts separately and hoping the recipient knows how to assemble them, you build the machine, put it inside a standard shipping container with all its tools and manuals, and ship the whole container. Docker does something similar for software.
    * **Explanation:** Docker allows us to package our application (like the Python backend and React frontend) along with *all* its dependencies (like specific libraries, Python itself, Node.js) into a standardized unit called a **Docker Image**. This image can then be run as a **Docker Container** on any machine that has Docker installed, guaranteeing it runs the same way everywhere.
    * **`Dockerfile`:** The blueprint or instruction manual for building a Docker image is a file named `Dockerfile`. It lists step-by-step commands: start with a base system (like a minimal Python environment), copy the application code in, install required libraries (`pip install`, `yarn install`), build the frontend (`yarn build`), and specify how to run the application.

2. **What does the `middleware` `Dockerfile` do?**
    * Our main `Dockerfile` (and a similar `Dockerfile.dev` for development) creates an image containing:
        * A specific Python version.
        * All Python backend dependencies (from `requirements.txt`).
        * PostgreSQL database server.
        * Redis server.
        * Node.js and Yarn.
        * All frontend dependencies (from `package.json`).
        * The built frontend code (`yarn build`).
        * Helper tools like `dbmate` (for database migrations) and `supervisor` (to run multiple processes like the backend, frontend, db, and redis *within* the single container).
    * This approach (putting everything in one image for development) simplifies the setup compared to managing multiple separate containers.

3. **What is Docker Compose? (The Assembly Instructions)**
    * **Analogy:** Usually, an application isn't just one container. You might have a container for the web app, another for the database, etc. Docker Compose is like the instruction manual for assembling and running a *set* of related containers.
    * **Explanation:** Docker Compose uses a configuration file, typically `docker-compose.yml`, to define all the services (containers) that make up your application, how they connect, which ports should be accessible, and which local folders should be linked to folders inside the containers (called "volumes").
    * **`middleware`'s `docker-compose.yml`:** In our *development* setup, the `docker-compose.yml` is quite simple because the `Dockerfile.dev` already packs most things into one image. It primarily defines:
        * A single service named `middleware-dev`.
        * How to build the image using `Dockerfile.dev`.
        * Which ports on your local machine should connect to ports inside the container (e.g., map local port 3333 to the container's port 3333 for the web app).
        * **Volumes:** Crucially, it maps your local code directories (`./backend`, `./web-server`) into the running container. This means changes you make to the code on your machine are instantly reflected inside the container!
        * **Watch Mode:** It uses `develop: watch:` instructions to automatically sync code changes and sometimes restart processes inside the container when you save files, providing a smooth development workflow.
        * Persistent data volumes (`dev_postgres_data`, `dev_keys`) to ensure your database data and configuration aren't lost when the container stops and starts.

4. **`dev.sh` Script (The Easy Button)**
    * **Explanation:** To make things even simpler, the project includes a script called `dev.sh`. This script automates the common steps needed to start the development environment using Docker Compose. It checks if Docker is running, potentially creates necessary configuration files (like `.env` from `env.example`), and then runs the key command: `docker compose up`.
    * **Role:** It's the recommended "easy button" for developers to start the entire application stack locally.

5. **Gitpod (Cloud Workshop-in-a-Box)**
    * **Explanation:** For developers who might not have Docker installed locally or prefer a cloud-based environment, Gitpod is an option. The `.gitpod.yml` file configures Gitpod to automatically use Docker (defined in the `Dockerfile`) to set up a ready-to-code workspace in the browser. It essentially runs the Docker setup within Gitpod's cloud environment.

## How it Works: Starting the Dev Environment

Let's solve our use case: a new developer wants to run `middleware`. Following the instructions in the `README.md`:

1. **Prerequisite:** Ensure [Docker Desktop](https://www.docker.com/products/docker-desktop/) is installed and running on their machine.
2. **Clone:** Get the project code:

    ```bash
    git clone https://github.com/middlewarehq/middleware
    cd middleware
    ```

3. **Run the Magic Script:** Execute the development script:

    ```bash
    ./dev.sh
    ```

**What happens when you run `./dev.sh`?**

1. **Script Execution:** The shell script starts.
2. **Checks (Optional):** It might check if Docker is running.
3. **Environment Setup (Optional):** It might copy `env.example` to `.env` if `.env` doesn't exist, providing default environment variables.
4. **Docker Compose Up:** The core step! The script runs a command similar to `docker compose -f docker-compose.yml up --build`.
5. **Compose Reads Config:** Docker Compose reads the `docker-compose.yml` file.
6. **Image Build:** It sees the `middleware-dev` service needs an image built using `Dockerfile.dev`.
    * Docker follows the steps in `Dockerfile.dev`: installs system dependencies, Python, Node.js, Python packages, Yarn packages, builds the frontend, sets up Supervisor, etc. (This might take a while the first time). If the image was already built and hasn't changed, Docker uses the cached version.
7. **Container Creation/Start:** Docker creates and starts a container based on the built image.
8. **Port Mapping:** It connects the ports defined in `docker-compose.yml` (e.g., host `3333` to container `3333`).
9. **Volume Mounting:** It mounts the local code folders (`backend`, `web-server`) into the specified locations within the container. It also connects the persistent volumes (`dev_postgres_data`, `dev_keys`).
10. **Container Command:** The container runs the `CMD` specified in the Dockerfile (`/app/setup_utils/start.sh`).
11. **Supervisor Starts Services:** The `start.sh` script uses `supervisord` (configured via `/etc/supervisord.conf`) to start and manage all the necessary processes *inside* the container:
    * PostgreSQL server
    * Redis server
    * Backend Flask API server (`analytics_server`)
    * Backend Flask Sync server (`sync_server`)
    * Frontend Next.js server (`web-server` in dev mode)
    * Cron daemon (for scheduled tasks)
12. **Ready!** The terminal running `./dev.sh` shows logs from all these services. The application is now running!

**The Result:**

* The application is accessible in the browser at `http://localhost:3333`.
* The backend API is running on `http://localhost:9696`.
* The sync server is running on `http://localhost:9697`.
* The developer can edit code in `./backend` or `./web-server` locally, and thanks to the mounted volumes and `develop: watch:`, changes are reflected quickly in the running application inside the container.
* Database data is stored in the `dev_postgres_data` Docker volume, so it persists even if the container is stopped and restarted.

## Under the Hood: File Deep Dive

Let's peek at simplified versions of the key files:

**1. `Dockerfile` (Blueprint for the Image)**

This file defines *how* to build the single image containing everything.

```dockerfile
# Dockerfile (Simplified for illustration - uses Dockerfile.dev in practice)

# Use a specific Python version as the base
FROM python:3.9-slim

# Set environment variables (e.g., for database connection, ports)
ENV DB_HOST=localhost
ENV DB_PORT=5434
# ... other ENVs

# Set working directory inside the image
WORKDIR /app

# --- Backend Setup ---
# Copy backend code and requirements file
COPY ./backend /app/backend
# Create a Python virtual environment and install dependencies
RUN python3 -m venv /opt/venv && \
    /opt/venv/bin/pip install -r /app/backend/requirements.txt

# --- System Dependencies ---
# Install Postgres, Redis, Node.js, Supervisor etc.
RUN apt-get update && apt-get install -y --no-install-recommends \
    postgresql redis-server nodejs supervisor cron curl \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs  # Ensure correct Node version

# --- Frontend Setup ---
COPY ./web-server /app/web-server
# Install frontend dependencies and build the frontend
RUN cd /app/web-server && \
    npm install --global yarn --force && \
    yarn install --frozen-lockfile && yarn build

# --- Configuration ---
# Copy supervisor config, startup scripts, cron jobs
COPY ./setup_utils /app/setup_utils
RUN mv /app/setup_utils/supervisord.conf /etc/supervisord.conf && \
    mv /app/setup_utils/cronjob.txt /etc/cron.d/cronjob && \
    chmod +x /app/setup_utils/*

# --- Database Migration Setup ---
# Install dbmate tool
RUN curl -fsSL -o /usr/local/bin/dbmate https://github.com/.../dbmate-linux-amd64 && \
    chmod +x /usr/local/bin/dbmate

# Add venv, postgresql bins to the PATH
ENV PATH="/opt/venv/bin:/usr/lib/postgresql/15/bin:$PATH"

# Expose ports for frontend, backend API, sync server
EXPOSE 3333 9696 9697

# Command to run when the container starts (uses Supervisor)
CMD ["/bin/bash", "-c", "/app/setup_utils/start.sh"]
```

**Explanation:**

* It starts from a base Python image.
* Sets `ENV` variables used by processes inside the container.
* Copies backend code and installs Python dependencies (`pip install`).
* Uses `apt-get` to install system packages like the database (PostgreSQL), cache (Redis), Node.js, Supervisor (a process manager), and `cron` (for scheduled tasks).
* Copies frontend code and installs dependencies (`yarn install`), then builds the static assets (`yarn build`).
* Copies configuration files for Supervisor and cron.
* Installs `dbmate` for managing database schema changes.
* Sets the `PATH` so commands like `python`, `psql` work easily.
* `EXPOSE` informs Docker which ports the container *intends* to use.
* `CMD` specifies the default command to run when the container starts, which is our `start.sh` script that launches Supervisor.

**2. `docker-compose.yml` (Defining the Dev Service)**

This file tells Docker Compose how to run the service using the image built from `Dockerfile.dev`.

```yaml
# docker-compose.yml (Simplified for illustration)

volumes:
  # Define named volumes for persistent data
  dev_postgres_data:
    driver: local
  dev_keys:
    driver: local

services:
  # Define the main development service run by 'dev.sh'
  middleware-dev:
    container_name: middleware-dev
    build:
      # Use the Dockerfile.dev in the current directory (.)
      context: ./
      dockerfile: Dockerfile.dev
      args:
        # Pass build arguments (can customize build)
        ENVIRONMENT: ${ENVIRONMENT:-dev}
    # Load environment variables from .env file into the container
    env_file:
      - .env
    ports:
      # Map host ports to container ports (host:container)
      - "127.0.0.1:${ANALYTICS_SERVER_PORT:-9696}:${ANALYTICS_SERVER_PORT:-9696}" # Backend API
      - "127.0.0.1:${SYNC_SERVER_PORT:-9697}:${SYNC_SERVER_PORT:-9697}"       # Backend Sync
      - "127.0.0.1:${PORT:-3333}:${PORT:-3333}"                               # Frontend
      - "127.0.0.1:${DB_PORT:-5434}:${DB_PORT:-5434}"                        # Database
      - "127.0.0.1:${REDIS_PORT:-6385}:${REDIS_PORT:-6385}"                    # Redis
    volumes:
      # Mount persistent volumes for data
      - dev_postgres_data:/var/lib/postgresql/15/main
      - dev_keys:/app/backend/analytics_server/mhq/config
    # --- Docker Compose Watch Mode ---
    develop:
      watch:
        # When backend code changes, sync it into the container
        - action: sync
          path: ./backend/analytics_server
          target: /app/backend/analytics_server
          ignore: [venv, __pycache__] # Don't sync venv or pycache
        # When requirements change, rebuild the image
        - action: rebuild
          path: ./backend/requirements.txt
        # When frontend code changes, sync it into the container
        - action: sync
          path: ./web-server
          target: /app/web-server
          ignore: [node_modules]
        # When supervisor config changes, sync and restart the service
        - action: sync+restart
          path: ./setup_utils/supervisord.conf
          target: /etc/supervisord.conf
```

**Explanation:**

* `volumes:` defines Docker-managed volumes (`dev_postgres_data`, `dev_keys`) that persist outside the container lifecycle.
* `services:` defines our single development service `middleware-dev`.
* `build:` tells Compose to build the image using `Dockerfile.dev`.
* `env_file: .env` loads environment variables from the `.env` file (created by `dev.sh` if needed) into the container, configuring database connections, etc. (See [Chapter 5: Configuration Settings (`mhq/service/settings`)](05_configuration_settings___mhq_service_settings___.md)).
* `ports:` maps ports from your local machine (`127.0.0.1:xxxx`) to the corresponding ports inside the container where the services are listening.
* `volumes:` under the service maps the persistent volumes to the correct paths inside the container where Postgres stores data and where the app expects keys.
* `develop: watch:` is key for development. It tells Docker Compose to:
  * `sync`: Copy changes from your local `backend` and `web-server` folders into the running container almost instantly.
  * `rebuild`: If critical files like `requirements.txt` change, trigger a rebuild of the Docker image on the next run.
  * `sync+restart`: If config files like `supervisord.conf` change, sync them and restart the container's main process.

**3. `dev.sh` (The Starter Script)**

A simplified view of the script logic.

```bash
#!/bin/bash

# Check if Docker command exists
if ! command -v docker &> /dev/null
then
    echo "Error: Docker is not installed or not in PATH. Please install docker."
    exit 1
fi

# Check if Docker Compose command exists
if ! command -v docker compose &> /dev/null # Note: space between 'docker' and 'compose'
then
    echo "Error: Docker Compose V2 is not installed or enabled."
    exit 1
fi

# Check if Docker daemon is running
if ! docker info &> /dev/null
then
  echo "Error: Docker daemon is not running."
  exit 1
fi

# Create .env from example if it doesn't exist
if [ ! -f .env ]; then
    echo "Creating .env file from env.example..."
    cp env.example .env
fi

# Run Docker Compose in detached mode (-d) initially, then follow logs
echo "Starting middleware development environment..."
docker compose -f docker-compose.yml up --build --remove-orphans -d

# Attach to logs (or run compose watch if available/intended)
echo "Attaching to logs..."
docker compose logs -f
```

**Explanation:**

* It performs basic checks for Docker and Docker Compose.
* It ensures a `.env` file exists for configuration.
* It runs `docker compose up --build` which tells Docker Compose to:
  * Use the `docker-compose.yml` file.
  * Build the image if it's missing or outdated (`--build`).
  * Start all defined services (`up`).
  * Initially run in the background (`-d`, detached).
* Finally, it runs `docker compose logs -f` to show the combined logs from the running services in the terminal. (The actual script in the repo might use `docker compose watch` directly, which handles both startup and log tailing).

## Conclusion

Docker and Docker Compose are essential tools in the `middleware` project for managing the application environment.

* **Docker (`Dockerfile`)** packages the application and all its dependencies (backend, frontend, database, Redis, specific library versions) into a single, portable image.
* **Docker Compose (`docker-compose.yml`)** defines how to run the application's services (in our dev case, a single service built from the Dockerfile) and configure ports, volumes for code syncing and data persistence.
* The **`dev.sh` script** provides a simple command for developers to start the entire environment, leveraging Docker Compose.
* This setup ensures **consistency** (runs the same way everywhere), **simplicity** (eliminates complex manual setup), and **isolation** (dependencies don't interfere with other projects on your machine).

By using this containerized approach, new developers can get up and running quickly, and the development environment closely mirrors how the application might be deployed, reducing potential issues. This concludes our journey through the `middleware` project's architecture! We hope these chapters have given you a clear understanding of how all the pieces fit together.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)
