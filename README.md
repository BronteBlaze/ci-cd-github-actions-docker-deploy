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

### 🧠 Key Skills Demonstrated

- SSH deployment automation
- Conditional container builds
- GitHub Actions CI/CD workflows
- Docker Compose production pipeline
- Cloudflare API integration
- Git submodule handling
- Deployment efficiency & optimization
