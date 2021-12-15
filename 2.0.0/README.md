# What's new in AMQ Streams 2.0.0

## Prerequisites

* OpenShift 4.9, 4.8, 4.7 or 4.6 cluster
  * Multiple worker nodes are needed for the Drain Cleaner lab

## Install AMQ Streams

* Install AMQ Streams 2.0.0 using one of the available methods.
  You can also alternatively use the Strimzi 0.26.0 release, which corresponds to AMQ Streams 2.0.0.

## User Operator

### Preparation

1. Deploy a Kafka cluster with different listeners using TLS and SCRAM-SHA-512 authentication and User Operator enabled.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.0.0/kafka.yaml
   ```

### Custom Password

2. Create the secret with the desired password.
   The secret should have a name different from the regular `KafkaUser` secret:
   ```
   kubectl create secret generic scram-sha-user-custom-password --from-literal=customPassword='myRandomAndSecretPassword'
   ```
3. Create the `KafkaUser` and reference the password from the secret created in the previous step:
   ```yaml
   # ...
   authentication:
     type: scram-sha-512
     password:
       valueFrom:
         secretKeyRef:
           name: scram-sha-user-custom-password
           key: customPassword
   # ...
   ```
   * You can use the [`scram-sha-user.yaml`](./scram-sha-user.yaml) file from this repository:
     ```
     kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.0.0/scram-sha-user.yaml
     ```
4. Check the created user:
   ```
   kubectl get kafkauser scram-sha-user -o yaml
   ```
   * The condition should be ready:
     ```yaml
     status:
       conditions:
         - lastTransitionTime: "2021-08-09T19:20:29.029522Z"
           status: "True"
           type: Ready
     ```
   * The custom password should also in the user secret:
     ```
     kubectl get secret scram-sha-user -o jsonpath='{.data.password}' | base64 -d
     ```
   * We can also check that we can connect with a client:
     * List metadata
       ```
       kubectl run kafka-metadata -ti --image=edenhill/kafkacat:1.6.0 --rm=true --restart=Never -- -b my-cluster-kafka-bootstrap:9092 -X security.protocol=SASL_PLAINTEXT -X sasl.mechanism=SCRAM-SHA-512 -X sasl.username=scram-sha-user -X sasl.password=myRandomAndSecretPassword -L
       ```
     * Produce and consume messages
       ```
       kubectl run kafka-producer -ti --image=edenhill/kafkacat:1.6.0 --rm=true --restart=Never -- -b my-cluster-kafka-bootstrap:9092 -X security.protocol=SASL_PLAINTEXT -X sasl.mechanism=SCRAM-SHA-512 -X sasl.username=scram-sha-user -X sasl.password=myRandomAndSecretPassword -t my-topic -P
       kubectl run kafka-consumer -ti --image=edenhill/kafkacat:1.6.0 --rm=true --restart=Never -- -b my-cluster-kafka-bootstrap:9092 -X security.protocol=SASL_PLAINTEXT -X sasl.mechanism=SCRAM-SHA-512 -X sasl.username=scram-sha-user -X sasl.password=myRandomAndSecretPassword -C -t my-topic -o beginning
       ```
5. You can also try what happens when the secret does not exist.
   The User Operator will not generate random password but wait for the secret to be created.
   You can try that by creating [`scram-sha-user2.yaml`](./scram-sha-user.yaml):
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.0.0/scram-sha-user2.yaml
   ```
6. The user should fail to create with the following condition:
   ```yaml
   status:
     conditions:
       - lastTransitionTime: "2021-08-09T19:16:51.116530Z"
         message: Secret scram-sha-user-custom-password with requested user password does not exist.
         reason: InvalidResourceException
         status: "True"
         type: NotReady
   ```
7. Create the secret and check that the next reconciliation creates the user:
   ```
   kubectl create secret generic scram-sha-user2-custom-password --from-literal=customPassword='myOtherRandomAndSecretPassword'
   ```

### `tls-external` authentication

