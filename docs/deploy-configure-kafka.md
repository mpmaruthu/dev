
# Deploy and Configure Kafka Broker on Hub Cluster
This document provides instructions for deploying and configuring a Kafka broker on the hub cluster. 
It is intended for users who need to set up a Kafka broker (an external Kafka receiver) to forward the 
spoke cluster logs to third-party systems like Splunk, ElasticSearch, or external Kafka receiver.

# Prerequisites
Before you begin, ensure that you have the following prerequisites:
- Access to the hub cluster with sufficient permissions to deploy applications.
- `oc` command-line tool installed and configured or web console enabled to interact with the hub cluster.
- Kafka Operator installed on the hub cluster. You can install it using the OperatorHub in OpenShift.
- How to install the Kafka Operator:
  1. Log in to the OpenShift web console using kube:admin login.
  2. Navigate to the "Operators" section and select "OperatorHub".
  3. Search for "Kafka" and select the "Streams for Apache Kafka".
  4. Make sure latest version is selected from the `stable` channel.
  5. Click on "Install" and follow the prompts to install the operator in the desired namespace (e.g., `openshift-operators`).
- A Kafka custom resource (CR) definition (CRD) available in your hub cluster. This is typically provided by the Kafka Operator.
```
$ oc -n openshift-operators get crds | grep kafka
kafkabridges.kafka.strimzi.io                                             
kafkaconnectors.kafka.strimzi.io                                          
kafkaconnects.kafka.strimzi.io                                            
kafkamirrormaker2s.kafka.strimzi.io                                       
kafkamirrormakers.kafka.strimzi.io                                        
kafkanodepools.kafka.strimzi.io                                           
kafkarebalances.kafka.strimzi.io                                          
kafkas.kafka.strimzi.io                                                   
kafkatopics.kafka.strimzi.io                                              
kafkausers.kafka.strimzi.io                                               
$ 
```
# Deploying Kafka Broker on Hub Cluster
To deploy a Kafka broker, follow these steps:
- Create a `Kafka` custom resource (CR) by enabling additional parameters such as storage, logs retention, 
and tls authentication.
```yaml
kind: Kafka
apiVersion: kafka.strimzi.io/v1beta2
metadata:
  name: kafka-logs-hub4
  namespace: openshift-operators
spec:
  kafka:
    version: 3.9.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
      - name: external
        port: 9094
        tls: true
        authentication:
          type: tls
        type: route
    config:
      offsets.topic.replication.factor: 3
      log.retention.minutes: 1440
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: '3.9'
    storage:
      type: ephemeral
      sizeLimit: 10Gi
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
```
- Save the above YAML content to a file named `kafka-instance.yaml`.
- Run the following command to create the Kafka broker on the hub cluster:
   ```
   oc create -f kafka-instance.yaml
   ```
Note: The above configuration sets up a Kafka broker with the following parameters:

1. Set `log.retention.minutes` to a reasonable value, like 1440 (24 hours), to ensure that logs 
are retained for a sufficient period.

2. Set the `kafka.storage` to a reasonable value, like 10Gi. 

- Create a `Kafka user` custom resource (CR).
```yaml
kind: KafkaUser
apiVersion: kafka.strimzi.io/v1beta2
metadata:
  name: kafka-logs-hub4-user
  labels:
    strimzi.io/cluster: kafka-logs-hub4
  namespace: openshift-operators
spec:
  authentication:
    type: tls
```
- Save the above YAML content to a file named `kafka-user.yaml`.
- Run the following command to create the Kafka user on the hub cluster:
   ```
   oc create -f kafka-user.yaml
   ```
  
Note: The above configuration creates a Kafka user with TLS authentication. 
This kafka user will be used to connect to the Kafka broker securely.
1. When the kafka user is created, it will automatically create a secret. Extract the `user.key` and `user.crt` 
values from the secret.
2. Later, these values will be used to configure the authentication secret on hub cluster under site-specific namespace 
called `ztp-group-mb-du` which requires to on-board DU node policies (Day2 configurations) on the spoke clusters.

- Create a `Kafka topic` custom resource (CR).

```yaml
kind: KafkaTopic
apiVersion: kafka.strimzi.io/v1beta2
metadata:
  name: kafka-logs-hub4-topic
  labels:
    strimzi.io/cluster: kafka-logs-hub4
  namespace: openshift-operators
spec:
  partitions: 10
  replicas: 3
```
- Save the above YAML content to a file named `kafka-topic.yaml`.
- Run the following command to create the Kafka topic on the hub cluster:
   ```
   oc create -f kafka-topic.yaml
   ```
  
