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
