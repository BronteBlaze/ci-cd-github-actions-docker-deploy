## 🚀 CI/CD Pipeline: Deploy to Production Server via GitHub Actions

This repository demonstrates a real-world CI/CD pipeline for automated production deployments using GitHub Actions, Docker Compose, SSH, and Cloudflare.

### 🔄 Trigger

- The workflow runs on every push to the `production` branch.

### 🔧 Steps Breakdown

1.  **Checkout Code & Submodules**

    - Clones the main repository including all submodules.
    - Useful when frontend/backend are managed as separate repos.
    - name: Checkout Code
      uses: actions/checkout@v3
      with:
      submodules: recursive
      token: ${{ secrets.GH_PAT }}

2.  **SSH Deployment with `appleboy/ssh-action`**

    - Securely SSH into the production server.
    - Pulls latest changes from `production` branch.
    - Updates submodules and fetches the latest commit for the frontend.
    - name: Deploy to Server
      uses: appleboy/ssh-action@v0.1.7
      with:
      host: ${{ secrets.SERVER_IP }}
      username: ${{ secrets.SERVER_USER }}
      key: ${{ secrets.SSH_PRIVATE_KEY }}
      script: | # Pull the latest Update
      cd project_name
      git pull origin master
      git submodule update --init --recursive

3.  **Change Detection**

    - Checks if backend or frontend files have changed since the last commit.
    - FRONTEND_CHANGED=$(git diff --name-only HEAD~1 frontend/)
      BACKEND_CHANGED=$(git diff --name-only HEAD~1 | grep -E '^backend/')

4.  **Conditional Docker Rebuilds**

    - Only rebuilds the affected containers:
    - Backend: Rebuild if any `backend/` files changed.
    - if [ -n "$BACKEND_CHANGED" ]; then
      echo "Backend has changed, rebuilding backend..."
      docker compose -f docker-compose.yml up -d --build --no-deps backend
      fi

5.  **Cloudflare Cache Purge**

    - Automatically purges CDN cache if frontend is updated to reflect the latest UI and to fix some stale cache issues.
    - curl -X POST "https://api.cloudflare.com/client/v4/zones/${{ secrets.CLOUDFLARE_ZONE_ID }}/purge_cache" \
       -H "Authorization: Bearer ${{ secrets.CLOUDFLARE_API_TOKEN }}" \
       -H "Content-Type: application/json" \
       --data '{"purge_everything":true}'

6.  **Cleanup**

    - Runs `docker system prune` to clean unused resources.
    - Only containers that are unused and old are removed.
    - The new one will be active after update.
    - docker system prune -af --volumes=false

7.  **Database Migration**
    - If backend changed, runs migration scripts inside the backend container.
    - if [ -n "$BACKEND_CHANGED" ]; then
      echo "Running database migrations..."
      docker compose -f docker-compose.yml exec backend npm run migration:run:prod
      fi

---

### Docker Compose Template

```bash
version: "3.8"
services:
  backend:
    image: [your_backend_image_name]
    deploy:
      resources:
        limits:
          memory: [memory_limit]
          cpus: [cpu_limit]
    build:
      context: ./backend
      dockerfile: [Dockerfile for production]
    ports:
      - "[host_port:docker_internal_port]" # Map port
    # These are added as per the requirements, these are extra environment variables that are separately
    # inlucuded from env file
    environment:
      - NODE_ENV=[your_application_environment] # production, staging, development
      - NODE_EXTRA=production
      - BASE_URL=[base_url_of_application]
    env_file:
      - ./backend/.env
    volumes:
      - backend_data:/app/data #Persistence Volume Claim
    depends_on:
      - database
    restart: always
    networks:
      - [networkName]

  frontend:
    image: [your_frontend_image_name]
    deploy:
      resources:
        limits:
          memory: [memory_limit]
          cpus: [cpu_limit]
    build:
      context: ./frontend
      dockerfile: [Dockerfile for production]
    ports:
      - [host_port:docker_internal_port]
    # These are added as per the requirements, these are extra environment variables that are separately
    # inlucuded from env file
    environment:
      - NEXT_ENV=production
    env_file:
      - ./frontend/.env
    depends_on:
      - backend
    restart: always
    networks:
      - [networkName]

  database:
    image: [your_database_image_from_docker_hub]
    deploy:
      resources:
        limits:
          memory: [your_frontend_image_name]
          cpus: [memory_limit]
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER} # Database user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD} # Database Password
      POSTGRES_DB: ${POSTGRES_DB} # Database name
    ports:
      - [host_port:docker_internal_port]
    volumes:
      - db-data:/var/lib/postgresql/data # Persistence Volume Claim
    networks:
      - [networkName]

# Mount the volumes
volumes:
  db-data:
  backend_data:

# Optional, this makes docker network [networkName] shared between different docker compose services
networks:
  networkName:
     external: true
```

### CI/CD Template

```bash
name: Deploy to Server

on:
  push:
    branches:
      - master

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
        with:
          submodules: recursive
          token: ${{ secrets.GH_PAT }}

      - name: Deploy to Server
        uses: appleboy/ssh-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            # Pull the latest Update
            cd [project_name]
            git pull origin master
            git submodule update --init --recursive

            # Update the frontend submodule to the latest remote commit
            git submodule update --remote frontend

            # Check whether frontend and backend files are really changed
            FRONTEND_CHANGED=$(git diff --name-only HEAD~1 frontend/)
            BACKEND_CHANGED=$(git diff --name-only HEAD~1 | grep -E '^backend/')

            # Debugging
            echo "Frontend submodule status: $(git submodule status frontend)"
            echo "Backend changes: $BACKEND_CHANGED"

            # Remove the old containers
            # docker compose -f docker-compose.yml down --volumes=false --remove-orphans

            # If backend changed, rebuild backend
            if [ -n "$BACKEND_CHANGED" ]; then
              echo "Backend has changed, rebuilding backend..."
              docker compose -f docker-compose.yml up -d --build --no-deps backend
            fi

            # If frontend changed, rebuild frontend
            if [ -n "$FRONTEND_CHANGED" ]; then
              echo "Frontend has changed, rebuilding frontend..."
              docker compose -f docker-compose.yml up -d --build --no-deps frontend

              echo "Waiting for frontend container to be healthy..."
              sleep 5

              echo "Purging Cloudflare cache..."
              curl -X POST "https://api.cloudflare.com/client/v4/zones/${{ secrets.CLOUDFLARE_ZONE_ID }}/purge_cache" \
              -H "Authorization: Bearer ${{ secrets.CLOUDFLARE_API_TOKEN }}" \
              -H "Content-Type: application/json" \
              --data '{"purge_everything":true}'
            fi

            # Clean up all unused resources
            docker system prune -af --volumes=false

            # If backend changed, run migrations
            if [ -n "$BACKEND_CHANGED" ]; then
              echo "Running database migrations..."
              docker compose -f docker-compose.yml exec backend npm run migration:run:prod
            fi

```

### 🧠 Key Skills Demonstrated

- SSH deployment automation
- Conditional container builds
- GitHub Actions CI/CD workflows
- Docker Compose production pipeline
- Cloudflare API integration
- Git submodule handling
- Deployment efficiency & optimization
