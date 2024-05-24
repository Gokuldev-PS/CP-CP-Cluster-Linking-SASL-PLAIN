# CP-CP-Cluster-Linking-SASL-PLAIN

### Kafka Cluster with Basic Authentication
In this example, both source and destination kafka are run in SASL_PLAIN mode

## Set up Pre-requisites

export TUTORIAL_HOME=<Tutorial directory>/CP-CP-Cluster-Linking-SASL-PLAIN

Create two namespaces, one for the source cluster components and one for the destination cluster components.
Note:: in this example, only deploy zookeeper and kafka for source and zookeeper, kafka and connect for destination

```
kubectl create ns source
kubectl create ns destination
```

Deploy Confluent for Kubernetes (CFK) in cluster mode, so that the one CFK instance can manage Confluent deployments in multiple namespaces. Here, CFk is deployed to the `default` namespace.

```
helm upgrade --install confluent-operator \
  confluentinc/confluent-for-kubernetes \
  --namespace default --set namespaced=false
```
 NOTE : If you get an error  ClusterRole already exist delete the cluster role and the role bindings using the kubectl command for the confluent operator
 
 ```
 kubectl delete role NAME_OF_ROLE -n NAMESPACE
 kubectl delete rolebinding NAME_OF_ROLEBINDING -n NAMESPACE

```
### create required secrets
```
kubectl -n source create secret generic credential \
    --from-file=plain-users.json=$TUTORIAL_HOME/creds-kafka-sasl-users.json \
    --from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt \
    --from-file=basic.txt=$TUTORIAL_HOME/creds-basic-users.txt

kubectl -n destination create secret generic credential \
    --from-file=plain-users.json=$TUTORIAL_HOME/creds-kafka-sasl-users.json \
    --from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt \
    --from-file=basic.txt=$TUTORIAL_HOME/creds-basic-users.txt
```

### Source Cluster Deployment


```
kubectl -n source create secret generic rest-credential \
    --from-file=basic.txt=$TUTORIAL_HOME/rest-credential.txt
```
#### deploy source zookeeper, kafka cluster and topic `demo` in namespace `source`
```
kubectl apply -f $TUTORIAL_HOME/zk-kafka-source.yaml
```

### Destination Cluster Deployment

```
kubectl -n destination create secret generic rest-credential \
    --from-file=basic.txt=$TUTORIAL_HOME/rest-credential.txt
    
kubectl -n destination create secret generic password-encoder-secret \
    --from-file=password-encoder.txt=$TUTORIAL_HOME/password-encoder-secret.txt
```

#### deploy destination zookeeper and kafka cluster in namespace `destination`

    kubectl apply -f $TUTORIAL_HOME/zk-kafka-destination.yaml

After the Kafka cluster is in running state, create cluster link between source and destination. Cluster link will be created in the destination cluster

#### create clusterlink between source and destination
    kubectl apply -f $TUTORIAL_HOME/clusterlink-sasl-plaintext.yaml
### Run test

#### exec into source kafka pod
    kubectl -n source exec kafka-0 -it -- bash

#### create kafka.properties

    cat <<EOF > /tmp/kafka.properties
    bootstrap.servers=kafka.source.svc.cluster.local:9071
    sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=kafka password=kafka-secret;
    sasl.mechanism=PLAIN
    security.protocol=SASL_PLAINTEXT
    
    EOF

#### produce in source kafka cluster

    seq 100 | kafka-console-producer --topic demo --broker-list kafka.source.svc.cluster.local:9071 --producer.config /tmp/kafka.properties
#### open a new terminal and exec into destination kafka pod
    kubectl -n destination exec kafka-0 -it -- bash
#### create kafka.properties for destination kafka cluster
    cat <<EOF > /tmp/kafka.properties
    bootstrap.servers=kafka.destination.svc.cluster.local:9071
    sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=kafka password=kafka-secret;
    sasl.mechanism=PLAIN
    security.protocol=SASL_PLAINTEXT
    
    EOF
#### validate topic is created in destination kafka cluster
    kafka-topics --describe --topic demo --bootstrap-server kafka.destination.svc.cluster.local:9071 --command-config /tmp/kafka.properties

#### consume in destination kafka cluster and confirm message delivery in destination cluster

    kafka-console-consumer --from-beginning --topic demo --bootstrap-server  kafka.destination.svc.cluster.local:9071  --consumer.config /tmp/kafka.properties
