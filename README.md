# AMQ Streams Multicluster Reference deployment

## Architecture

The architecture deployed by this template is shown in this diagram:

![amqstreams-dr-reference-architecture](media/amqstreams-dr-reference-architecture.png)

In this diagram two sites/data-centers are shown with a strimzi cluster operator and kafka cluster running within each site. These sites represent Production and Disaster Recovery(DR) environments. Kafka clients  produce and consume messages using an URL which is backed with Global Traffic Manager(GTM) a.k.a global load balancer. GTM will direct traffic to either Prod/DR clusters. An assumption here is, intelligence can be incorporated in GTM where it will redirect traffic to DR cluster only if Production kafka cluster healthcheck fails for configurable amount of time (say X seconds). When Production cluster becomes available, fail-back to Production needs to be planned and done during maintenance windows when clients are neither producing or consuming messages.  This is to minimize the message loss due to one-way MirrorMaker mirroring limitation.

Strimzi Cluster operator can be configured to watch the creation kafka crds i.e. Kafka, KafkaTopic, KafkaUser, KafkaMirrorMaker etc. objects in certain namespaces and create kafka and zookeeper clusters.      

MirrorMaker 1.0 which does one-way replication i.e. from Production to DR will run in DR cluster. It will consume messages from Production Cluster and produce to DR cluster. This is based on eventual consistency model. With this model, there is potential for messages produced and committed in Production cluster for not being available in DR when failed over to DR cluster.
 
*Note*: Since it is a TEST environment and lack of DNS server and GTM infrastructure availability, instructions outlined below does not deploy DNS URL and GTM. Clients will directly connect to kafka's bootstrap URL.

## Installation

Instructions below deploys two operators and two kafka and zookeper clusters in different namespaces simulating multiple clusters/environments. E.g. amq-operator-a, datacenter-a namespaces represent Production Cluster. Whereas, amq-operator-b, datacenter-b namespaces represent DR Cluster.

Get a [pull secret](https://access.redhat.com/terms-based-registry/#/accounts) for the registry.redhat.con and store it in a file called `pull-secret.base64`

In order to deploy execute the following steps

1. Set some environment variables:

    ```shell
    export PULL_SECRET=$(cat ./pull-secret.base64)
    ```
    
2. Clone this repository to your local computer and login to the OpenShift Cluster using a user who has ClusterAdmin access. ClusterAdmin is needed to install and create Kafka Custom Resource Definitions(CRDs).  

3. Deploy Kafka CRDs:

    ```shell
	oc apply -f crd/
    ```

4. Create ClusterRoles that are eventually assigned to Strimzi Operator ServiceAccounts

    ```shell
	oc apply -f clusterrole/
    ```
    
5. Create projects and deploy strimzi operator, kafka clusters. AMQ Operator deployed in amq-operator-a watches for Kafka CRDs in datacenter-a. Kafka cluster is created with own cluster certificates.

    ```shell
    oc new-project amq-operator-a
	oc new-project datacenter-a
	oc process -f templates/operator_template.yaml -p NAMESPACES_TO_WATCH=datacenter-a -p PULL_SECRET=$PULL_SECRET -p SERVICE_ACCOUNT_NAME=strimzi-cluster-operator| oc apply -f - -n amq-operator-a
	oc process -f templates/cluster_template.yaml -p OPERATOR_SERVICE_ACCOUNT_NAME=strimzi-cluster-operator -p AMQ_OPERATOR_NAMESPACE=amq-operator-a -p KAFKA_CLUSTER=my-cluster -p KAFKA_BROKER_SIZE=3 -p ZK_SIZE=3 | oc apply -f - -n datacenter-a
    ```
    
6. AMQ Operator in above step is created in *paused* state. So, resume the deployment
   
       ```shell
    oc rollout resume  deployment strimzi-cluster-operator -n amq-operator-a
    ```

7. Repeat above two steps where AMQ Operator deployed in amq-operator-b watches for Kafka CRDs in datacenter-b.

    ```shell
    oc new-project amq-operator-b
    oc new-project datacenter-b
    oc process -f templates/operator_template.yaml -p NAMESPACES_TO_WATCH=datacenter-b -p PULL_SECRET=$PULL_SECRET -p SERVICE_ACCOUNT_NAME=strimzi-cluster-operator| oc apply -f - -n amq-operator-b
    oc process -f templates/cluster_template.yaml -p OPERATOR_SERVICE_ACCOUNT_NAME=strimzi-cluster-operator -p AMQ_OPERATOR_NAMESPACE=amq-operator-b -p KAFKA_CLUSTER=my-second-cluster -p KAFKA_BROKER_SIZE=3 -p ZK_SIZE=3 | oc apply -f - -n datacenter-b
    oc rollout resume  deployment strimzi-cluster-operator -n amq-operator-b 
  ```

