# CI/CD Process for Deploying a Microservice on Greep

## Overview
At Greep, we use Earthly to manage the CI processes alongside GitHub Actions for automation. This document outlines the steps required to deploy a microservice on the Greep server.

## Prerequisites
To proceed, ensure you have the following installed:
- [Earthly](https://earthly.dev/)
- Docker (You can run a lightweight Docker by installing Colima: `brew install colima`, then start it with `colima start`).

## Deployment Process

### Step 1: Generate Microservice Configuration
1. Clone the `GreepAppConfigs` repository if you haven't already:
   ```sh
   git clone https://github.com/GreepPay/GreepAppConfigs.git
   ```
2. Navigate into the `GreepAppConfigs` folder and run the following command:
   ```sh
   earthly github.com/Doctordrayfocus/K8AutoSetup+install --service=greep-wallet --apptype=bunjs
   ```
   - Replace `greep-wallet` with your microservice name following the convention `greep-[service name]`.
   - Replace `bunjs` with the appropriate application type (options: `bunjs`, `nodejs`, `java`, `php`, `rust`, `python`).

### Step 2: Modify Configuration
1. Update the Docker generation config by modifying:
   ```sh
   /templates/{apptype}/docker/Earthfile
   ```
2. Modify Kubernetes deployment settings, config maps, and secrets in:
   ```sh
   /templates/{apptype}/kubernetes/Earthfile
   ```
3. Edit the root `Earthfile` if necessary.
4. Commit and push your changes to the `main` branch.

### Step 3: Set Up GitHub Actions
1. In the microservice codebase, add the workflow file at `.github/workflows/deploy.yml`:

   ```yaml
   name: Deploy to Staging and Production

    on:
      push:
        branches:
          - dev
          - prod

    jobs:
      deploy-staging:
        if: github.ref == 'refs/heads/dev'
        name: Deploy to Staging
        runs-on: ubuntu-latest

        steps:
          - name: Checkout Repository
            uses: actions/checkout@v4

          - name: Deploy via SSH
            uses: appleboy/ssh-action@v1.0.0
            with:
              host: "${{ secrets.SSH_HOST }}"
              username: "${{ secrets.SSH_USER }}"
              key: "${{ secrets.SSH_PRIVATE_KEY }}"
              script: |
                cd ~/GreepAppConfigs/greep-wallet

                # Collect secrets into EARTHLY_BUILD_ARGS
                export EARTHLY_BUILD_ARGS=""

                # Add secret environment variables to EARTHLY_BUILD_ARGS

                # Default secrets
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_CLIENT_ID=${{ secrets.AZURE_CLIENT_ID }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_CLIENT_SECRET=${{ secrets.AZURE_CLIENT_SECRET }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_KUBERNETES_CLUSTER_NAME=${{ secrets.AZURE_KUBERNETES_CLUSTER_NAME }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_REGISTRY_NAME=${{ secrets.AZURE_REGISTRY_NAME }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_RESOURCE_GROUP=${{ secrets.AZURE_RESOURCE_GROUP }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_SUBSCRIPTION_ID=${{ secrets.AZURE_SUBSCRIPTION_ID }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_TENANT_ID=${{ secrets.AZURE_TENANT_ID }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}SSH_HOST=${{ secrets.SSH_HOST }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}SSH_PRIVATE_KEY=${{ secrets.SSH_PRIVATE_KEY }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}SSH_USER=${{ secrets.SSH_USER }},"

                # Additional secrets
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}DB_HOST=${{ secrets.DB_HOST }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}DB_PASSWORD=${{ secrets.DB_PASSWORD }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}DB_USER=${{ secrets.DB_USER }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}YELLOW_CARD_API_KEY=${{ secrets.YELLOW_CARD_API_KEY_DEV }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}YELLOW_CARD_SECRET_KEY=${{ secrets.YELLOW_CARD_SECRET_KEY_DEV }},"

                # Add general variables to EARTHLY_BUILD_ARGS

                # Default variables
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}CRD_CONTROLLER_NAME=${{ vars.CRD_CONTROLLER_NAME }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}CRD_GROUP=${{ vars.CRD_GROUP }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}CRD_KIND=${{ vars.CRD_KIND }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}DOCKER_REGISTRY=${{ vars.DOCKER_REGISTRY }},"

                # Additional variables
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}YELLOW_CARD_BASE_URL=${{ vars.YELLOW_CARD_BASE_URL_DEV }},"


                # Remove trailing comma (if any)
                export EARTHLY_BUILD_ARGS=$(echo $EARTHLY_BUILD_ARGS | sed 's/,$//')

                # Run earthly with the dynamically created EARTHLY_BUILD_ARGS
                earthly --no-cache +setup --envs=${{ github.ref_name }} --apptype="bunjs" --service="greep-wallet"

                az acr login --name ${{ secrets.AZURE_REGISTRY_NAME }}

                earthly --push +build --envs=${{ github.ref_name }} --version=${{ github.run_number }}  --apptype="bunjs" --service="greep-wallet"

                # Deploy using the EARTHLY_BUILD_ARGS
                earthly --no-cache +deploy --envs=${{ github.ref_name }} --version=${{ github.run_number }}  --apptype="bunjs" --service="greep-wallet"

      deploy-production:
        if: github.ref == 'refs/heads/prod'
        name: Deploy to Production
        runs-on: ubuntu-latest

        steps:
          - name: Checkout Repository
            uses: actions/checkout@v4

          - name: Deploy via SSH
            uses: appleboy/ssh-action@v1.0.0
            with:
              host: "${{ secrets.SSH_HOST }}"
              username: "${{ secrets.SSH_USER }}"
              key: "${{ secrets.SSH_PRIVATE_KEY }}"
              script: |
                cd ~/GreepAppConfigs/greep-wallet

                # Collect secrets into EARTHLY_BUILD_ARGS
                export EARTHLY_BUILD_ARGS=""

                # Add secret environment variables to EARTHLY_BUILD_ARGS

                # Default secrets
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_CLIENT_ID=${{ secrets.AZURE_CLIENT_ID }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_CLIENT_SECRET=${{ secrets.AZURE_CLIENT_SECRET }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_KUBERNETES_CLUSTER_NAME=${{ secrets.AZURE_KUBERNETES_CLUSTER_NAME }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_REGISTRY_NAME=${{ secrets.AZURE_REGISTRY_NAME }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_RESOURCE_GROUP=${{ secrets.AZURE_RESOURCE_GROUP }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_SUBSCRIPTION_ID=${{ secrets.AZURE_SUBSCRIPTION_ID }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}AZURE_TENANT_ID=${{ secrets.AZURE_TENANT_ID }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}SSH_HOST=${{ secrets.SSH_HOST }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}SSH_PRIVATE_KEY=${{ secrets.SSH_PRIVATE_KEY }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}SSH_USER=${{ secrets.SSH_USER }},"

                # Additional secrets
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}DB_HOST=${{ secrets.DB_HOST }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}DB_PASSWORD=${{ secrets.DB_PASSWORD }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}DB_USER=${{ secrets.DB_USER }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}YELLOW_CARD_API_KEY=${{ secrets.YELLOW_CARD_API_KEY_PROD }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}YELLOW_CARD_SECRET_KEY=${{ secrets.YELLOW_CARD_SECRET_KEY_PROD }},"

                # Add general variables to EARTHLY_BUILD_ARGS

                # Default variables
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}CRD_CONTROLLER_NAME=${{ vars.CRD_CONTROLLER_NAME }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}CRD_GROUP=${{ vars.CRD_GROUP }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}CRD_KIND=${{ vars.CRD_KIND }},"
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}DOCKER_REGISTRY=${{ vars.DOCKER_REGISTRY }},"

                # Additional variables
                export EARTHLY_BUILD_ARGS="${EARTHLY_BUILD_ARGS}YELLOW_CARD_BASE_URL=${{ vars.YELLOW_CARD_BASE_URL_PROD }},"


                # Remove trailing comma (if any)
                export EARTHLY_BUILD_ARGS=$(echo $EARTHLY_BUILD_ARGS | sed 's/,$//')

                # Run earthly with the dynamically created EARTHLY_BUILD_ARGS
                earthly --no-cache +setup --envs=${{ github.ref_name }} --apptype="bunjs" --service="greep-wallet"

                az acr login --name ${{ secrets.AZURE_REGISTRY_NAME }}

                earthly --push +build --envs=${{ github.ref_name }} --version=${{ github.run_number }} --apptype="bunjs" --service="greep-wallet" || exit 1

                # Deploy using the EARTHLY_BUILD_ARGS
                earthly --no-cache +deploy --envs=${{ github.ref_name }} --version=${{ github.run_number }}  --apptype="bunjs" --service="greep-wallet"
```
2. Set `--apptype` and `--service` to match your microservice.
3. Note:
   - The `dev` branch represents **staging**.
   - The `prod` branch represents **production**.
   - Add any extra secrets and environment variables required for the service.

### Step 4: Update Database Credentials (For Bun.js Microservices)
1. Update PostgreSQL credentials to use SSL (modify `data-source.ts`).
2. Use the `auth-service` repository as a reference:
   ```
   https://github.com/GreepPay/auth-service
   ```
3. Ensure `src/app.ts` is structured similarly to `auth-service`.
4. Add the database migration script to `package.json`:
   ```json
   {
     "scripts": {
       "migrate:db": "npx typeorm-ts-node-esm migration:run -d src/data-source.ts"
     }
   }
   ```
5. Test your service locally before pushing.

## Final Notes
- Ensure your microservice runs correctly in the local environment before deployment.
- Follow best practices for handling secrets and environment variables.
- For troubleshooting, check logs in GitHub Actions and Kubernetes.

By following this guide, you can efficiently deploy microservices on the Greep server using Earthly and GitHub Actions.
