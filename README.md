# azure-key-vault-with-aks-managed-identity--postgreqsl

----------------------integrate-azure-key-vault-with-azure-kubernetes-service using managed identity---------- (Successful)--
https://thecodeblogger.com/2020/06/13/managing-azure-key-vault-and-secrets-with-azure-cli/
https://medium.com/swlh/integrate-azure-key-vault-with-azure-kubernetes-service-1a8740429bea
https://www.linkedin.com/pulse/configuring-aks-read-secrets-certificates-from-azure-marasco
https://faun.pub/how-to-use-secrets-from-azure-key-vault-in-azure-kubernetes-service-704973be5fc1
https://rtfm.co.ua/en/kubernetes-clusterip-vs-nodeport-vs-loadbalancer-services-and-ingress-an-overview-with-examples/


az group create --name aks2akvrg --location eastus
az aks create --resource-group aks2akvrg --name myk8s --node-count 1 --generate-ssh-keys --enable-managed-identity --network-plugin azure
az aks get-credentials --resource-group aks2akvrg --name myk8s

az keyvault create --name "myk8skv29" --resource-group "aks2akvrg" --location eastus
az keyvault secret set --vault-name "myk8skv29" --name "ExamplePassword" --value "hVFkk965BuUv"

helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts
helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name --set secrets-store-csi-driver.syncSecret.enabled=true


kubectl get pods

shreya@Azure:~$ kubectl get pods
NAME                                                READY   STATUS    RESTARTS   AGE
csi-secrets-store-provider-azure-1640800209-pktrr   1/1     Running   0          17s
secrets-store-csi-driver-8jqfg                      3/3     Running   0          17s


clientId=`az aks show --name myk8s --resource-group aks2akvrg |jq -r .identityProfile.kubeletidentity.clientId`
nodeResourceGroup=`az aks show --name myk8s --resource-group aks2akvrg |jq -r .nodeResourceGroup` 
subId=`az account show | jq -r .id`
az role assignment create --role "Managed Identity Operator" --assignee $clientId --scope /subscriptions/$subId/resourcegroups/aks2akvrg
az role assignment create --role "Managed Identity Operator" --assignee $clientId --scope /subscriptions/$subId/resourcegroups/$nodeResourceGroup
az role assignment create --role "Virtual Machine Contributor" --assignee $clientId --scope /subscriptions/$subId/resourcegroups/$nodeResourceGroup



helm repo add aad-pod-identity https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts
helm install pod-identity aad-pod-identity/aad-pod-identity

kubectl get pod

shreya@Azure:~$ kubectl get pods
NAME                                                READY   STATUS    RESTARTS   AGE
aad-pod-identity-mic-77b8567cc4-cnj7t               1/1     Running   0          81s
aad-pod-identity-mic-77b8567cc4-nwstq               1/1     Running   0          81s
aad-pod-identity-nmi-t9hm7                          1/1     Running   0          81s
csi-secrets-store-provider-azure-1640800209-pktrr   1/1     Running   0          3m3s
secrets-store-csi-driver-8jqfg                      3/3     Running   0          3m3s

az identity create -g aks2akvrg -n aks2kvIdentity

shreya@Azure:~$ az identity create -g aks2akvrg -n aks2kvIdentity
{
  "clientId": "e4a94d8f-a04f-4c70-ae7d-c59d6d5968c2",
  "clientSecretUrl": "https://control-eastus.identity.azure.net/subscriptions/5558daf3-543e-4f1e-883d-49a9ef02f618/resourcegroups/aks2akvrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aks2kvIdentity/credentials?tid=8ce07706-2f54-46a7-868e-fdea1905827d&oid=6566d828-3645-487f-a0e9-f6f578b02666&aid=e4a94d8f-a04f-4c70-ae7d-c59d6d5968c2",
  "id": "/subscriptions/5558daf3-543e-4f1e-883d-49a9ef02f618/resourcegroups/aks2akvrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aks2kvIdentity",
  "location": "eastus",
  "name": "aks2kvIdentity",
  "principalId": "6566d828-3645-487f-a0e9-f6f578b02666",
  "resourceGroup": "aks2akvrg",
  "tags": {},
  "tenantId": "8ce07706-2f54-46a7-868e-fdea1905827d",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities"
}