Note: The above configuration creates a Kafka topic to which the logs from the spoke clusters will be forwarded.

1. The label `strimzi.io/cluster` connecting the Kafka topic and Kafka user CRs to the created Kafka broker CR.
2. The route CR created automatically with name `kafka-logs-hub4-kafka-bootstrap` for external connectivity. 
3. You can get the route with the command `oc -n openshift-operators get route kafka-logs-hub4-kafka-bootstrap`
4. Now, we have access to the external url to connect to the Kafka broker. 
5. Later, this route will be used to configure the TLS validation on the spoke clusters.
6. By default, there are two listeners named `plain` and `tls` without authentication enabled. 
7. We can use the `listeners` named `external` for connecting with the logs visualizer (provided by Kafka Console Operator) if needed.

# Verify the Kafka Broker Deployment

To verify that the Kafka broker is deployed correctly, you can check the status of the Kafka custom resource and the associated pods, services, and routes. Use the following command:
```
$ oc -n openshift-operators get all -L app.kubernetes.io/instance=kafka-logs-hub4
Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
NAME                                                        READY   STATUS    RESTARTS   AGE    INSTANCE=KAFKA-LOGS-HUB4
pod/amq-streams-cluster-operator-v2.9.1-0-65f779486-s8kxs   1/1     Running   0          3d5h   
pod/kafka-logs-hub4-entity-operator-747449ccf6-722j4        2/2     Running   0          22m    
pod/kafka-logs-hub4-kafka-0                                 1/1     Running   0          22m    
pod/kafka-logs-hub4-kafka-1                                 1/1     Running   0          22m    
pod/kafka-logs-hub4-kafka-2                                 1/1     Running   0          22m    
pod/kafka-logs-hub4-zookeeper-0                             1/1     Running   0          23m    
pod/kafka-logs-hub4-zookeeper-1                             1/1     Running   0          23m    
pod/kafka-logs-hub4-zookeeper-2                             1/1     Running   0          23m    

NAME                                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                        AGE   INSTANCE=KAFKA-LOGS-HUB4
service/kafka-logs-hub4-kafka-0                    ClusterIP   172.30.182.40    <none>        9094/TCP                                       22m   
service/kafka-logs-hub4-kafka-1                    ClusterIP   172.30.160.173   <none>        9094/TCP                                       22m   
service/kafka-logs-hub4-kafka-2                    ClusterIP   172.30.89.120    <none>        9094/TCP                                       22m   
service/kafka-logs-hub4-kafka-bootstrap            ClusterIP   172.30.82.150    <none>        9091/TCP,9092/TCP,9093/TCP                     22m   
service/kafka-logs-hub4-kafka-brokers              ClusterIP   None             <none>        9090/TCP,9091/TCP,8443/TCP,9092/TCP,9093/TCP   22m   
service/kafka-logs-hub4-kafka-external-bootstrap   ClusterIP   172.30.43.199    <none>        9094/TCP                                       22m   
service/kafka-logs-hub4-zookeeper-client           ClusterIP   172.30.230.58    <none>        2181/TCP                                       23m   
service/kafka-logs-hub4-zookeeper-nodes            ClusterIP   None             <none>        2181/TCP,2888/TCP,3888/TCP                     23m   

NAME                                                    READY   UP-TO-DATE   AVAILABLE   AGE    INSTANCE=KAFKA-LOGS-HUB4
deployment.apps/amq-streams-cluster-operator-v2.9.1-0   1/1     1            1           3d5h   
deployment.apps/kafka-logs-hub4-entity-operator         1/1     1            1           22m    

NAME                                                              DESIRED   CURRENT   READY   AGE    INSTANCE=KAFKA-LOGS-HUB4
replicaset.apps/amq-streams-cluster-operator-v2.9.1-0-65f779486   1         1         1       3d5h   
replicaset.apps/kafka-logs-hub4-entity-operator-747449ccf6        1         1         1       22m    

NAME                                                       HOST/PORT                                                                          PATH   SERVICES                                   PORT   TERMINATION   WILDCARD   INSTANCE=KAFKA-LOGS-HUB4
route.route.openshift.io/kafka-logs-hub4-kafka-0           kafka-logs-hub4-kafka-0-openshift-operators.apps.hubmgmt4.slcm3.bos2.lab                  kafka-logs-hub4-kafka-0                    9094   passthrough   None       
route.route.openshift.io/kafka-logs-hub4-kafka-1           kafka-logs-hub4-kafka-1-openshift-operators.apps.hubmgmt4.slcm3.bos2.lab                  kafka-logs-hub4-kafka-1                    9094   passthrough   None       
route.route.openshift.io/kafka-logs-hub4-kafka-2           kafka-logs-hub4-kafka-2-openshift-operators.apps.hubmgmt4.slcm3.bos2.lab                  kafka-logs-hub4-kafka-2                    9094   passthrough   None       
route.route.openshift.io/kafka-logs-hub4-kafka-bootstrap   kafka-logs-hub4-kafka-bootstrap-openshift-operators.apps.hubmgmt4.slcm3.bos2.lab          kafka-logs-hub4-kafka-external-bootstrap   9094   passthrough   None       
$
```
# Configure the Kafka Broker on Hub Cluster
To configure the Kafka broker, you need to create the necessary custom resources for log forwarding and authentication.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-logging-kafka
  namespace: ztp-group-mb-du
