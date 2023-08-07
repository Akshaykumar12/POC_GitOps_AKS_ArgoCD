### Pull-based CI/CD Architecture and Dataflow

![Figure 2 - Option 2 Pull based Architecture with GitHub Actions for CI and Argo CD for CD](./media/72be57feef5bb9b47658cfc16f3d779f3.png)

This scenario covers a pull-based DevOps pipeline for a web application with a front-end component. This pipeline uses GitHub Actions for build and push it uses Argo CD a GitOps operator pull/sync for deployment. The data flows through the scenario as follows:

1.  The App code is developed.
1.  The App code is committed to the GitHub git repository.
1.  GitHub Actions Builds a container image from the App code and pushes the container image to Azure Container Registry.
1.  GitHub Actions Updates a Kubernetes Manifest Deployment file with the current image version based on the version number of the container image in the Azure Container Registry.
1.  The GitOps Operator Argo CD syncs / pulls with the Git repository.
1.  The GitOps Operator Argo CD deploys the app to the AKS cluster.

## Deploy this scenario

## Pull-based CI/CD(GitOps)

This article outlines how to deploy your workload using the pull option as described in the [CI/CD pipeline for container-based workloads](https://learn.microsoft.com/azure/architecture/example-scenario/apps/devops-with-aks) article. To deploy this scenario, follow the prerequisites steps outlined [here](README.md) (if you haven't already), then perform the following steps:

This article outlines how to deploy your workload using the push option as described in the [CI/CD pipeline for container-based workloads](https://learn.microsoft.com/azure/architecture/example-scenario/apps/devops-with-aks) article. To deploy this scenario, follow the prerequisites steps outlined [here](README.md) (if you haven't already), then perform the following steps:

1. Go to Actions on the forked repo and enable Workflows as shown: <https://github.com/YOURUSERNAME/aks-baseline-automation/actions>
   ![](media/c2a38551af1c5f6f86944cedc5fd660a.png)
2. Go to **Settings** on the forked repo and create a new environment
    1. You can access the page directly from here: https://github.com/YOUR-REPO/settings/environments/new
    2. Click the **New environment** button
    3. Name it "prod"
3. Set Azure subscription
    1. In Azure cloud shell run
       ```bash
       az account show  # Show current subscription

       az account set --subscription "YOURAZURESUBSCRIPTION" *\#Set a subscription to be the current active subscription*
       ```
    2. Create a file called `ghToAzAuth.sh` in your bash working directory and copy the code block in [this .md file](https://github.com/Azure/aks-baseline-automation/blob/main/IaC/oidc-federated-credentials.md) into it. You will need to update the following variable values:
       ```bash
       APPNAME=myApp
       RG=<AKS resource group name>
       GHORG=<your github org or user name>
       GHREPO=aks-baseline-automation
       GHBRANCH=main
       GHENV=prod
       ```
    3. Save the shell script after you have made the updates to those variables and run the script in your cloud shell
       ```bash
       bash ghToAzAuth.sh
       ```
       This script will create the federated credentials in Azure AD for you. Navigate to **Azure Portal \>  Azure Active Directory \> App registrations \> YOURREGISTEREDAPPNAME \| Certificates & secrets**.
       You should see the following 3 Federated credentials similar to what is shown *in* the following screenshot:
       ![](media/0664a3dd619ba6e98b475b29856e6c57.png)
       Next you need to create the Environment and GitHub Actions Repository secrets *in* your repo.
4. Add Actions secrets for the "Prod" Environment you created previously per [these instructions](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Clinux#create-github-secrets):
    1. Navigate to Github Actions Secrets: from your repo select **Settings > Environments > Prod**.
    2. Click **Add secret** and create the following 3 secrets:
       ```bash
       # The values should be in the following format shown in these examples.
       # Make sure you replace these values with the ones from your own environment
        AZURE_CLIENT_ID = 1gce4f22-5ca0-873c-54ac-b451d7f73e622
        AZURE_TENANT_ID: 43f977bf-83f1-41zs-91cg-2d3cd022ty43
        AZURE_SUBSCRIPTION_ID: C25c2f54-gg5a-567e-be90-11f5ca072277
       ```
       ![](media/a1026d5ff5825e899f2633c2b10177df.png)
    3. When *done* you should see the following secrets *in* your GitHub Settings:
       ![](media/049073d69afee0baddf4396830c99f17.png)
5. Customize the deployment files (if needed)
   
   If you are using the IaC option, you will need to update the *workloads/flask/ingress.yaml* file to use the traefik ingress option by commenting out the *Http agic* ingress and uncommenting the *Https traefik* ingress. You will also need to update the fqdn in Https traefik to match the configuration you have in your Application gateway. For the quick option with AKS Construction helper, no change is required.
6. Install Argo CD on your AKS cluster by following the steps in [Get Started with Argo CD](https://argo-cd.readthedocs.io/en/stable/getting_started/).
   
7. Run the [.github/workflows/App-Flask-GitOps.yml](../../.github/workflows/App-Flask-GitOps.yml) workflow by clicking on **Actions** and selecting the display name for this workflow, which is **App Deploy Flask - GitOps**. This workflow will build and push the container image to the Azure Container Registry (ACR), then it will update the application *deployment.yaml* file with the names of the pushed image. 
   
    Enter the parameters requested by this workflow as shown below:
       ![](media/b4bf25dc9497c669d54a205648cb864c.png)
8. Create a new app for the App in Argo CD by following [these steps](https://argo-cd.readthedocs.io/en/stable/getting_started/#creating-apps-via-ui). Make sure you enter the following parameters:
   - Application Name: flask
   - Project Name: default
   - Sync Policy: Automatic
   - Source
     - Repository URL: https://github.com/YOURREPO/aks-baseline-automation.git
     - Revision: HEAD
     - Path: workloads/flask
   - Destination:
     - Cluster URL: https://kubernetes.default.svc
     - Namespace: default

    After the app is created, click on the **SYNC** button in the Argo CD portal to deploy it.

    This is an example of the successful App in Argo CD:

![](media/58af037d65b2303dbb1c2d4196ac300f.png)
![](media/66908c97c321303ba2bcd58ba6431bdd.png)

9. Check the deployment from the Azure portal as in [Option #1 Push-based CI/CD](./app-flask-push-dockerbuild.md) to make sure that the application was successfully deployed. Also test that you can access it. 
