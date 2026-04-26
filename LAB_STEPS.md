# Deploy to Azure — Lab Steps

**Source:** https://github.com/skills/deploy-to-azure  
**Goal:** Create two deployment workflows using GitHub Actions and Microsoft Azure.

---

## Overview

In this course, you will:

1. Configure a job
2. Set up an Azure environment
3. Spin up the environment
4. Deploy to staging
5. Deploy to production
6. Destroy the environment

---

## Step 1: Trigger a job based on labels

**What you'll learn:** Use labels on pull requests to conditionally trigger GitHub Actions jobs.

### Activity 1: Configure `GITHUB_TOKEN` permissions

1. Go to **Settings > Actions > General**.
2. Under **Workflow permissions**, enable **Read and write permissions** for `GITHUB_TOKEN`. This is required to upload images to the container registry.

### Activity 2: Configure a trigger based on labels

1. Go to the **Actions** tab and click **New workflow**.
2. Search for "simple workflow" and click **Configure**.
3. Name the workflow `deploy-staging.yml`.
4. Remove all existing triggers and jobs, then add a conditional that filters the `build` job when a label named `stage` is present:

   ```yaml
   name: Stage the app

   on:
     pull_request:
       types: [labeled]

   jobs:
     build:
       runs-on: ubuntu-latest

       if: contains(github.event.pull_request.labels.*.name, 'stage')
   ```

5. Click **Start commit**, create a new branch named `staging-workflow`, and click **Propose changes**.
6. Click **Create pull request**.
7. Wait ~20 seconds and refresh the page; GitHub Actions will advance to the next step.

---

## Step 2: Set up an Azure environment

**What you'll learn:** Store Azure credentials as GitHub secrets and expand the staging workflow with build, Docker image, and deployment jobs.

### Key Actions used

- `actions/checkout`
- `actions/upload-artifact` / `actions/download-artifact`
- `docker/login-action`
- `docker/build-push-action`
- `azure/login`
- `azure/webapps-deploy`

### Activity 1: Store credentials in GitHub secrets and finish the workflow

1. Create an [Azure account](https://azure.microsoft.com/en-us/free/) if you don't have one.
2. Create a new subscription in the Azure Portal (must be "Pay as you go").
3. Install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) and run `az login`.
4. Note your **Subscription ID** (`AZURE_SUBSCRIPTION_ID`).
5. Run the following to create a service principal:

   ```shell
   az ad sp create-for-rbac --name "GitHub-Actions" --role contributor \
     --scopes /subscriptions/{subscription-id} \
     --sdk-auth
   ```

6. Copy the entire JSON output — this is `AZURE_CREDENTIALS`.
7. In your repo, go to **Settings > Secrets and variables > Actions** and add:
   - `AZURE_SUBSCRIPTION_ID`
   - `AZURE_CREDENTIALS`
