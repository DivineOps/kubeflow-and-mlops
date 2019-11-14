[![Build Status](https://dev.azure.com/sasrose/kubeflow/_apis/build/status/DivineOps.TacosAndBurritos.CI?branchName=master)](https://dev.azure.com/sasrose/kubeflow/_build/latest?definitionId=86&branchName=master)
# MLOps with Kubeflow, Azure Machine Learning and Azure Pipelines

This repository provides a sample ML CI/CD pipeline using Kubeflow, Azure ML workspaces and Azure Pipelines.

It requires:
- An Azure Account (A trial account works!)
- An Azure DevOps Organization (The free tier works!)

# Slides
Link to the slides to the All Things Open version of this talk [Slides](https://www.slideshare.net/AlexandraRosenbaum/mlops-by-sasha-rosenbaum)

# Deploy Kubeflow

Note: this guide deploys Kubeflow on AKS on Azure platform. For more examples, and guides for other platforms, please refer to the [kubeflow.org getting started page](https://www.kubeflow.org/docs/started/getting-started/)

## Prerequisites

#### Install Azure CLI
[Install Azure CLI on Windows machine](
https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest) 

Install Azure CLI on a Mac:
```
brew update && brew install azure-cli
```
#### Install/Upgrade kubectl
```
az aks install-cli
```

## 1. Create the AKS cluster for Kubeflow

Note: if you already have an AKS cluster, you can skip to the step 1.9.

1. Login
```
az login
```

2. See available subscriptions 
```
az account list -o table
```

3. Choose the required subscription
```
az account set --subscription <name or id>
```

4. Get the location code for the required data center 
```
az account list-locations
```

5. Get the VM size code for the cluster VMs
```
az vm list-sizes --location <region_you_plan_to_use> -o table
```

Note: for this workload the VMs must use premium storage. It is alsonrecommended to use GPU enabled VMs. 

6. Create the resource group
```
az group create -n <resource_group_name> -l <location>
```

7. Create the Service Principal (SP)
```
az ad sp create-for-rbac 
```

The output of the above command will be similar to the below. Please note the GUIDs, as we will use them going forward

```
{
  "appId": "559513bd-0c19-4c1a-87cd-XXXXXXX",
  "displayName": "azure-cli-2019-03-04-21-35-28",
  "name": "http://azure-cli-2019-03-04-21-35-28",
  "password": "e763725a-5eee-40e8-a466-XXXXXX",
  "tenant": "72f988bf-86f1-41af-91ab-XXXXXXX"
}
```

8. Create the cluster
```
az aks create -g <resource_group_name> -n <cluster_name> -s <agent_vm_size> -c <agent_count> -l <location> --generate-ssh-keys --service-principal <appId> --client-secret <password>
```

9. Connect to the AKS cluster you've deployed
```
az aks get-credentials -n <cluster_name> -g <resource_group_name>
```

10. Sanity check that all the base namespaces are running
```
kubectl get all --all-namespaces 
```

## 2. Install Kubeflow

For more examples, and guides for other platforms, please refer to the [kubeflow.org getting started page](https://www.kubeflow.org/docs/started/getting-started/)
1. Download the latest kfctl executable from the [Kubeflow releases page](https://github.com/kubeflow/kubeflow/releases/) to your local machine

#### On a Mac
Use the latest Mac executable, e.g. 
https://github.com/kubeflow/kubeflow/releases/download/v0.6.2/kfctl_v0.6.2_darwin.tar.gz

#### On a Windows machine
Use the [Linux Subsystem](https://docs.microsoft.com/en-us/windows/wsl/install-win10) or the [Azure Cloud Shell](http://shell.azure.com) bash environment.

Use the latest Linux executable, e.g. https://github.com/kubeflow/kubeflow/releases/download/v0.6.2/kfctl_v0.6.2_linux.tar.gz

2. Choose the desired location, and unpack the tar.gz
```
tar -xvf kfctl_<release tag>_<platform>.tar.gz
```

3. Add the kfctl location to PATH
```
export PATH=$PATH:<path to where kfctl was unpacked>
```

4. Choose and set the directory name for Kubeflow setup files
```
export KFAPP=<your choice of application directory name> (ensure the name is lowercase)
```

5. Choose the latest istio config file 
```
export CONFIG=<file path or url to config to use>
```
e.g.
```
export CONFIG=https://raw.githubusercontent.com/kubeflow/kubeflow/v0.6.2/bootstrap/config/kfctl_k8s_istio.0.6.2.yaml
````

6. Generate the configuration files and deploy Kubeflow

Note: make sure that you are connected to the cluster. Refer to step 1.9. for details. 

```
cd ${KFAPP}
kfctl generate all -V
kfctl apply all -V
```

7. Test that kufeflow is now running
```
kubectl get all -n kubeflow
```

8. Connect to the Kubeflow Dashboard

#### Open the dashboard locally
```
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```
And browse to http://localhost:8080 on your local machine

#### Enable external IP in AKS
To expose the Kubeflow Dashboard over an external IP in AKS, modify the type of the istio ingress gateway. To do so, run:
```
kubectl edit -n istio-system svc/istio-ingressgateway
```
And then replace ```type: NodePort``` with ```type: LoadBalancer``` and save the file.

Use the command 
```
kubectl get -w -n istio-system svc/istio-ingressgateway
```
To get the new IP of the Kubeflow Dashboard.

Your dashboard should look similar to the following
![Kubeflow Dashboard](/readme-images/kubeflow-dashboard.png)


## 3. Deploy the additional Azure resources required for the CI/CD pipeline

## Deploy Azure Container Registry

1. Create the Azure Container Registry
```
az acr create -n <acr_name> -g <resource_group_name> --sku Standard 
```

2. Add permissions for the AKS Service Principal (SP) over the Azure Container Registry (ACR)

a. Get the ACR ID
```
ACR_ID=$(az acr show --name $ACR --resource-group <resource_group_name> --query "id" --output tsv)
```

b. Get the SP ID
```
CLIENT_ID=$(az aks show --resource-group <resource_group_name> --name <cluster_name> --query "servicePrincipalProfile.clientId" --output tsv) 
```

Note: do not assume that the SP ID is the same as the one we deployed it with. 

c. Assign the permissions
```
az role assignment create --assignee $CLIENT_ID --role acrpull --scope $ACR_ID
```

Note: lack of appropriate permissions can cause issues with Docker image download. 

d. To double check that the SP has permissions over the ACR, you can log in with the SP and list available 
```
az login --service-principal -u <client_id> -p <sp_secret> -t <tenant_id>
az acr list
```
e. If you need to get/reset the SP password, you can run
```
SP_SECRET=$(az ad sp credential reset --name $CLIENT_ID -query password -o tsv)
```
### Deploy Azure Machine Learning (AML) workspace

3. Install AML CLI extension
```
az extension add -n azure-cli-ml
```

4. Create the AML Workspace
```
az group create --name <resource_group_name> --location <location>
```

## 4. Setup the Kubeflow Pipeline

Note: a similar (but not identical) walkthrough for the Tacos and Buritos model is provided on the [kubeflow website](https://www.kubeflow.org/docs/azure/azureendtoend/). 

1. [Fork](https://help.github.com/en/articles/fork-a-repo#fork-an-example-repository) this repo and clone it
 to your local machine
    ```
    git@github.com:DivineOps/kubeflow-and-mlops.git
    ```

2. Create a persistent volume claim (PVC) in Kubeflow on AKS

    A persistent volume claim is a dynamically provisioned storage resource attached to a Kubernetes cluster. It is used in the pipeline to store data and files across pipeline steps.
    ```
    cd kubernetes
    kubectl apply -f pvc.yaml
    ```

3. Update the pipeline  

    a. Open code/pipeline.py

    b. Update the Azure Container Registry name for the Docker images (for *preprocess*, *training* and *register* containers) to your ACR name

    c. If needed, update the Dockerfiles in the corresponding folders to point to the tensorflow images you wish to use. Note that there are separate GPU and non-GPU images. See the list of the latest tags here https://hub.docker.com/r/tensorflow/tensorflow/tags 

    d. If needed, update the tensorflow version in code/deploy/environment.yml to latest. 

4. Setup the Python environment

    a. Install [Python 3](https://www.python.org/downloads/)

    b. Install required modules
    ```
    pip3 install kubernetes
	pip3 install kfp
    ```

5. Compile the pipeline
    The following command will create a pipeline.py.tar.gz

    ```
    cd code
    python3 pipeline.py
    ```

6. Upload the pipeline

    a. On your Kubeflow Dashboard, click Pipelines -> Upload Pipeline -> Choose file, and upload the pipeline.py.tar.gz you've just created.
    b. Click on Pipelines
    c. Right click on the new Pipeline and select "Copy link address"
    d. Note the Pipeline GUID from the link

    Note: this is the only way I found to get the Pipeline/Experiment GUIDs. Please let me know if you have a better way. 

7. Create an experiment
    a. On your Kubeflow Dashboard, click on the new Pipeline -> Create experiment. 
    b. Give your experiment a name and click Next.
    c. On the next page, click "Skip this step". 
    d. Click on Experiments
    e. Right click on the new Experiment and select "Copy link address"
    f. Note the Experiment GUID fro the link

    Note: we will need the experiment to run the CI pipeline, but we won't be able to create a successful run until we push the Docker images to the registry. 

8. (Optional) Create the docker images locally

    The following steps describe how to create the Docker images locally. The Docker images will also be created by the CI pipeline in Azure Pipelines, so you may skip this step and proceed to part 5. 

    a. [Install Docker](https://docs.docker.com/install/)

    b. Setup variables and login to ACR
    ```
    export REGISTRY_PATH=<registry_name>.azurecr.io
    export VERSION_TAG=latest
    az acr login --name <registry_name>
    ```
    c. Go to the code directory of this repo 

    d. Build and push the *preprocess* Docker image
    ```
    cd preprocess
    docker build . -t ${REGISTRY_PATH}/preprocess:${VERSION_TAG}
    docker push ${REGISTRY_PATH}/preprocess:${VERSION_TAG}
    ```
    e. Build and push the *training* Docker image
    ```
    cd ../training
    docker build . -t ${REGISTRY_PATH}/training:${VERSION_TAG}
    docker push ${REGISTRY_PATH}/training:${VERSION_TAG}
    ```
    f. Build and push the *register* Docker image
    ```
    cd ../register
    docker build . -t ${REGISTRY_PATH}/register:${VERSION_TAG}
    docker push ${REGISTRY_PATH}/register:${VERSION_TAG}
    ```

9. (Optional) Create a run
    If you've  completed step 4.8, you can manually trigger a Kubeflow Pipeline run by clicking on Pipelines -> Experiments -> <Your Experiment> -> Create run and filling in the required parameters. 

    To wire up the run automation, proceed to part 5. 

## 5. Set up the Continuous Integration (CI) pipeline in Azure Pipelines

In this part, we will create a CI pipeline in Azure Pipelines. The CI pipeline will build and push the 3 required Docker images - *preprocess*, *training* and *register*, and then trigger the Kubeflow Pipeline. This part depends on part 4.1-7 being completed successfully.

1. [Create an Azure DevOps Account](https://docs.microsoft.com/en-us/azure/devops/user-guide/sign-up-invite-teammates?view=azure-devops)

2. Set up the pipeline variables

    For security purposes, it is recommended that you save sensitive information like resource names, IDs and passwords in a secret store. You can use Azure DevOps variable groups, Azure KeyVault or another store of your choosing. 

    a. In Azure Pipelines, go to Pipelines -> Library -> Variable Groups -> Add Variable Group. 
    Give the group a name and add the following variables based on the names and IDs from the previous steps of this guide.


    KF_RESOURCE_GROUP - name of the resource group (step 1.6)

    KF_AKS_CLUSTER - name of the kubeflow AKS cluster (step 1.7)

    KF_ACR - name of the Azure Container Registry (step 3.1)

    KF_SUBSCRIPTION_ID - the ID of your Azure Subscription

    KF_TENANT_ID - the ID of your Azure Tenant

    KF_SERVICE_PRINCIPAL_ID - the ID of the Service Principal (step 3.2.b)

    KF_SERVICE_PRINCIPAL_PASSWORD - the password/secret of the SP 

    KF_WORKSPACE - name of the Azure ML Workspace (step 3.3.4)

    KF_DATA_DOWNLOAD - the URL to the zip of the data set

    Note that the TacosAndBurritos pipeline currently uses the data set available at the following URL https://aiadvocate.blob.core.windows.net/public/tacodata.zip

    KF_PIPELINE_ID -  the Kubeflow pipeline ID (4.6.d)

    KF_EXPERIMENT_ID - the Kubeflow experiment ID (step 4.7.f)

    b. Variables in the YAML file
    The following variables are safe to check into source control, and they are already a part of the current YAML pipeline file:

    ```
    KF_BATCH: 32
    KF_EPOCHS: 5
    KF_LEARNING_RATE: 0.0001
    KF_MODEL_NAME: tacosandburritos
    KF_PERSISTENT_VOLUME_NAME: azure
    KF_PERSISTENT_VOLUME_PATH: /mnt/azure
    ```

3. Create the CI pipeline
    
    a. In Azure DevOps, click on Pipelines -> Pipelines -> New Pipeline
    b. In "Where is your code" choose GitHub (YAML)
    c. Choose your fork of this repository as source
    d. In "Configure your pipeline" choose "Existing Azure Pipelines YAML file" 
    e. Choose the master branch
    f. Choose the ```azure-pipelines-tnb.yml``` file 
    g. Click run to run the CI pipeline

Note: you may need to authorize the GitHub repo and the subscription connections. 

4. Validate the Kubeflow run

    The last step of the CI pipeline triggers the Kubeflow Pipeline. Now that our CI pipeline completed successfully, we can check that the Kubeflow Pipeline also completed successfully.

    a. Browse to your Kubeflow Dashboard and open your pipeline and experiment
    b. Validate that the Kubeflow Pipeline ran and completed successfully

## 6. Create the Release (CD) pipeline in Azure Pipelines

1. Install the [Azure Machine Learning extension](https://marketplace.visualstudio.com/items?itemName=ms-air-aiagility.vss-services-azureml) for Azure Devops. 

2. Import the Release Pipeline
    a. In Azure DevOps, click Pipelines -> Releases
    b. Click New -> Import Pipeline 
    c. Browse to this repo/release-pipelines and import ```Tacos vs. Burritos - Release Model.json```

3. Add the ML model artifact 
    Note: this step depends on the Kubeflow pipeline completing sucessfully and AML model being registered in AML workspace.

    a. Delete the existing pipeline artifacts 
    b. Add a new artifact of type AzureML. Select your AML workspace name (same as KF_WORKSPACE)
    ![AML artifact](/readme-images/aml-artifact.png)
    c. Name the source alias *amlmodel* (the name is referenced in pipeline steps)

4. Add the model configuration artifact

    Note: this is somewhat hacky, as the latest model technically doesn't have to match the latest config files in GitHub. 

    a. Add a new artifact of type GitHub
    b. Choose your repository fork
    c. Choose the master branch, and the latest version
    d. Name the source alias *github-kubeflow-and-mlops* (the name is referenced in pipeline steps) 
    
5. Configure the Azure Resource connections
    a. Click into each step of the pipeline and configure the Azure Subscription
    b. The Deploy ACI step will create an ACI environment if it doesn't exist
    c. The Deploy AKS step will *NOT* create the AKS cluster if it doesn't exist. If you do not wish to deploy an AKS cluster, you can disable this pipeline stage. 

6. (Optional) Deploy an AKS cluster to run the model 
For more details on this process, please refer to the [Azure ML docs](https://docs.microsoft.com/en-us/azure/machine-learning/service/how-to-deploy-azure-kubernetes-service)

    a. Deploy a *different* AKS cluster with a *minimum of 12 CPUs* (please refer to part 1 of this guide for AKS deployment instructions) 
    
    b. Attach the cluster as a compute target for Azure ML
    ```
    aksresourceid=$(az aks show -n <new_cluster_name> -g <resource_group_name> --query id)
    az ml computetarget attach aks -n <name> -i aksresourceid -g <resource_group_name> -w <aml_myworkspace_name>
    ```
    Note: I had to copy-paste the resource ID rather than use the variable, probably a quotes issue. 

    Note: the above step takes a few minutes.

    c. Run the following to see if the AKS cluster is ready
    ```
    az ml computetarget show -n <new_cluster_name> -g <resource_group_name> -w <aml_myworkspace_name>
    ```

7. Trigger the pipeline and validate that it ran successfully.

## 7. Troubleshooting

Coming soon