8. Create a `KafkaUser` with `tls-external` authentication type.
   You can use the [`tls-external-user.yaml`](./tls-external-user.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.0.0/tls-external-user.yaml
   ```
9. Check that the user was created, but without generating the certificate
   * Check that the user secret was not created:
     ```
     kubectl get secret tls-external-user
     ```
   * Check that the ACLs were set for the user `CN=tls-external-user`:
     ```
     kubectl exec -ti my-cluster-zookeeper-0 -- bin/zookeeper-shell.sh localhost:12181 get /kafka-acl/Topic/my-topic
     ```

## Kafka Connect

### Deploy Kafka Connect with `maven` type artifact

2. Check the [`connect.yaml`](./connect.yaml) file with the Kafka Connect configuration.
   Notice the `camel-timer-connector` which uses the new `maven` artifact type and will download the connector directly from Maven repositories.
   ```yaml
   - name: camel-timer-connector
     artifacts:
       - type: maven
         group: org.apache.camel.kafkaconnector
         artifact: camel-timer-kafka-connector
         version: 0.11.0
    ```
3. Before deploying the Kafka Connect cluster, you need to configure the container repository where the new container with the additional plugins should be pushed.
   If you have some registry where you can push without authentication, you can just update the image address in the `output` section of [`connect.yaml`](./connect.yaml):
   ```yaml
    output:
      type: docker
      image: my-registry:5000/myproject/kafka-connect:latest
   ```
   If your registry requires authentication, you need to first create the secret with credentials.
   The secret should look similar to this:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: docker-credentials
   type: kubernetes.io/dockerconfigjson
   data:
     .dockerconfigjson: Cg==
   ```
   You can follow the [Kubernetes documentation](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry) for more details how to create it.
   Once you have the secret, you have to reference it in the `output` section of [`connect.yaml`](./connect.yaml):
   ```yaml
   output:
     type: docker
     image: docker.io/my-org/kafka-connect:latest
     pushSecret: docker-credentials
   ```
4. Deploy the Kafka Connect cluster:
   ```
   kubectl apply -f connect.yaml
   ```
   Wait until the build is finished and the Connect cluster is deployed:
   ```
   kubectl wait kafkaconnect/my-connect --for=condition=Ready --timeout=300s
   ```
5. Check the available connectors in the status of the `KafkaConnect` custom resource:
   ```
   kubectl get kafkaconnect my-connect -o yaml
   ```
   You should see something like this:
   ```yaml
   # ...
   status:
     conditions:
      - lastTransitionTime: "2021-10-14T15:30:22.766432Z"
        status: "True"
        type: Ready
     connectorPlugins:
       - class: cz.scholz.kafka.connect.echosink.EchoSinkConnector
         type: sink
         version: 1.2.0
       - class: org.apache.camel.kafkaconnector.CamelSinkConnector
         type: sink
         version: 0.11.0
       - class: org.apache.camel.kafkaconnector.CamelSourceConnector
         type: source
         version: 0.11.0
       - class: org.apache.camel.kafkaconnector.timer.CamelTimerSourceConnector
         type: source
         version: 0.11.0
       - class: org.apache.kafka.connect.file.FileStreamSinkConnector
         type: sink
         version: 3.0.0
       - class: org.apache.kafka.connect.file.FileStreamSourceConnector
         type: source
         version: 3.0.0
       - class: org.apache.kafka.connect.mirror.MirrorCheckpointConnector
         type: source
         version: "1"
       - class: org.apache.kafka.connect.mirror.MirrorHeartbeatConnector
         type: source
         version: "1"
       - class: org.apache.kafka.connect.mirror.MirrorSourceConnector
         type: source
         version: "1"
     labelSelector: strimzi.io/cluster=my-connect,strimzi.io/name=my-connect-connect,strimzi.io/kind=KafkaConnect
     observedGeneration: 1
     replicas: 1
     url: http://my-connect-connect-api.myproject.svc:8083
   ```
   Notice the `CamelTimerSourceConnector` connector.
   We can also check inside the new container that the connector was pulled including all dependencies:
   ```
   kubectl exec -ti deployment/my-connect-connect -- ls plugins/camel-timer-connector/1a9cd13e
   ```
   Which should show output similar or equal to this:
   ```
   annotations-13.0.jar
   apicurio-registry-common-1.3.2.Final.jar
   apicurio-registry-rest-client-1.3.2.Final.jar
   apicurio-registry-utils-converter-1.3.2.Final.jar
   apicurio-registry-utils-serde-1.3.2.Final.jar
   avro-1.10.2.jar
   camel-api-3.11.1.jar
   camel-base-3.11.1.jar
   camel-base-engine-3.11.1.jar
   camel-core-engine-3.11.1.jar
   camel-core-languages-3.11.1.jar
   camel-core-model-3.11.1.jar
   camel-core-processor-3.11.1.jar
   camel-core-reifier-3.11.1.jar
   camel-direct-3.11.1.jar
   camel-jackson-3.11.1.jar
   camel-kafka-3.11.1.jar
   camel-kafka-connector-0.11.0.jar
   camel-main-3.11.1.jar
   camel-management-api-3.11.1.jar
   camel-seda-3.11.1.jar
   camel-support-3.11.1.jar
   camel-timer-3.11.1.jar
   camel-timer-kafka-connector-0.11.0.jar
   camel-util-3.11.1.jar
   commons-compress-1.20.jar
   connect-api-2.8.0.jar
   connect-json-2.6.0.jar
   connect-transforms-2.8.0.jar
   converter-jackson-2.9.0.jar
   jackson-annotations-2.12.3.jar
   jackson-core-2.12.3.jar
   jackson-databind-2.12.3.jar
   jackson-dataformat-avro-2.12.2.jar
   jackson-datatype-jdk8-2.12.2.jar
   javax.annotation-api-1.3.2.jar
   javax.ws.rs-api-2.1.1.jar
   jboss-jaxrs-api_2.1_spec-2.0.1.Final.jar
   jctools-core-3.3.0.jar
   kafka-clients-2.8.0.jar
   kotlin-reflect-1.3.20.jar
   kotlin-stdlib-1.3.20.jar
   kotlin-stdlib-common-1.3.20.jar
   lz4-java-1.7.1.jar
   medeia-validator-core-1.1.1.jar
   medeia-validator-jackson-1.1.1.jar
   okhttp-4.8.1.jar
   okio-2.7.0.jar
   protobuf-java-3.13.0.jar
   retrofit-2.9.0.jar
   slf4j-api-1.7.30.jar
   snappy-java-1.1.8.1.jar
   zstd-jni-1.4.9-1.jar  
   ```
6. With the connector plugins ready, we can now create the connector instances using the file [`connectors.yaml`](./connectors.yaml) which creates:
     * Kafka topic named `timer-topic`
     * Connector named `camel-timer-connector` which will send a new message every second
     * Connector names `echo-sink-timer-connector` which will read the messages from the first connector and print their content into the log
   You can deploy it using:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.0.0/connectors.yaml
   ```
7. Once the connectors are running, you can check the Connect logs:
   ```
   kubectl logs -f deployment/my-connect-connect
   ```
   And you should see the messages sent by the timer connector:
   ```
   2021-10-14 15:48:21,569 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226501519}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   2021-10-14 15:48:22,570 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226502520}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   2021-10-14 15:48:23,571 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226503520}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   2021-10-14 15:48:24,571 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226504521}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   2021-10-14 15:48:25,571 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226505521}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   2021-10-14 15:48:26,572 INFO Received message with key 'null' and value '{"message":"Hello World","timestamp":1634226506522}' (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-timer-connector-0]
   ```

## Drain cleaner

1. Install Drain Cleaner in your cluster.
   You can use the AMQ Streams 2.0.0 installation files or the files from https://github.com/strimzi/drain-cleaner/releases/tag/0.2.0.
   You can also use the installation files from this repository (Bbased on upstream Strimzi Drain Cleaner 0.2.0):
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.0.0/drain-cleaner.yaml
   ```