8. In your open `staging-workflow` pull request, edit `.github/workflows/deploy-staging.yml` to contain the full workflow below (replace `<username>` with your GitHub username):

   ```yaml
   name: Deploy to staging

   on:
     pull_request:
       types: [labeled]

   env:
     IMAGE_REGISTRY_URL: ghcr.io
     DOCKER_IMAGE_NAME: <username>-azure-ttt
     AZURE_WEBAPP_NAME: <username>-ttt-app

   jobs:
     build:
       if: contains(github.event.pull_request.labels.*.name, 'stage')
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-node@v4
           with:
             node-version: 16
         - name: npm install and build webpack
           run: |
             npm install
             npm run build
         - uses: actions/upload-artifact@v4
           with:
             name: webpack artifacts
             path: public/

     Build-Docker-Image:
       runs-on: ubuntu-latest
       needs: build
       name: Build image and store in GitHub Container Registry
       steps:
         - name: Checkout
           uses: actions/checkout@v4
         - name: Download built artifact
           uses: actions/download-artifact@v4
           with:
             name: webpack artifacts
             path: public
         - name: Log in to GHCR
           uses: docker/login-action@v3
           with:
             registry: ${{ env.IMAGE_REGISTRY_URL }}
             username: ${{ github.actor }}
             password: ${{ secrets.CR_PAT }}
         - name: Extract metadata (tags, labels) for Docker
           id: meta
           uses: docker/metadata-action@v5
           with:
             images: ${{env.IMAGE_REGISTRY_URL}}/${{ github.repository }}/${{env.DOCKER_IMAGE_NAME}}
             tags: |
               type=sha,format=long,prefix=
         - name: Build and push Docker image
           uses: docker/build-push-action@v5
           with:
             context: .
             push: true
             tags: ${{ steps.meta.outputs.tags }}
             labels: ${{ steps.meta.outputs.labels }}

     Deploy-to-Azure:
       runs-on: ubuntu-latest
       needs: Build-Docker-Image
       name: Deploy app container to Azure
       steps:
         - name: "Login via Azure CLI"
           uses: azure/login@v2
           with:
             creds: ${{ secrets.AZURE_CREDENTIALS }}
         - uses: azure/docker-login@v1
           with:
             login-server: ${{env.IMAGE_REGISTRY_URL}}
             username: ${{ github.actor }}
             password: ${{ secrets.CR_PAT }}
         - name: Deploy web app container
           uses: azure/webapps-deploy@v3
           with:
             app-name: ${{env.AZURE_WEBAPP_NAME}}
             images: ${{env.IMAGE_REGISTRY_URL}}/${{ github.repository }}/${{env.DOCKER_IMAGE_NAME}}:${{ github.sha }}
         - name: Azure logout via Azure CLI
           uses: azure/CLI@v2
           with:
             inlineScript: |
               az logout
               az cache purge
               az account clear
   ```

9. Commit the changes to the `staging-workflow` branch.
10. Wait ~20 seconds and refresh; GitHub Actions will advance to the next step.

---

## Step 3: Spin up an environment based on labels

**What you'll learn:** Use labels to create and destroy Azure resources via a workflow.

### Azure resources used

- **Web app** — where the application is deployed
- **Resource group** — a collection of Azure resources
- **App Service plan** — manages billing and runs the web app

### Activity 1: Set up a Personal Access Token (PAT) and configure Azure

1. Create a PAT with `repo` and `write:packages` scopes ([guide](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)).
2. Add a repository secret named `CR_PAT` with the PAT as the value.
3. Create a new branch `azure-configuration` from `main`.
4. In `.github/workflows/`, create a new file `spinup-destroy.yml` with the following content (replace `<username>`):

   ```yaml
   name: Configure Azure environment

   on:
     pull_request:
       types: [labeled]

   env:
     IMAGE_REGISTRY_URL: ghcr.io
     AZURE_RESOURCE_GROUP: cd-with-actions
     AZURE_APP_PLAN: actions-ttt-deployment
     AZURE_LOCATION: '"East US"'
     AZURE_WEBAPP_NAME: <username>-ttt-app

   jobs:
     setup-up-azure-resources:
       runs-on: ubuntu-latest
       if: contains(github.event.pull_request.labels.*.name, 'spin up environment')
       steps:
         - name: Checkout repository
           uses: actions/checkout@v4
         - name: Azure login
           uses: azure/login@v2
           with:
             creds: ${{ secrets.AZURE_CREDENTIALS }}
         - name: Create Azure resource group
           if: success()
           run: |
             az group create --location ${{env.AZURE_LOCATION}} --name ${{env.AZURE_RESOURCE_GROUP}} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}
         - name: Create Azure app service plan
           if: success()
           run: |
             az appservice plan create --resource-group ${{env.AZURE_RESOURCE_GROUP}} --name ${{env.AZURE_APP_PLAN}} --is-linux --sku F1 --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}
         - name: Create webapp resource
           if: success()
           run: |
             az webapp create --resource-group ${{ env.AZURE_RESOURCE_GROUP }} --plan ${{ env.AZURE_APP_PLAN }} --name ${{ env.AZURE_WEBAPP_NAME }} --deployment-container-image-name nginx --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}
         - name: Configure webapp to use GHCR
           if: success()
           run: |
             az webapp config container set --docker-custom-image-name nginx --docker-registry-server-password ${{secrets.CR_PAT}} --docker-registry-server-url https://${{env.IMAGE_REGISTRY_URL}} --docker-registry-server-user ${{github.actor}} --name ${{ env.AZURE_WEBAPP_NAME }} --resource-group ${{ env.AZURE_RESOURCE_GROUP }} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}

     destroy-azure-resources:
       runs-on: ubuntu-latest
       if: contains(github.event.pull_request.labels.*.name, 'destroy environment')
       steps:
         - name: Checkout repository
           uses: actions/checkout@v4
         - name: Azure login
           uses: azure/login@v2
           with:
             creds: ${{ secrets.AZURE_CREDENTIALS }}
         - name: Destroy Azure environment
           if: success()
           run: |
             az group delete --name ${{env.AZURE_RESOURCE_GROUP}} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}} --yes
   ```

