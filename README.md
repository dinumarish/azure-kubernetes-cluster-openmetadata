
# Openmetadata Deployment on Azure Kubernetes Service Cluster
The repository contains files to set-up Openmetadata on Azure Kubernetes Service cluster and fills the gap in official Openmetadata documentation that lacks set-up instruction for Azure cloud.


### Step 1 - Create a AKS cluster
This step is optional. Openmetadata can also be deployed to an existing AKS cluster.
```azure-cli
az aks create   --resource-group  MyResourceGroup    \
                --name MyAKSClusterName              \
                --nodepool-name agentpool            \
                --outbound-type loadbalancer         \
                --location germanywestcentral        \
                --generate-ssh-keys                  \
    	        --kubernetes-version 1.25.15         \
		        --node-count 1                       \
		        --enable-addons monitoring           \
		        --aks-custom-headers EnableAzureDiskFileCSIDriver=true
```
For existing cluster it is important to enable the CSI storage drivers
```azure-cli
az aks update -n myAKSCluster -g myResourceGroup --enable-disk-driver --enable-file-driver
```

### Step 2 - Create a Namespace (optional)
```azure-cli
kubectl create namespace openmetadata
```

### Step 3 - Create Persistent Volume Claims
The `logs_dags_pvc.yaml` in the root of the repository has the manifest to create the PVC. Update the namespace in the yaml file with the namespace created in step 2 above. 
```azure-cli
kubectl apply -f logs_dags_pvc.yaml
```

### Step 4 - Change owner and update permission for persistent volumes
Airflow pods run as non-root user and lack write access to our persistent volumes. To fix this we create a job `permissions_pod.yaml` that runs a pod that mounts volumnes into the persistent volume claim and updates the owner of the mounted folders /airflow-dags and /airflow-logs to user id 5000, which is the default linux user id of Airflow pods.
```azure-cli
kubectl apply -f permissions_pod.yaml
```

### Step 5 - Add the Helm Openmetadata repo and set-up secrets
#### Add Helm Repo
``` azure-cli
helm repo add open-metadata https://helm.open-metadata.org/
```
#### Create secrets
The official documentation of openmetadata recommends setting up external database for the metadata and search. The folowing implementation uses external postgresql DB from Azure Database.

```azure-cli
kubectl create secret generic airflow-secrets                                    \
                    --namespace openmetadata                                     \
                    --from-literal=openmetadata-airflow-password=<AdminPassword> 
```
For production deployments connecting external postgresql database provide external database connection details by settings up appropriate secrets as below to use in manifests.

```azure-cli
kubectl create secret generic postgresql-secret                                       \
                                --namespace openmetadata                              \
                                --from-literal=postgresql-password=<MyPGDBPassword>   
 
```

### Step 6 - Install Openmetadata dependencies
The values-dependencies-yaml is used to overwride default values in the official helm chart and must be configured for customizing for use cases. Uncomment the externalDatabase section with meaningful values to connect to external database for production deployments. We set sensitive information like host address, DB name and DB username through the CLI

```azure-cli
helm install openmetadata-dependencies open-metadata/openmetadata-dependencies  
                            --values values-dependencies.yaml                           \
                            --namespace openmetadata                                    \
                            --set mysql.enabled=false                                   \
                            --set airflow.externalDatabase.host=<MyDBHostAddress>       \
                            --set airflow.externalDatabase.user=<MyDBUser>              \
                            --set airflow.externalDatabase.database=<MyDBUser>          

```

For subsequent revisions to the deployment update the yaml file and upgrade the deplyoment using
```azure-cli
helm upgrade --install openmetadata-dependencies open-metadata/openmetadata-dependencies 
                            --values values-dependencies.yaml 
                            --namespace openmetadata
```

It takes a few minutes for all the pods to be correctly set-up and running.
```azure-cli
kubectl get pods -n openmetadata 
```
```
NAME                                                       READY   STATUS    RESTARTS   AGE
openmetadata-dependencies-db-migrations-69fcf8c9d9-ctd2f   1/1     Running   0          4m51s
openmetadata-dependencies-pgbouncer-d9476f85-bwht9         1/1     Running   0          4m54s
openmetadata-dependencies-scheduler-5f785954cb-792ls       1/1     Running   0          4m54s
openmetadata-dependencies-sync-users-b58ccc589-ncb2d       1/1     Running   0          4m47s
openmetadata-dependencies-triggerer-684b8bb998-mbzvs       1/1     Running   0          4m53s
openmetadata-dependencies-web-9f6b4ff-5hfqj                1/1     Running   0          4m53s
opensearch-0                                               1/1     Running   0          42m

```

### Step 7 - Install Openmetadata
Finally install Openmetadata and customizing the apiEndpoints using the `values.yaml` file and set sensitive information like host address, db name and username through the CLI to avoid pushing the information into the repository.
```azure-cli
helm install openmetadata open-metadata/openmetadata    \
                            --values values.yaml        \
                            --namespace openmetadata    \
                            --set openmetadata.config.database.host=<MyDBHostAddress>   \
                            --set openmetadata.config.database.databaseName=<MyDBName>  \
                            --set openmetadata.config.database.auth.username=<MyDBUser> \
                                                       
 ```
### Step 8 - Launch Openmetadata UI

```azure-cli
kubectl port-forward service/openmetadata 8585:http -n openmetadata
```
#### Expose UI 
Alternatively use the `openmetadata_lb.yaml` manifest to expose port 8585 of openmetadata web service to a public IP and port of your choice
```azure-cli
kubectl apply -f openmetadata_lb.yaml
```
Open your browser and go to the specified IP(if exposing UI)/localhost to port 8585 to access openmetadata's UI

### References
1. https://medium.com/syntio/deploying-openmetadata-on-azure-kubernetes-service-270a736679eb
2. https://docs.open-metadata.org/v1.2.x/deployment/kubernetes/eks
3. https://github.com/open-metadata/openmetadata-helm-charts/blob/main/charts/deps/values.yaml
4. https://github.com/airflow-helm/charts/blob/main/charts/airflow/values.yaml