2. Open a new terminal to watch the Drain Cleaner logs:
   ```
   kubectl logs -f deployment/strimzi-drain-cleaner -n strimzi-drain-cleaner
   ```
3. Make sure your Kafka cluster is configured with `maxUnavailable: 0` Pod Disruption Budget for both ZooKeeper and Kafka:
   ```yaml
   template:
     podDisruptionBudget:
       maxUnavailable: 0
   ```
4. List your pods to check the worker nodes on which they are running:
   ```
   kubectl get pods -o wide
   ```
   You should see output similar to this:
   ```
   NAME                                          READY   STATUS      RESTARTS   AGE   IP            NODE                                       NOMINATED NODE   READINESS GATES
   my-cluster-entity-operator-5cf9f67dd6-8gcx9   3/3     Running     0          65m   10.129.2.15   ci-ln-7jz8rnt-72292-5ltvx-worker-a-r4v7m   <none>           <none>
   my-cluster-kafka-0                            1/1     Running     0          66m   10.131.0.53   ci-ln-7jz8rnt-72292-5ltvx-worker-b-4s9sd   <none>           <none>
   my-cluster-kafka-1                            1/1     Running     0          66m   10.129.2.14   ci-ln-7jz8rnt-72292-5ltvx-worker-a-r4v7m   <none>           <none>
   my-cluster-kafka-2                            1/1     Running     0          66m   10.128.2.12   ci-ln-7jz8rnt-72292-5ltvx-worker-c-5slqn   <none>           <none>
   my-cluster-zookeeper-0                        1/1     Running     0          67m   10.131.0.50   ci-ln-7jz8rnt-72292-5ltvx-worker-b-4s9sd   <none>           <none>
   my-cluster-zookeeper-1                        1/1     Running     0          67m   10.128.2.11   ci-ln-7jz8rnt-72292-5ltvx-worker-c-5slqn   <none>           <none>
   my-cluster-zookeeper-2                        1/1     Running     0          67m   10.129.2.13   ci-ln-7jz8rnt-72292-5ltvx-worker-a-r4v7m   <none>           <none>
   my-connect-connect-build-1-build              0/1     Completed   0          41m   10.131.0.68   ci-ln-7jz8rnt-72292-5ltvx-worker-b-4s9sd   <none>           <none>
   my-connect-connect-f59bf69f9-d9mcv            1/1     Running     0          39m   10.131.0.70   ci-ln-7jz8rnt-72292-5ltvx-worker-b-4s9sd   <none>           <none>
   ```