clientId=`az identity show --name aks2kvIdentity --resource-group aks2akvrg |jq -r .clientId`
principalId=`az identity show --name aks2kvIdentity --resource-group aks2akvrg |jq -r .principalId`
subId=`az account show | jq -r .id`
az role assignment create --role "Reader" --assignee $principalId --scope /subscriptions/$subId/resourceGroups/aks2akvrg/providers/Microsoft.KeyVault/vaults/myk8skv29
az keyvault set-policy -n myk8skv29 --secret-permissions get --spn $clientId
az keyvault set-policy -n myk8skv29 --key-permissions get --spn $clientId

filename.yaml

apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentity
metadata:
  name: "aks-kv-identity"               
spec:
  type: 0                                 
  resourceID: /subscriptions/<subscription id>/resourcegroups/aks2akvrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aks2kvIdentity
  clientID: "<clientId>"
---
apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentityBinding
metadata:
  name: azure-pod-identity-binding
spec:
  azureIdentity: "aks-kv-identity"      
  selector: azure-pod-identity-binding-selector


echo $clientId
echo $subId

shreya@Azure:~$ echo $clientId
e4a94d8f-a04f-4c70-ae7d-c59d6d5968c2
shreya@Azure:~$ echo $subId
5558daf3-543e-4f1e-883d-49a9ef02f618


filename.yaml

apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentity
metadata:
  name: "aks-kv-identity"               
spec:
  type: 0                                 
  resourceID: /subscriptions/5558daf3-543e-4f1e-883d-49a9ef02f618/resourcegroups/aks2akvrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aks2kvIdentity
  clientID: "e4a94d8f-a04f-4c70-ae7d-c59d6d5968c2"
---
apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentityBinding
metadata:
  name: azure-pod-identity-binding
spec:
  azureIdentity: "aks-kv-identity"      
  selector: azure-pod-identity-binding-selector
  
  
kubectl apply -f filename.yaml

az keyvault show -n myk8skv29 | grep tenantId

shreya@Azure:~$ az keyvault show -n myk8skv29 | grep tenantId
        "tenantId": "8ce07706-2f54-46a7-868e-fdea1905827d"
        "tenantId": "8ce07706-2f54-46a7-868e-fdea1905827d"
    "tenantId": "8ce07706-2f54-46a7-868e-fdea1905827d",
	
	
myfile.yaml

apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: spc-myk8skv29
spec:
  provider: azure
  secretObjects:
  - secretName: test-secret
    data:
    - key: key
      objectName: ExamplePassword
    type: Opaque
  parameters:
    usePodIdentity: "true"                                      
    useVMManagedIdentity: "false"                               
    userAssignedIdentityID: ""                                                                                                
    keyvaultName: "myk8skv29"                                                                
    cloudName: ""                                               
    objects:  |
      array:
        - |
          objectName: ExamplePassword                              
          objectType: secret                                    
          objectVersion: ""                                     
    resourceGroup: "aks2akvrg"                      
    subscriptionId: "5558daf3-543e-4f1e-883d-49a9ef02f618"      
    tenantId: "8ce07706-2f54-46a7-868e-fdea1905827d" 


kubectl apply -f myfile.yaml

vi pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: inject-secrets-from-akv
  labels:
    aadpodidbinding: azure-pod-identity-binding-selector
  
spec:
  containers:
    - name: nginx
      image: nginx
      env:
      - name: SECRET
        valueFrom:
          secretKeyRef:
            name: test-secret
            key: key
      volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: spc-myk8skv29	
		  
kubectl apply -f pod.yaml 

hreya@Azure:~$ kubectl get pods
NAME                                                READY   STATUS    RESTARTS   AGE
aad-pod-identity-mic-77b8567cc4-mphvr               1/1     Running   0          35m
aad-pod-identity-mic-77b8567cc4-rl5z4               1/1     Running   0          35m
aad-pod-identity-nmi-hfm68                          1/1     Running   0          35m
csi-secrets-store-provider-azure-1640707432-cfg4q   1/1     Running   0          50m
inject-secrets-from-akv                             1/1     Running   0          5m56s
secrets-store-csi-driver-4kd2l                      3/3     Running   0          50m

