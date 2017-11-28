# Microservices  Reference Implementation
Microsoft patterns & practices

https://docs.microsoft.com/azure/architecture/microservices

---

## Provisioning

this section is meant to provide guidance on how to get provisioned Drone Delivery Reference Implementation
into your ACS Kubernetes cluster.

The Drone Delivery Reference Implementation is composed by 7 microservices:

* *account*
* *delivery*
* *scheduler*
* *ingestion*
* *dronescheduler*
* *package*
* *thirdparty*

### Prerequisites

a. install [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

b. install kubeclt
   ```bash
    sudo az acs kubernetes install-cli --client-version=1.7.7
   ```
   *Important*: Please note that you might need to use a higher client version depending on the cluster version

c. set the following environment variables:
   ```bash
    export LOCATION=your_location_here && \
    export RESOURCE_GROUP=your_resource_group_here && \
    export CLUSTER_NAME=your_cluster_name_here && \
    export RESOURCE_GROUP_STO=your_resource_group_4storage_here && \
    export DS_STO_NAME=your_deliveries_storage_name_here
   ```
   Note: the creation of your cluster might take some time

d. login in your azure subscription:
   ```bash
   az login
   ```

e. create the resource group:
   ```bash
   az group create --name=$RESOURCE_GROUP --location=$LOCATION && \
   az group create --name=$RESOURCE_GROUP_STO --location=$LOCATION
   ```

f. create your ACS Cluster:
   ```bash
   az acs create --orchestrator-type=kubernetes \
                 --resource-group $RESOURCE_GROUP \
                 --name=$CLUSTER_NAME \
                 --generate-ssh-keys
   ```

g. check your ACS cluster has been created:
   ```bash
   az acs show -g $RESOURCE_GROUP -n $CLUSTER_NAME
   ```

h. download the credentails:
   ```bash
   az acs kubernetes get-credentials --resource-group=$RESOURCE_GROUP --name=$CLUSTER_NAME
   ```

i. test the cluster is up and running:
   ```bash
   kubectl get nodes
   ```

j. Deploy the following Azure Resources:

   1. delivery microservice:
   ```bash
   #! bin/bash
   export redis_name="${DS_STO_NAME}-redis"
   export cosmosdb_name="${DS_STO_NAME}-cosmosdb"
   export database_name="${cosmosdb_name}-db"
   export collection_name="${database_name}-col"
    
   #Create Delivery Service Azure Redis 
   az redis create --location $LOCATION \
                --name $redis_name \
                --resource-group $RESOURCE_GROUP_STO \
                --sku Premium \
                --vm-size P4

   #Create Delivery Service DocumentDB API Cosmos DB account
   az cosmosdb create \
       --name $cosmosdb_name \
       --kind GlobalDocumentDB \
       --resource-group $RESOURCE_GROUP_STO \
       --max-interval 10 \
       --max-staleness-prefix 200 
   
   # Create a database 
   az cosmosdb database create \
       --name $cosmosdb_name \
       --db-name $database_name \
       --resource-group $RESOURCE_GROUP_STO
   
   # Create a collection
   az cosmosdb collection create \
       --collection-name $collection_name \
       --name $cosmosdb_name \
       --db-name $database_name \
       --resource-group $RESOURCE_GROUP_STO
   ```

   2. package microservice: [TBD] azure templates or az steps 
   3. ingestion/scheduler: [TBD] azure templates or az steps

k. build docker image for Delivery Service
   ```bash
   docker build -t your_repo/fabrikam.dronedelivery.deliveryservice:0.1.0 ./microservices-reference-implementation/src/bc-shipping/delivery/Fabrikam.DroneDelivery.DeliveryService/. && \
   sed -i "s#image:#image: your_repo/fabrikam.dronedelivery.deliveryservice:0.1.0#g" ./microservices-reference-implementation/k8s/delivery.yaml
   ```

l. (Optional) add linkerd to your cluster:
   ```bash
   wget https://raw.githubusercontent.com/linkerd/linkerd-examples/master/k8s-daemonset/k8s/linkerd.yml && \
   sed -i "s/default/bc-shipping/g" linkerd.yml && \
   kubectl apply -f linkerd.yml
   ```

### Deploying the application 

1. clone the repository
   ```bash
   git clone https://github.com/mspnp/microservices-reference-implementation.git
   ```

2. Create the bc-shipping namespace:
   ```bash
   kubeclt create namespace bc-shipping
   ```

3. Create the required secrets 
   ```bash
   kubectl --namespace bc-shipping create --save-config=true secret generic delivery-storageconf | \
                  --from-literal=CosmosDB_Key=$(az cosmosdb list-keys --name $cosmosdb_name --resource-group $RESOURCE_GROUP_STO --query "primaryMasterKey") | \
                  --from-literal=CosmosDB_Endpoint=$(az cosmosdb show --name $cosmosdb_name --resource-group $RESOURCE_GROUP_STO --query documentEndpoint) | \
                  --from-literal=Redis_HostName=$(az cosmosdb show --name $cosmosdb_name --resource-group $RESOURCE_GROUP_STO --query documentEndpoint) | \
                  --from-literal=Redis_PrimaryKey=$(az redis list-keys --name $redis_name --resource-group $RESOURCE_GROUP_STO --query primaryKey) | \
                  --from-literal=EH_ConnectionString= | \
                  --from-literal=Redis_SecondaryKey=
   kubectl create secret generic package-secrets --from-literal=mongodb-pwd=your_mongodb_connection_string
   ```

4. Add values to database and collection env variables in the k8s delivery configuration file:

   ```bash
   sed -i "s/value: \"CosmosDB_DatabaseId\"/value: your_database_name_here/g" "./microservices-reference-implementation/k8s/delivery.yaml" && \
   sed -i "s/value: \"CosmosDB_CollectionId\"/value: your_collection_here/g"  "./microservices-reference-implementation/k8s/delivery.yaml" && \
   sed -i "s/value: \"EH_EntityPath\"/value:/g"                               "./microservices-reference-implementation/k8s/delivery.yaml"
   ```

5. Deploy the Drone Delivery microservices in your cluster:
   ```bash
   kubectl --namespace bc-shipping apply -f ./microservices-reference-implementation/k8s/
   ```

6. Check Drone Delivery is up and running:
   ```bash
   kubectl get all -l bc=shipping
   ```

### Testing Drone Delivery in your ACS Kubernetes cluster

  [TBD]

---

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.