5. Pick one of the nodes with at least one Kafka broker and drain it.
   For example:
   ```
   kubectl drain ci-ln-7jz8rnt-72292-5ltvx-worker-c-5slqn --delete-emptydir-data --ignore-daemonsets --timeout=6000s --force
   ```
   And observe the draining.

   In the output of the `kubectl drain` command, you should see that it is not shuting down the ZooKeeper or Kafka pods automatically:
   ```
   ...
   error when evicting pods/"my-cluster-kafka-2" -n "myproject" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
   error when evicting pods/"my-cluster-zookeeper-1" -n "myproject" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
   ...
   ```

   In the Drain Cleaner logs, you should see how it caught the eviction requests and annotated the pods for rolling update by the operator:
   ```
   ...
   2021-12-15 00:08:21,532 INFO  [io.str.ValidatingWebhook] (executor-thread-9) Received eviction webhook for Pod my-cluster-zookeeper-1 in namespace myproject
   2021-12-15 00:08:21,533 INFO  [io.str.ValidatingWebhook] (executor-thread-11) Received eviction webhook for Pod my-cluster-kafka-2 in namespace myproject
   2021-12-15 00:08:21,628 INFO  [io.str.ValidatingWebhook] (executor-thread-11) Pod my-cluster-kafka-2 in namespace myproject should be annotated for restart
   2021-12-15 00:08:21,660 INFO  [io.str.ValidatingWebhook] (executor-thread-9) Pod my-cluster-zookeeper-1 in namespace myproject should be annotated for restart
   2021-12-15 00:08:21,800 INFO  [io.str.ValidatingWebhook] (executor-thread-11) Pod my-cluster-kafka-2 in namespace myproject was patched
   2021-12-15 00:08:21,816 INFO  [io.str.ValidatingWebhook] (executor-thread-9) Pod my-cluster-zookeeper-1 in namespace myproject was patched
   ...
   ```
   You might also see some messages about other pods being evicted which are ignored by the Drain Cleaner tool.

   Finally, in the Cluster Operator log, you should see how the operator is rolling the pods safely:
   ```
   ...
   2021-12-15 00:08:24 INFO  AbstractOperator:219 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Kafka my-cluster will be checked for creation or modification
   2021-12-15 00:08:25 INFO  ZookeeperLeaderFinder:92 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Trusting certificate ca.crt from Secret my-cluster-cluster-ca-cert
   2021-12-15 00:08:25 WARN  ZookeeperLeaderFinder:101 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Ignoring non-certificate ca.p12 in Secret my-cluster-cluster-ca-cert
   2021-12-15 00:08:25 WARN  ZookeeperLeaderFinder:101 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Ignoring non-certificate ca.password in Secret my-cluster-cluster-ca-cert
   2021-12-15 00:08:25 INFO  ZookeeperLeaderFinder:225 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Pod my-cluster-zookeeper-0 is not a leader
   2021-12-15 00:08:25 INFO  ZookeeperLeaderFinder:225 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Pod my-cluster-zookeeper-1 is not a leader
   2021-12-15 00:08:25 INFO  ZookeeperLeaderFinder:222 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Pod my-cluster-zookeeper-2 is leader
   2021-12-15 00:08:25 INFO  PodOperator:68 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Rolling pod my-cluster-zookeeper-1
   2021-12-15 00:09:24 INFO  AbstractOperator:363 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Reconciliation is in progress
   2021-12-15 00:09:39 INFO  KafkaRoller:561 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Dynamic reconfiguration for broker 0 was successful.
   2021-12-15 00:09:40 INFO  KafkaRoller:299 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Could not roll pod 1 due to io.strimzi.operator.cluster.operator.resource.KafkaRoller$ForceableProblem: Pod my-cluster-kafka-1 is currently the controller and there are other pods still to roll, retrying after at least 250ms
   2021-12-15 00:09:40 INFO  KafkaRoller:507 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Pod 2 needs to be restarted. Reason: [manual rolling update annotation on a pod]
   2021-12-15 00:09:40 INFO  PodOperator:68 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Rolling pod my-cluster-kafka-2
   2021-12-15 00:10:24 INFO  ClusterOperator:118 - Triggering periodic reconciliation for namespace *
   2021-12-15 00:10:24 INFO  AbstractOperator:363 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Reconciliation is in progress
   2021-12-15 00:10:25 INFO  KafkaRoller:561 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Dynamic reconfiguration for broker 1 was successful.
   2021-12-15 00:10:26 INFO  KafkaRoller:561 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Dynamic reconfiguration for broker 0 was successful.
   2021-12-15 00:10:26 INFO  KafkaRoller:299 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Could not roll pod 1 due to io.strimzi.operator.cluster.operator.resource.KafkaRoller$ForceableProblem: Pod my-cluster-kafka-1 is currently the controller and there are other pods still to roll, retrying after at least 250ms
   2021-12-15 00:10:27 INFO  KafkaRoller:561 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Dynamic reconfiguration for broker 2 was successful.
   2021-12-15 00:10:27 INFO  KafkaRoller:561 - Reconciliation #79(timer) Kafka(myproject/my-cluster): Dynamic reconfiguration for broker 1 was successful.
   2021-12-15 00:10:28 INFO  AbstractOperator:466 - Reconciliation #79(timer) Kafka(myproject/my-cluster): reconciled
   ...
   ```