shreya@Azure:~$ kubectl get secret
NAME                                                                TYPE                                  DATA   AGE
aad-pod-identity-mic-token-dfmpz                                    kubernetes.io/service-account-token   3      15m
aad-pod-identity-nmi-token-l9knp                                    kubernetes.io/service-account-token   3      15m
csi-secrets-store-provider-azure-token-7zpdv                        kubernetes.io/service-account-token   3      16m
default-token-xjscp                                                 kubernetes.io/service-account-token   3      20m
secrets-store-csi-driver-token-c5xqd                                kubernetes.io/service-account-token   3      16m
sh.helm.release.v1.csi-secrets-store-provider-azure-1640800209.v1   helm.sh/release.v1                    1      17m
sh.helm.release.v1.pod-identity.v1                                  helm.sh/release.v1                    1      15m

#Show secrets
kubectl exec -it inject-secrets-from-akv ls /mnt/secrets-store/

kubectl exec -it inject-secrets-from-akv -- ls /mnt/secrets-store/

# Verify secrets mentioned in provider class
kubectl exec -it inject-secrets-from-akv -- cat /mnt/secrets-store/ExamplePassword
kubectl exec -it nginx-secrets-store cat /mnt/secrets-store/db-password

-------------------------------------------------------------------------------------

POSTGRESQL with AKS and Key Vault:


install kubectl:

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo cp kubectl /usr/local/bin
export PATH=$PATH:/usr/local/bin
kubectl version
az group create --name aks2akvrg --location eastus
az aks create --resource-group aks2akvrg --name myk8s --node-count 1 --generate-ssh-keys --enable-managed-identity --network-plugin azure
az aks get-credentials --resource-group aks2akvrg --name myk8s

az keyvault create --name "myk8skv29" --resource-group "aks2akvrg" --location eastus
az keyvault secret set --vault-name "myk8skv29" --name "ExamplePassword" --value "hVFkk965BuUv"

az keyvault secret set --vault-name "myk8skv29" --name "dbUserName" --value "postgresql29dec"
az keyvault secret set --vault-name "myk8skv29" --name "dbPassword" --value "Uldebasish@2003"

helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts
helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name --set secrets-store-csi-driver.syncSecret.enabled=true


kubectl get pods


shreya@Azure:~$ kubectl get pods
NAME                                                READY   STATUS    RESTARTS   AGE
csi-secrets-store-provider-azure-1640805259-q2p2x   1/1     Running   0          9s
secrets-store-csi-driver-wzs8k                      3/3     Running   0          9s


clientId=`az aks show --name myk8s --resource-group aks2akvrg |jq -r .identityProfile.kubeletidentity.clientId`
nodeResourceGroup=`az aks show --name myk8s --resource-group aks2akvrg |jq -r .nodeResourceGroup` 
subId=`az account show | jq -r .id`
az role assignment create --role "Managed Identity Operator" --assignee $clientId --scope /subscriptions/$subId/resourcegroups/aks2akvrg
az role assignment create --role "Managed Identity Operator" --assignee $clientId --scope /subscriptions/$subId/resourcegroups/$nodeResourceGroup
az role assignment create --role "Virtual Machine Contributor" --assignee $clientId --scope /subscriptions/$subId/resourcegroups/$nodeResourceGroup



helm repo add aad-pod-identity https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts
helm install pod-identity aad-pod-identity/aad-pod-identity

kubectl get pod

shreya@Azure:~$ kubectl get pod
NAME                                                READY   STATUS    RESTARTS   AGE
aad-pod-identity-mic-77b8567cc4-knkg9               1/1     Running   0          14s
aad-pod-identity-mic-77b8567cc4-r98ld               1/1     Running   0          14s
aad-pod-identity-nmi-m7btj                          1/1     Running   0          14s
csi-secrets-store-provider-azure-1640805259-q2p2x   1/1     Running   0          107s
secrets-store-csi-driver-wzs8k                      3/3     Running   0          107s

