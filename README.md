## Resources Needed
- Azure Container Registry (called rcdemoreg in this example)
- Azure Kubernetes Service cluster (called rcdemokluster in this example)

## Create the Container Repository
`az acr create --resource-group DemoAKS --name rcdemoreg --sku Basic`

Now log in to the Container Registry:
`az acr login --name rcdemoreg`
/*
Login Succeeded
*/

## Create a local Docker Image
Here we are building a Docker Container using the Dockerfile in the project. You do not need to make any changes to the file.

`docker build -t asp-netcore -f Dockerfile .`

## Test your Docker image locally by running a container
Here we run a container from the Docker Image created. We're listening on local port 8080 and forwarding that traffic to Container Port 80, so
if you've got another app running on local Port 8080, please change the port number appropriately. Otherwise, once the Container is running, 
you can navigate to **http://localhost:8080** to see the ASP.NET Core Web App running.

`docker container run -p 8080:80 asp-netcore`


## Push an image to the Registry
`docker tag asp-netcore rcdemoreg.azurecr.io/asp-netcore:v1`

`docker push rcdemoreg.azurecr.io/asp-netcore:v1`
/*
4b0ad09bf643: Pushed
c9ec1f82e805: Pushed
33e20f752cf0: Pushed
17c445eb92ad: Pushing [==================>                                ]  29.12MB/76.77MB
b19f476f8dd3: Pushed
5128b7cb97a9: Pushing [====>                                              ]  3.829MB/41.33MB
87c8a1d8f54f: Pushing [=========>                                         ]  13.04MB/69.23MB
*/

## List the Containers in the ACR
`az acr repository list --name rcdemoreg --output table`
/*
Result
-----------
asp-netcore
*/

## Run the image from the Azure Container Registry
docker run rcdemoreg.azurecr.io/asp-netcore:v1

This will download the image to your local docker registry

Optionally, you can browse to your Azure Container Registry's Repository, and create an Azure Container Instance (ACI), based on that Container image.


# Introducing Azure Kubernetes Service


## Install the Kubernetes CLI
To connect to the Kubernetes cluster from your local computer, you use kubectl, the Kubernetes command-line client.

If you use the Azure Cloud Shell, kubectl is already installed. To use it from your local Linux/Mac/Windows workstation, you need to install it locally using the following command:

`az aks install-cli command`

## Connect to the AKS Cluster you deployed in Azure using kubectl
Once the Kubernetes Cluster has been created, it’s time to connect to it, and for that you’ll need credentials, which are retrieved directly from the Kubernetes cluster. The credentials are merged into your local kube config file.

az aks get-credentials --resource-group DemoAKS --name rcdemokluster
//Merged "rcdemokluster" as current context in /home/reuben/.kube/config

Verify the connection to the cluster by connecting to it and getting node
information for the cluster.

`kubectl get nodes`
/*
NAME                                STATUS   ROLES   AGE     VERSION
aks-agentpool-32976850-vmss000000   Ready    agent   6m56s   v1.19.3
*/

## Attach the AKS Cluster to the Azure Container Registry
`az aks update -n rcdemokluster -g DemoAKS --attach-acr rcdemoreg`

/*
AAD role propagation done[############################################]  100.0000%
*/

## Apply the changes to the AKS Cluster
Apply the manifest deployment file to Kubernetes by using the kubectl command
`kubectl apply -f deployment.yaml`
/*
deployment.apps/asp-netcore-deployment created
service/asp-netcore created
*/

Once applied, watch the Loadbalancer service provision a public-ip with the command:
`kubectl get service asp-netcore –-watch`

Navigate to the public IP, and you should see your ASP.NET Core app loaded!


## Set the number of Pods
Scaling out an application by increasing the number of pods is simple. Either modify the deployment descriptor and submit it again (kubectl apply -f deployment.yaml), or simply tell Kubernetes to scale the pods directly via a kubectl command:

`kubectl scale --replicas=5 deployment/asp-netcore-deployment`

/*
deployment.extensions/coreasp-deployment scaled
*/

## Get the number of Pods running
To see the new pods, run the kubectl get pods command again, and you should see a total of 5 pods:

`kubectl get pods`

/*
NAME                                      READY   STATUS    RESTARTS   AGE
asp-netcore-deployment-5cf5bd446d-5nspl   1/1     Running   0          2s
asp-netcore-deployment-5cf5bd446d-kk9rv   1/1     Running   0          4h16m
asp-netcore-deployment-5cf5bd446d-kzfwk   1/1     Running   0          2s
asp-netcore-deployment-5cf5bd446d-nqtqx   1/1     Running   0          4h16m
asp-netcore-deployment-5cf5bd446d-zshds   1/1     Running   0          2s
*/

## Set the number of Nodes
The example we’ve built create a node cluster consisting of 2 nodes, but you can adjust the number of nodes manually if you plan more or fewer container workloads on your cluster.

The following example increases the number of nodes to 3 in the Kubernetes cluster named rcakscluster. The command takes a couple of minutes to complete.

`az aks scale --resource-group kuber --name rcakscluster --node-count 3`

The command should take a few minutes to execute and running kubectl get nodes should now show three nodes present in the cluster. Scaling it back down is just as easy.