5. Commit directly to `azure-configuration`, open a pull request titled `Added spinup-destroy.yml workflow`.

### Activity 2: Apply labels to create resources

1. Replace `<username>` in `spinup-destroy.yml` with your GitHub username and commit to the `azure-configuration` branch.
2. Apply the `spin up environment` label to the open pull request.
3. Wait for the workflow to spin up your Azure environment.
4. Wait ~20 seconds and refresh; GitHub Actions will advance to the next step.

---

## Step 4: Deploy to a staging environment based on labels

**What you'll learn:** Trigger a staging deployment by applying a label to a pull request.

### Activities

1. Create a new branch `staging-test` from `main`.
2. Edit `.github/workflows/deploy-staging.yml` and replace every `<username>` with your GitHub username.
3. Commit the change to the `staging-test` branch.
4. Open a pull request from `staging-test` to `main`.

### Activity 1: Add the proper label to your pull request

1. Ensure `GITHUB_TOKEN` has **Read and write permissions** under **Workflow permissions** (Settings > Actions > General).
2. Create and apply the `stage` label to the open pull request.
3. Wait for the workflow to deploy to Azure. Once successful, you'll see green checks and a deployment URL — play the game!
4. Wait ~20 seconds and refresh; GitHub Actions will advance to the next step.

---

## Step 5: Deploy to a production environment

**What you'll learn:** Deploy to production when commits are merged to `main`.

### Activity 1: Add triggers to the production deployment workflow

1. Create a new branch `production-deployment-workflow` from `main`.
2. In `.github/workflows/`, create `deploy-prod.yml` (same as the staging workflow, but triggered on push to `main` instead of by label). Replace `<username>` with your GitHub username:

   ```yaml
   name: Deploy to production

   on:
     push:
       branches:
         - main

   env:
     IMAGE_REGISTRY_URL: ghcr.io
     DOCKER_IMAGE_NAME: <username>-azure-ttt
     AZURE_WEBAPP_NAME: <username>-ttt-app
   # ... (same jobs as deploy-staging.yml)
   ```

3. Commit to the `production-deployment-workflow` branch.
4. Open a pull request for the branch.

### Activity 2: Merge your pull request

1. Merge the pull request.
2. Wait for the deployment to complete — find the URL in the Actions log.
3. Wait ~20 seconds and refresh; GitHub Actions will advance to the next step.

---

## Step 6: Destroy the Azure environment

**What you'll learn:** Tear down cloud resources to avoid unwanted charges.

### Activity 1: Destroy running resources

1. Apply the `destroy environment` label to the merged `production-deployment-workflow` pull request (re-open it from the **Closed** PR list if needed).
2. Wait for the `destroy-azure-resources` job to complete.
3. Verify the environment is gone by visiting the deployment URL or checking the Azure portal.
4. Wait ~20 seconds and refresh; GitHub Actions will advance to the finish step.

---

## Finish

Congratulations! You've completed the **Deploy to Azure** course. You've learned how to:

- Trigger workflows based on PR labels
- Store credentials securely in GitHub secrets
- Build and push Docker images to GitHub Container Registry (GHCR)
- Deploy containerized applications to Azure App Service
- Destroy cloud resources via automation