az identity create -g aks2akvrg -n aks2kvIdentity

shreya@Azure:~$ az identity create -g aks2akvrg -n aks2kvIdentity
{
  "clientId": "e4a94d8f-a04f-4c70-ae7d-c59d6d5968c2",
  "clientSecretUrl": "https://control-eastus.identity.azure.net/subscriptions/5558daf3-543e-4f1e-883d-49a9ef02f618/resourcegroups/aks2akvrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aks2kvIdentity/credentials?tid=8ce07706-2f54-46a7-868e-fdea1905827d&oid=6566d828-3645-487f-a0e9-f6f578b02666&aid=e4a94d8f-a04f-4c70-ae7d-c59d6d5968c2",
  "id": "/subscriptions/5558daf3-543e-4f1e-883d-49a9ef02f618/resourcegroups/aks2akvrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aks2kvIdentity",
  "location": "eastus",
  "name": "aks2kvIdentity",
  "principalId": "6566d828-3645-487f-a0e9-f6f578b02666",
  "resourceGroup": "aks2akvrg",
  "tags": {},
  "tenantId": "8ce07706-2f54-46a7-868e-fdea1905827d",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities"
}


clientId=`az identity show --name aks2kvIdentity --resource-group aks2akvrg |jq -r .clientId`
principalId=`az identity show --name aks2kvIdentity --resource-group aks2akvrg |jq -r .principalId`
subId=`az account show | jq -r .id`
az role assignment create --role "Reader" --assignee $principalId --scope /subscriptions/$subId/resourceGroups/aks2akvrg/providers/Microsoft.KeyVault/vaults/myk8skv29
az keyvault set-policy -n myk8skv29 --secret-permissions get --spn $clientId
az keyvault set-policy -n myk8skv29 --key-permissions get --spn $clientId

iden.yaml

apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentity
metadata:
  name: "aks-kv-identity"               
spec:
  type: 0                                 
  resourceID: /subscriptions/<subscription id>/resourcegroups/aks2akvrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aks2kvIdentity
  clientID: "<clientId>"
---
apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentityBinding
metadata:
  name: azure-pod-identity-binding
spec:
  azureIdentity: "aks-kv-identity"      
  selector: azure-pod-identity-binding-selector


echo $clientId
echo $subId

shreya@Azure:~$ echo $clientId
e4a94d8f-a04f-4c70-ae7d-c59d6d5968c2
shreya@Azure:~$ echo $subId
5558daf3-543e-4f1e-883d-49a9ef02f618


iden.yaml

apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentity
metadata:
  name: "aks-kv-identity"               
spec:
  type: 0                                 
  resourceID: /subscriptions/5558daf3-543e-4f1e-883d-49a9ef02f618/resourcegroups/aks2akvrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/aks2kvIdentity
  clientID: "e4a94d8f-a04f-4c70-ae7d-c59d6d5968c2"
---
apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentityBinding
metadata:
  name: azure-pod-identity-binding
spec:
  azureIdentity: "aks-kv-identity"      
  selector: azure-pod-identity-binding-selector
  
  
kubectl apply -f iden.yaml

az keyvault show -n myk8skv29 | grep tenantId

shreya@Azure:~$ az keyvault show -n myk8skv29 | grep tenantId

        "tenantId": "8ce07706-2f54-46a7-868e-fdea1905827d"
        "tenantId": "8ce07706-2f54-46a7-868e-fdea1905827d"
    "tenantId": "8ce07706-2f54-46a7-868e-fdea1905827d",
	
	
provider.yaml

apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: spc-myk8skv29
spec:
  provider: azure
  secretObjects:
  - secretName: test-secret1
    data:
    - key: dbUserName
      objectName: dbUserName
    type: Opaque
  - secretName: test-secret2
    data:
    - key: dbPassword
      objectName: dbPassword
    type: Opaque
  parameters:
    usePodIdentity: "true"                                      
    useVMManagedIdentity: "false"                               
    userAssignedIdentityID: ""                                                                                                
    keyvaultName: "myk8skv29"                                                                
    cloudName: ""                                               
    objects:  |
      array:
        - |
          objectName: dbUserName                              
          objectType: secret                                    
          objectVersion: ""
        - | 
		  objectName: dbPassword                              
          objectType: secret                                    
          objectVersion: ""
		  
    resourceGroup: "aks2akvrg"                      
    subscriptionId: "5558daf3-543e-4f1e-883d-49a9ef02f618"      
    tenantId: "8ce07706-2f54-46a7-868e-fdea1905827d" 


kubectl apply -f provider.yaml

vi pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: inject-secrets-from-akv
  labels:
    aadpodidbinding: azure-pod-identity-binding-selector
  
spec:
  containers:
    - name: nginx
      image: nginx
      env:
      - name: SECRET1
        valueFrom:
          secretKeyRef:
            name: test-secret1
            key: dbUserName
	  - name: SECRET2
        valueFrom:
          secretKeyRef:
	        name: test-secret2
            key: dbPassword		
      volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: spc-myk8skv29	
		  
kubectl apply -f pod.yaml 

shreya@Azure:~$ kubectl get pods
NAME                                                READY   STATUS    RESTARTS   AGE
aad-pod-identity-mic-77b8567cc4-knkg9               1/1     Running   0          38m
aad-pod-identity-mic-77b8567cc4-r98ld               1/1     Running   0          38m
aad-pod-identity-nmi-m7btj                          1/1     Running   0          38m
csi-secrets-store-provider-azure-1640805259-q2p2x   1/1     Running   0          39m
inject-secrets-from-akv                             1/1     Running   0          2m22s
secrets-store-csi-driver-wzs8k                      3/3     Running   0          39m
shreya@Azure:~$ kubectl get secrets
NAME                                                                TYPE                                  DATA   AGE
aad-pod-identity-mic-token-8fwp9                                    kubernetes.io/service-account-token   3      38m
aad-pod-identity-nmi-token-fpbx9                                    kubernetes.io/service-account-token   3      38m
csi-secrets-store-provider-azure-token-6zjwc                        kubernetes.io/service-account-token   3      40m
default-token-q7zjx                                                 kubernetes.io/service-account-token   3      44m
secrets-store-csi-driver-token-s22wh                                kubernetes.io/service-account-token   3      40m
sh.helm.release.v1.csi-secrets-store-provider-azure-1640805259.v1   helm.sh/release.v1                    1      40m
sh.helm.release.v1.pod-identity.v1                                  helm.sh/release.v1                    1      38m
test-secret1                                                        Opaque                                1      109s
test-secret2                                                        Opaque                                1      109s
shreya@Azure:~$ kubectl exec -it inject-secrets-from-akv -- ls /mnt/secrets-store/
dbPassword  dbUserName
shreya@Azure:~$ kubectl exec -it inject-secrets-from-akv -- cat /mnt/secrets-store/dbUserName
postgresql29decshreya@Azure:~$ kubectl exec -it inject-secrets-from-akv -- cat /mnt/secrets-store/dbPassword
Uldebasish@2003shreya@Azure:~$

#Show secrets
kubectl exec -it inject-secrets-from-akv ls /mnt/secrets-store/

kubectl exec -it inject-secrets-from-akv -- ls /mnt/secrets-store/

# Verify secrets mentioned in provider class
kubectl exec -it inject-secrets-from-akv -- cat /mnt/secrets-store/dbUserName
kubectl exec -it inject-secrets-from-akv -- cat /mnt/secrets-store/dbPassword



pos-deployment.yaml		  
			  
			  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
		aadpodidbinding: azure-pod-identity-binding-selector
    spec:
      containers:
        - name: postgres
          image: postgres:10.4
          imagePullPolicy: "IfNotPresent"
          ports:
            - containerPort: 5432
          env:
           - name: POSTGRES_USER
             valueFrom:
               secretKeyRef:
                 name: test-secret1
                 key: dbUserName
           - name: POSTGRES_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: test-secret2
                 key: dbPassword
          volumeMounts:
            - mountPath: "/mnt/secrets-store"
              name: postgredb
			  readOnly: true
      volumes:
        - name: postgredb
          csi:
		    driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: spc-myk8skv29
			  
			  
