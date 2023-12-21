
# Openmetadata Deployment on Azure Kubernetes Service Cluster
The repository contains files to set-up Openmetadata on Azure Kubernetes Service cluster and fills the gap in official Openmetadata documentation that lacks set-up instruction for Azure cloud.


### Step 1 - Create a AKS cluster
This step is optional. Openmetadata can also be deployed to an existing AKS cluster.
```azure-cli
az aks create   --resource-group  MyResourceGroup
                --name MyAKSClusterName 
                --nodepool-name agentpool 
                --outbound-type loadbalancer 
                --location germanywestcentral 
                --generate-ssh-keys 
    	        --kubernetes-version 1.25.15 
		        --node-count 1 
		        --enable-addons monitoring 
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
The official documentation of openmetadata recommends setting up external database for the metadata and search. The folowing implementation using the bundled MySQL DB is not recommended for production applications.
```azure-cli
kubectl create secret generic mysql-secrets 
                    --namespace openmetadata 
                    --from-literal=openmetadata-mysql-password=openmetadata_password 
```
```azure-cli
kubectl create secret generic airflow-secrets 
                    --namespace openmetadata 
                    --from-literal=openmetadata-airflow-password=admin 
```
```azure-cli
kubectl create secret generic airflow-mysql-secrets 
                    --namespace openmetadata 
                    --from-literal=airflow-mysql-password=airflow_pass
```
### Step 6 - Install Openmetadata dependencies
The values-dependencies-yaml is used to overwride default values in the official helm chart and must be configured for customozing for use cases. Uncomment the externalDatabase section with meaningful values to connect to external database for production deployments.

```azure-cli
helm install openmetadata-dependencies open-metadata/openmetadata-dependencies --values values-dependencies.yaml --namespace openmetadata
```

For subsequent revisions to the deployment update the yaml file and upgrade the deplyoment using
```azure-cli
helm upgrade --install openmetadata-dependencies open-metadata/openmetadata-dependencies --values values-dependencies.yaml --namespace openmetadata
```

It takes a few minutes for all the pods to be correctly set-up and running.
```azure-cli
kubectl get pods -n openmetadata 
```
```
NAME                                                      READY   STATUS    RESTARTS       AGE
mysql-0                                                   1/1     Running   0              15h
openmetadata-889bdcf8d-pgrvb                              1/1     Running   0              31m
openmetadata-dependencies-db-migrations-676f6bbd8-hp9qc   1/1     Running   0              15h
openmetadata-dependencies-scheduler-8495549bb5-2mqp2      1/1     Running   0              15h
openmetadata-dependencies-sync-users-5c7544d6d7-c54k8     1/1     Running   0              15h
openmetadata-dependencies-triggerer-c4c64f796-p6ghj       1/1     Running   0              15h
openmetadata-dependencies-web-7d8cd694df-l8h8m            1/1     Running   0              22m
opensearch-0                                              1/1     Running   0              23m
```
### References
https://medium.com/syntio/deploying-openmetadata-on-azure-kubernetes-service-270a736679eb
https://www.syntio.net/en/labs-musings/deploying-openmetadata-on-azure-kubernetes-service/
