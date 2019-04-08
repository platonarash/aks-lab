
# Azure Docker and AKS Lab - dotnetcore

This is a lab for Windows users running an example Dot net core application in Azure AKS.

Good to have: https://kubernetes.io/docs/reference/kubectl/cheatsheet/

*Prequisites:*

*1. Docker desktop installed*

*2. Admin user on the windows machine. You can use setup your own machine in Azure: Create a  D3_v3 size Azure VM and follow the instructions: https://docs.microsoft.com/en-us/azure/virtual-machines/windows/nested-virtualization#enable-the-hyper-v-feature-on-the-azure-vm* 

*3. Install Azure CLI: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest*

*3. Install AKS CLI - Open Powershell as Admin user then run:
```markdown
    az aks install-cli
```

# Start

This lab will run through the following process:


**1. Build your docker image on your local computer**

**2. Run the container on local computer**

**3. Push your local docker image to Azure Container Registry** 

**4. Setup AKS cluster in Azure**  

**5. Deploy your dotnetcore app to AKS**

**Extras: 6. Use Helm**

**Extras: 6. Use Draft**



# 1. Build local docker image
  
```markdown
1. Clone the git repo: 
   git clone https://github.com/dotnet/dotnet-docker-samples/

2a. cd .\dotnet-docker-samples\aspnetapp\

2b. Check out the "Dockerfile" 

3. docker build -t aspnetapp .

3. docker images 

4. docker run -d -p 8080:80 --name myapp aspnetapp

5. docker container list

6. Browse http://localhost:8080

7. docker stop myapp

8. Should fail to browse http://localhost:8080

8a. Make a hange in file: .\dotnet\dotnet-docker-samples\aspnetapp\Views\Home\About.cshtml

9. Build the image again

10. Run the container again and browsa to  http://localhost:8080 
    About text should have changed.
```


# 2. Push image to Azure Container Registry

```markdown

1.  Create a resource group:
    >az group create --name <Your RG name> --location westeurope

2.  Create Azure Container Registry:
    >az acr create --resource-group <Your RG name> --name <Your ACR name> --sku Basic


3.  Login to your Azure Container Registry:
    >az acr login --name <Your ACR Name>

4a. Check your local docker images
    >docker images

4b. docker tag aspnetapp <Your ACR Name>.azurecr.io/aspnetapp:v1

4c. docker images (same image id but different repository in the list)

5.  docker push <Your ACR Name>.azurecr.io/aspnetapp:v1

6.  az acr repository list --name <Your ACR Name> --output table
```

# 3. Create AKS cluster

```markdown

1. az aks create \
    --resource-group <Your RG name> \
    --name <name of AKS cluster> \
    --node-count 1 \
    --enable-addons monitoring \
    --generate-ssh-keys

2.  az aks get-credentials --resource-group <AKS-Resource-Group> --name <Name-Of-AKS-Cluster>

3a. File change C:\Users\<UserName>\.kube\config

4. kubectl cluster-info

5. kubectl get nodes, kubectl get nodes -o wide

6. kubectl get pods, kubectl get pods -o wide

7. kubectl get deployment

8. kubectl describe pods

9. kubectl get pods --namespace kube-system -o wide

10. kubectl explain pod | more


Upgrade cluster - https://docs.microsoft.com/en-us/azure/aks/upgrade-cluster

12. az aks get-upgrades --resource-group myResourceGroup --name myAKSCluster --output table

13. az aks upgrade --resource-group myResourceGroup --name myAKSCluster --kubernetes-version 1.12.6

Access to K8s dashboard: https://docs.microsoft.com/en-us/azure/aks/kubernetes-dashboard

14. kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboarda

15. az aks browse --resource-group <AKS-ResourceGroup> --name <name of AKS cluster> 
```

# 3. Make AKS able to authenticate to Azure Container Registry (ACR)

The AKS cluster needs rights in order to pull images from the ACR in order to spin up pods.

```markdown

1. $CLIENT_ID=az aks show --resource-group <AKS-ResourceGroup> --name <AKS-cluster-name> --query "servicePrincipalProfile.clientId" --output tsv 

2. $ACR_ID=az acr show --name <ACR-name> --resource-group <AKS-ResourceGroup> --query "id" --output tsv

3. az role assignment create --assignee $CLIENT_ID --role Reader --scope $ACR_ID
```


# 3. Deploy to AKS cluster

The following ConfigMap (https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) lets you deploy 4 replicated Pods from your images that was pushed to ACR.

Copy the text into a file aspnetapp-aks.yaml.

```markdown
apiVersion: v1
kind: Service
metadata:
  name: apsnetapp
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: aspnetapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aspnetapp
spec:
  replicas: 4
  selector:
    matchLabels:
      app: aspnetapp
  template:
    metadata:
      labels:
        app: aspnetapp
    spec:
      containers:
      - name: aspnetapp
        image: <Name of you ACR>.azurecr.io/dotnet/aspnetapp:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 80
```

```markdown
1. kubectl apply -f aspnetapp-aks.yaml
3. kubectl get pods eller kubectl get nodes -o wide
```

# 3. Update you app and deploy to AKS


1. Change the aspnet about file .\dotnet-docker-samples\aspnetapp\Views\Home\About.cshtml

```markdown
@{
    ViewData["Title"] = "THIS TEXT IN CHANGED";
}
```

```markdown
2. docker build -t aspnetapp <Your ACR Name>.azurecr.io/aspnetapp:v2 .  - note the v2 in the end  .

3. docker run (as before - you might need to kill some containers: docker container, docker container rm <containerId>)

4. Browse http://localhost:8080/Home/About - You shold see the changed text

5.  docker push <Your ACR Name>.azurecr.io/aspnetapp:v2
```

6. Change in aspnetapp-aks.yaml - change to v2

```markdown
 spec:
      containers:
      - name: aspnetapp
        image: <Your ACR Name>.azurecr.io/dotnet/aspnetapp:v2     
```
```markdown
7. kubectl -f  aspnetapp-aks.yaml

8. Browse About page in the external endpoint in your cluster, it should have changed. 
```

# 3. HELM & DRAFT

This installation is on windows:

```markdown
1. install https://chocolatey.org/

2a. choco install kubernetes-helm     -  https://chocolatey.org/packages/kubernetes-helm
2b  choco install draft - https://chocolatey.org/packages/draft

3a. helm init
3b. draft init

4. kubectl create serviceaccount --namespace kube-system tiller

Run the following command in Azure Cloud shell: https://shell.azure.com/ - in Bash mode

5. kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}' 

6. On your local computer and in Powershell step into the directoruy: .\dotnet-docker-samples\aspnetapp

7. draft config set registry <Your ACR Name>.azurecr.io

Note that the command --pack=csharp tells draft that this is a charp project, more info at https://github.com/Azure/draft/tree/master/packs

8. draft create --pack=csharp -a aspnetapp2

9. draft up

10. Watch the Kubernetes dashboard and see that there are two deployments side by side, aspnet and aspnetapp2

6. Change in file aspnetapp\values.yaml

-----

type: LoadBalancer

----

7. draft up

8. Watch the Kubernetes dashboard services and an extern endpoint is created. Browse to the endpoint.

```