kubectl get pods

shreya@Azure:~$ kubectl get pods
NAME                                                READY   STATUS    RESTARTS   AGE
aad-pod-identity-mic-77b8567cc4-6h56s               1/1     Running   0          5m18s
aad-pod-identity-mic-77b8567cc4-fr4h2               1/1     Running   0          5m18s
aad-pod-identity-nmi-qkl7q                          1/1     Running   0          5m18s
csi-secrets-store-provider-azure-1640846526-msx6t   1/1     Running   0          11m
postgres-77685b9c9f-vdh7l                           1/1     Running   0          67s
secrets-store-csi-driver-x2sm6                      3/3     Running   0          11m


connect to postgrsql db:

shreya@Azure:~$ kubectl exec -it postgres-77685b9c9f-vdh7l -- /bin/bash

root@postgres-77685b9c9f-vdh7l:/# psql -U postgresql29dec -W postgres
Password for user postgresql29dec:Uldebasish@2003
psql (10.4 (Debian 10.4-2.pgdg90+1))
Type "help" for help.

postgres=#

\dt 
select * from customer;

postgres=# select * from customer;
 id | country_of_birth | country_of_residence |   name   | segment
----+------------------+----------------------+----------+---------
  1 | india            | india                | debasish | retail
(1 row)


to connect to db application we need:

db username, pwd, port no, hostname or ip or end point.

postgres-service.yaml


apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  type: ClusterIP
  ports:
   - port: 5432
  selector:
   app: postgres
   
   
kubectl apply -f postgres-service.yaml


shreya@Azure:~$ kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.0.0.1      <none>        443/TCP    29m
postgres     ClusterIP   10.0.47.191   <none>        5432/TCP   14s

-------------Manual--leave it and jump to next step----------

shreya@Azure:~/kubernetes/spring-boot-postgresql/src/main/resources$ export POSTGRES_HOST=10.0.47.191
shreya@Azure:~/kubernetes/spring-boot-postgresql/src/main/resources$ export POSTGRES_USER=postgresql29dec
shreya@Azure:~/kubernetes/spring-boot-postgresql/src/main/resources$ export POSTGRES_PASSWORD=Uldebasish@2003


https://rtfm.co.ua/en/kubernetes-clusterip-vs-nodeport-vs-loadbalancer-services-and-ingress-an-overview-with-examples/


curl --header "Content-Type: application/json" \
  --request POST \
  --data '{"username":"xyz","password":"xyz"}' \
  http://10.0.47.191:33333/createnewcustomer
  
  
curl --header "Content-Type: application/json" \
    --request POST \
    --data '{"name": "debasish", "country_of_birth": "india","country_of_residence": "india","segment": "retail"}' \
    http://10.0.47.191:33333/createnewcustomer

------------------

craete dockerfile


FROM java:8
COPY target/springboot-postgres-docker-assignment-1.0-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]

docker image build -t debago/springboot-postgres-docker-assignment:dec31 .

git token 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

docker images push debago/springboot-postgres-docker-assignment:dec31

springboot-deployment.yaml 

apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-postgres-sample
  namespace: default
spec:
  selector:
    matchLabels:
      app: spring-boot-postgres-sample
  replicas: 1
  template:
    metadata:
      name: spring-boot-postgres-sample
      labels:
        app: spring-boot-postgres-sample
    spec:
      containers:
      - name: spring-boot-postgres-sample
        env:
          - name: POSTGRES_USER
            valueFrom:
              secretKeyRef:
                name: test-secret1
                key: dbUserName
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef:
                name: test-secret2
                key: dbPassword
          - name: POSTGRES_HOST
            valueFrom:
              configMapKeyRef:
                name: hostname-config
                key: postgres_host
        image: debago/springboot-postgres-docker-assignment:dec31
		

Create a config map with the hostname of Postgres:

kubectl create configmap hostname-config --from-literal=postgres_host=$(kubectl get svc postgres -o jsonpath="{.spec.clusterIP}")		

kubectl get cm
kubectl describe cm hostname-config

