# Generate application log entries

For simplicity, script will be executed in `api-gateway` pod to send REST requests to all 4 service components of the petclinic application. Log entries will be generated for each service component.

## Step 1 - Send requests to the sample application

To generate application log entries,

1. Go back to `IBM Cloud Shell` terminal.

1. Retrieve the Pod information where `api-gateway` component is running.

    ```
    kubectl get pods -l tier=api-gateway

    NAME                           READY   STATUS    RESTARTS   AGE
    api-gateway-575f59b7d8-75wbm   1/1     Running   0          81m
    ```
1. Store the Pod name.

    ```
    export API_GATEWAY_POD=<API Gateway Pod Name>
    echo $API_GATEWAY_POD
    ```

1. Get into the `API-Gateway' pod.

    ```
    kubectl exec $API_GATEWAY_POD -ti sh
    ```

1. The prompt change shows that you are in the `API-Gateway` pod now.

1. Copy/paste and Execute the following command in the pod. The script sends 100 requests to each service component.

    ```
    for i in `seq 1 100` ; do wget -q -O - http://customers-service/owners && wget -q -O - http://vets-service/vets && wget -q -O - http://visits-service/pets/visits?petId=7 &&  wget -q -O - http://petclinic.<INGRESS_SUBDOMAIN>/#!/welcome; done
    ```

    For example, 

    ```
    for i in `seq 1 100` ; do wget -q -O - http://customers-service/owners && wget -q -O - http://vets-service/vets && wget -q -O - http://visits-service/pets/visits?petId=7 &&  wget -q -O - http://petclinic.leez-iks03-f0093114134cf555e1a213f3756140db-0000.us-east.containers.appdomain.cloud/#!/welcome; done
    ```

1. Exit the pod.

    ```
    exit
    ```


## Step 2 - Review the log entries in application pods

`kubectl` commands can be used to verify application log entries in each component pod.

1. Review log entries of `api-gateway` component.

    ```
    kubectl logs $API_GATEWAY_POD
    ```

1. Review log entries of `customers` component.

    * Retrieve `Pod` information of `customers` component.

        ```
        kubectl get pods -l tier=customers

        NAME                        READY   STATUS    RESTARTS   AGE
        customers-98d7b966c-8wx5x   1/1     Running   0          89m
        ```

    * Retrieve log entries of `customers` component.

      ```
      kubectl logs <Customers Pod Name>
      ```

1. Review log entries of `vets` component.

     * Retrieve `Pod` information of `vets` component.

        ```
        kubectl get pods -l tier=vets

        NAME                        READY   STATUS    RESTARTS   AGE
        customers-98d7b966c-8wx5x   1/1     Running   0          89m
        ```

    * Retrieve log entries of `vets` component.

      ```
      kubectl logs <Vets Pod Name>
      ```

1. Review log entries of `visits` component.

    * Retrieve `Pod` information of `visits` component.

      ```
      kubectl get pods -l tier=visits

      NAME                        READY   STATUS    RESTARTS   AGE
      customers-98d7b966c-8wx5x   1/1     Running   0          89m
      ```

    * Retrieve log entries of `visits` component.

      ```
      kubectl logs <Visits Pod Name>
      ```