5. Once you are finished, you can uncordon the worker node to be able to use it again
   ```
   kubectl uncordon ci-ln-7jz8rnt-72292-5ltvx-worker-c-5slqn
   ```

## Disabling Network Policies

1. Edit the Cluster Operator deployment and set the environment variable `STRIMZI_NETWORK_POLICY_GENERATION` to `false`.
   * If you deployed it using YAML files, you can just edit the deployment file and add:
     ```yaml
     - name: STRIMZI_NETWORK_POLICY_GENERATION
       value: "false"
     ```
   * If you use Operator Hub, you can set it in the `Subscription` resource:
     ```yaml
     spec:
     # ...
     config:
       env:
         - name: STRIMZI_NETWORK_POLICY_GENERATION
         value: "false"
     ```
2. Use the existing Kafka cluster and check the effect of the disabled Network Policies
     * Edit one of the network policies and save the changes
       ```
       kubectl edit networkpolicy my-cluster-network-policy-kafka
       ```
     * Watch that the operator will not revert the changes in any way since the network policies were disabled
       ```
       kubectl get networkpolicy my-cluster-network-policy-kafka -o yaml
       ```
3. _(On your own)_ 
   Deploy a fresh Kafka cluster and check that for a new Kafka cluster, not network policies will be created

### Cleanup

4. Delete the Kafka cluster and the network policies after the demo:
   ```
   kubectl delete kafka my-cluster
   kubectl delete networkpolicy my-cluster-network-policy-kafka
   ```

### Cleanup

8. Once done you can delete all the Strimzi resources used during the demo:
   ```
   kubectl delete $(kubectl get strimzi -o name)
   ```
   And uninstall the operator.
   If you created the secret with the container registry credentials, don't forget to delete it as well.