install azure cli:
----------------------

sudo apt-get update
sudo apt-get install ca-certificates curl apt-transport-https lsb-release gnupg -y 

curl -sL https://packages.microsoft.com/keys/microsoft.asc |
    gpg --dearmor |
    sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null
	
	
AZ_REPO=$(lsb_release -cs)
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" |
    sudo tee /etc/apt/sources.list.d/azure-cli.list
	
sudo apt-get update
sudo apt-get install azure-cli -y 


install kubectl:
----------------------

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo cp kubectl /usr/local/bin
export PATH=$PATH:/usr/local/bin
kubectl version

az aks get-credentials --resource-group aks2akvrg --name myk8s

kubectl apply -f springboot-deployment.yaml

azure@ubuntu-vm:~/springboot-postgresql-docker-assignment$ kubectl get pods
NAME                                                READY   STATUS    RESTARTS   AGE
aad-pod-identity-mic-77b8567cc4-6h56s               1/1     Running   0          29h
aad-pod-identity-mic-77b8567cc4-fr4h2               1/1     Running   0          29h
aad-pod-identity-nmi-qkl7q                          1/1     Running   0          29h
csi-secrets-store-provider-azure-1640846526-msx6t   1/1     Running   0          29h
postgres-77685b9c9f-vdh7l                           1/1     Running   0          28h
secrets-store-csi-driver-x2sm6                      3/3     Running   0          29h
spring-boot-postgres-sample-6d9bbd6c9c-qfgvr        1/1     Running   0          38s


springboot-service.yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    app: spring-boot-postgres-sample
  name: spring-boot-postgres-sample
spec:
  ports:
    - name: spring-boot-postgres-sample
      port: 33333
      protocol: TCP
  selector:
    app: spring-boot-postgres-sample
  type: LoadBalancer
  
  
kubectl apply -f springboot-service.yaml

azure@ubuntu-vm:~/springboot-postgresql-docker-assignment$ kubectl get svc
NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)           AGE
kubernetes                    ClusterIP      10.0.0.1       <none>          443/TCP           29h
postgres                      ClusterIP      10.0.47.191    <none>          5432/TCP          28h
spring-boot-postgres-sample   LoadBalancer   10.0.158.174   20.81.105.245   33333:30007/TCP   5m21s



http://server-ip:30007/createnewcustomer
http://20.127.132.72:33333/createnewcustomer

http://20.127.132.72:30007/createnewcustomer

curl --header "Content-Type: application/json" \
    --request POST \
    --data '{"name": "debasish", "country_of_birth": "india","country_of_residence": "india","segment": "retail"}' \
    http://20.81.105.245:33333/createnewcustomer
	
	
http://20.81.105.245:33333/listallcustomers

20.81.105.245..loadbalancer



for loadbalancer :

http:// external ip:application port

for node port :

http:// serverip:exposed port

kubectl apply -f springboot-service.yaml

azure@ubuntu-vm:~/springboot-postgresql-docker-assignment$ kubectl get svc
NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)           AGE
kubernetes                    ClusterIP      10.0.0.1       <none>          443/TCP           29h
postgres                      ClusterIP      10.0.47.191    <none>          5432/TCP          28h
spring-boot-postgres-sample   LoadBalancer   10.0.158.174   20.81.105.245   33333:30007/TCP   5m21s

http://20.81.105.245:33333/createnewcustomer
http://20.81.105.245:33333/listallcustomers

in above case external ip for type: loadbalancer and port:application port not exposed port

ClusterIP: the default type, will create a Service resource with an IP address from the cluster’s pool, such a Service will be available from within the cluster only (or with kube-proxy)
NodePort: will open a TCP port on each WorkerNode EС2, “behind it” automatically will create a ClusterIP Service and will route traffic from this TCP port on an ЕС2 to this ClusterIP – such a service will be accessible from the world (obviously, if an EC2 has a public IP), or within a VPC
LoadBalancer: will create an   external Load Balancer (AWS Classic LB), “behind it” automatically will create a NodePort, then ClusterIP and in this way will route traffic from the Load Balancer to a pod in a cluster.