data:
  topic: "kafka-logs-hub4-topic" # The topic to which logs will be forwarded.

  url: "kafka-logs-hub4-kafka-bootstrap-openshift-operators.apps.<FQDN>" # The external URL of the Kafka broker.
  caCrt: | # The CA certificate for the Kafka broker.
    -----BEGIN CERTIFICATE-----
    MIIH.............................AQEN
    .....................................
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    MIIF.............................AQEN
    ..................................... 
    -----END CERTIFICATE-----
---
apiVersion: v1
kind: Secret
metadata:
  name: cluster-logging-kafka-auth
  namespace: ztp-group-mb-du
type: Opaque
data:
  # base64 encoded values for userCrt and userKey
  userCrt: "LS0t....="
  userKey: "LS0t....K"
```

- Save the above YAML content to a file named `kafka-config.yaml`.
- Run the following command to create the Kafka configuration on the hub cluster:
   ```
   oc create -f kafka-config.yaml
   ```
  
Note: The above configuration creates a ConfigMap and a Secret for Kafka configuration.

1. The `ConfigMap` contains the topic name, the external URL of the Kafka broker, and the CA certificate.

`caCrt`  (bundle) can be extracted from the external kafka broker url. 

Replace `<FQDN>` with the actual fully qualified domain name of your Kafka broker.

```
$ echo | openssl s_client -showcerts -connect 
kafka-logs-hub4-kafka-bootstrap-openshift-operators.apps.<FQDN>:443 2>/dev/null | 
awk '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/'
```

2. The `Secret` contains the base64 encoded values for the user certificate (`userCrt`) and user key (`userKey`)

`userCrt`and `userKey`are created automatically when creating KakaUser.

```
$ oc -n openshift-operators get secret kafka-logs-hub4-user -o yaml | grep -iE 'user.crt|user.key'
    user.crt: LS0tLS1...
    user.key: LS0tLS1...
```

# Verify the Kafka Broker Configuration
To verify that the Kafka broker is configured correctly, you can check the status of the ConfigMap and Secret created 
for Kafka configuration. Use the following commands:

```
$ oc -n ztp-group-mb-du get configmap cluster-logging-kafka
$ oc -n ztp-group-mb-du get secret cluster-logging-kafka-auth
```


# Troubleshooting
If you encounter any issues during the deployment or configuration of the Kafka broker, consider the following troubleshooting steps:
   - Check the logs of the Kafka broker pods for any error messages:
     ```
     oc logs <kafka-broker-pod-name> -n openshift-operators
     ```
   - Ensure that the Kafka Operator is running and healthy.
   - Verify that the Zookeeper pods are also running and healthy, as Kafka relies on Zookeeper for coordination.
   - Check the status of the Kafka custom resource to ensure it is in a `Ready` state.
   - Review the OpenShift events for any warnings or errors related to the Kafka deployment:
     ```
     oc get events -n openshift-operators
     ```

# Conclusion
You have successfully deployed and configured a Kafka broker on the hub cluster.
If you have any questions or need further assistance, feel free to reach out to me (email: pmohanra@redhat.com) or 
refer to the additional resources provided below.

# Additional Resources
https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/logging/logging-6-2

https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/2.8/html/getting_started_with_streams_for_apache_kafka_on_openshift/proc-deploying-cluster-operator-kafka-str#proc-deploying-cluster-operator-kafka-str

https://github.com/openshift/cluster-logging-operator/tree/master?tab=readme-ov-file

https://vector.dev/

https://kafka.apache.org/40/documentation/streams/core-concepts

https://kafka.apache.org/40/documentation/streams/architecture

