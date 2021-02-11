# Deploy sample application to IKS cluster

Setup (lab1)
Instructions to get the cluster name.

Cluster name:
ibmcloud ks cluster get -c $MYCLUSTER |  grep "Name:"| head -1 | awk '{print $2}'
ibmcloud ks cluster get -c $MYCLUSTER --output json | jq '.name'

export MYCLUSTER=`ibmcloud ks cluster get -c $MYCLUSTER --output json | jq -r '.name'`


Change all the script permissions
chmod +x k8s/*.sh

Ingress domain:
ibmcloud ks cluster get -c $MYCLUSTER |  grep "Ingress Subdomain" | awk '{print $3}'
ibmcloud ks cluster get -c $MYCLUSTER --output json | jq -r '.ingressHostname'

export INGRESS_SUBDOMAIN=`ibmcloud ks cluster get -c $MYCLUSTER --output json | jq -r '.ingressHostname'`

Replace Ingress vale
sed -i "s/<INGRESS_SUBDOMAIN>/${INGRESS_SUBDOMAIN}/" k8s/ingress.yaml

Mac:
sed -i "" "s/<INGRESS_SUBDOMAIN>/${INGRESS_SUBDOMAIN}/" k8s/ingress.yaml

echo "https://petclinic.${INGRESS_SUBDOMAIN}"


## Step 1 - Deploy petclinic application

In the `IBM Cloud Shell`, deploy the sample petclinic application.

1. Prepare the deployment scripts by changing their execution permission to run.
    ```
    chmod +x k8s/*.sh
    ```

1. Deploy four microservices of the sample petclinic application by running the command below: 
    ```
    k8s/deploy-petclinic.sh
    ```
    ```
    $ k8s/deploy-petclinic.sh
    deployment.apps/api-gateway created
    service/api-gateway created
    deployment.apps/customers created
    service/customers-service created
    deployment.apps/vets created
    deployment.apps/visits created
    ```
    This deploys `Deployment` and `Service` resources for each microservice component.

1. Verify the deployment resources.

    ```
    kubectl get pods

    NAME                           READY   STATUS    RESTARTS   AGE
    api-gateway-575f59b7d8-hwnct   1/1     Running   0          32s
    customers-687749cfb-cjcpc      1/1     Running   0          32s
    vets-6bb6655b7f-8x2tg          1/1     Running   0          31s
    visits-784749c647-l8btx        1/1     Running   0          30s
    ```

1. Verify the service resources.

    ```
    kubectl get svc

    NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
    api-gateway         NodePort    172.21.212.167   <none>        80:32002/TCP   88s
    customers-service   NodePort    172.21.145.118   <none>        80:32003/TCP   88s
    kubernetes          ClusterIP   172.21.0.1       <none>        443/TCP        2d22h
    vets-service        NodePort    172.21.17.247    <none>        80:32005/TCP   87s
    visits-service      NodePort    172.21.165.91    <none>        80:32004/TCP   86s
    ```

## Step 2 - Deploy Ingress resource

When deployed the sample application to a non-Lite tier IKS cluster, it's possible to expose the application with an external hostname.

1. Retrieve `Ingress Subdomain`.

    ```
    ibmcloud ks cluster get -c $MYCLUSTER

    Retrieving cluster mycluster...
    OK
                                
    Name:                           mycluster
    ID:                             xxxx
    State:                          normal
    Created:                        2019-03-20T06:13:44+0000
    Location:                       seo01
    Master URL:                     https://xxx.xxx.xxx:xxxx
    Public Service Endpoint URL:    https://xxx.xxx.xxx:xxxx
    Private Service Endpoint URL:   -
    Master Location:                seo01
    Master Status:                  Ready (1 day ago)
    Master State:                   deployed
    Master Health:                  normal
    Ingress Subdomain:              mycluster.seo01.containers.appdomain.cloud
    Ingress Secret:                 mycluster
    Workers:                        1
    Worker Zones:                   seo01
    Version:                        1.12.9_1555* (1.13.6_1524 latest)
    Owner:                          xxxx@xxxx.xxx
    Monitoring Dashboard:           -
    Resource Group ID:              xxxxxxxxxxx
    Resource Group Name:            default
    ```

1. Store the `Ingress Subdomain`.

  ```
  export INGRESS_SUBDOMAIN=<Ingress Subdomain>
  echo $INGRESS_SUBDOMAIN
  ```

1. Modfyi `k8s/ingress.yaml` file.

    * Open `k8s/ingress.yaml` file in an editor.

    * Locate the section,

        ```
        spec:
          rules:
          - host: petclinic.<INGRESS_SUBDOMAIN>   
        ```

    * Replace `<INGRESS_SUBDOMAIN>` with the value that you retrieved in the previous step.

    * Save the changes.

1. Deploy Ingress resourcve

  ```
  kubectl create -f k8s/ingress.yaml
  ```

## Step 3 - Verify petclinic application

If everything goes as planned, the `petclinic` application can be accessed at `https://petclinic.<INGRESS_SUBDOMAIN>`. By default, an internal database is used to stored data.


## Step 4 - Deploy MYSQL database to the IKS cluster (Optionally)

Instead of running the `petclinic` application on an internal database, you may choose to deploy an instance of `MYSQL` database on the same IKS cluster.


### Step 4.1 - Prepare Persisent Volume

There are various persistent storage options to store the data of `MYSQL` DATABASE, local storage and cloud storage and etc. For the simplicity, the local file system system on the Node server is used for this repo. Execute the command below to create a 5Gi local-volume.

  ```
  kubectl create -f k8s/mysql/local-volumes.yaml
  ```

### Step 4.2 - Create a secret storing MYSQL credential

User and password of `MYSQL` database is stored in a `secret` resource for security reason.

  ```
  kubectl create -f k8s/mysql/mysql-secret.yaml
  ```

### Step 4.3 - Deploy MYSQL database

One deployment, one service and one persistent-volume-claim resources are created when deploying `MYSQL` database.

  ```
  kubectl create -f k8s/mysql/mysql.yaml
  ```

### Step 4.4 - Populate MYSQL database

To populate MYSQL database running on the cluster,

1. Retrieve the pod information where MYSQL database is running.

  ```
  kubectl get pod -l app=mysql

  NAME                     READY   STATUS    RESTARTS   AGE
  mysql-6d87765586-2q7sn   1/1     Running   0          19h
  ```

1. Store the pod name of MYSQL. 

  ```
  export MYSQL_POD=<MYSQL POD NAME>
  ```

1. Copy SQL files to the pod.

  ```
  kubectl cp k8s/mysql/sql/mysql-schema.sql $MYSQL_POD:/tmp/
  kubectl cp k8s/mysql/sql/mysql-data.sql $MYSQL_POD:/tmp/
  ```

1. Populate MYSQL database

  ```
  kubectl exec $MYSQL_POD -- sh -c 'mysql -uroot -ppetclinic petclinic < /tmp/mysql-schema.sql'
  kubectl exec $MYSQL_POD -- sh -c 'mysql -uroot -ppetclinic petclinic < /tmp/mysql-data.sql'
  ```

1. Retrieve data from MYSQL database for verification.

  ```
  kubectl exec $MYSQL_POD -- sh -c 'mysql -u root -ppetclinic -e "select * from vets" petclinic'

  mysql: [Warning] Using a password on the command line interface can be insecure.
  id	first_name	last_name
  1 James	Carter
  2	Helen	Leary
  3	Linda	Douglas
  4	Rafael	Ortega
  5	Henry	Stevens
  6	Sharon	Jenkins
  ```

## Step 5 - Run sample application on MYSQL database 

`MYSQL` database has been successfully deployed in the same IKS cluster. Now, you are goint to run the sample `petclinic` application on MYSQL database instead of the internal database.

### Step 5.1 - Store database connection information in configMap

To store MYSQL database connection information in `configMap` resource,

  ```
  kubectl create -f k8s/mysql/mysql-configmap.yaml
  ```

### Step 5.2 - Modify sample application deployment to run on MySQL DB

  ```
  kubectl apply -f k8s/mysql/mysql-customers-service.yaml
  kubectl apply -f k8s/mysql/mysql-vets-service.yaml
  kubectl apply -f k8s/mysql/mysql-visits-service.yaml
  ```


## Step 6 - Verify petclinic application

1. Retrieve `Ingress Subdomain`.

    ```
    echo https://petclinic.$INGRESS_SUBDOMAIN

    https://petclinic.leez-iks03-2bef1f4b4097001da9502000c44fc2b2-0000.us-south.containers.appdomain.cloud
    ```

1. Access the `petclinic` application via `https://petclinic.$INGRESS_SUBDOMAIN`. Now, it's running on `MYSQL` database instead of the internal database.



