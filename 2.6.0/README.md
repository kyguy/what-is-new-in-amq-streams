# What's new in AMQ Streams 2.6.0

## Prerequisites

* OpenShift or Kubernetes cluster

## Install AMQ Streams

1. Install AMQ Streams 2.6.0 using one o
f the available methods.
   You can also alternatively use the Strimzi 0.38.0 release, which corresponds to AMQ Streams 2.6.0.
 
```
kubectl apply -f https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.38.0/strimzi-cluster-operator-0.38.0.yaml
```

## Preparation

2. Create a new project named `myproject` and set it as the default namespace/project.
   If you decide to use a different namespace/project, you might need to adjust the labs accordingly.

3. Deploy a new Apache Kafka cluster and wait until it is ready.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/kyguy/what-is-new-in-amq-streams/main/2.6.0/kafka.yaml
   ```

4. Create a new topic `my-topic`.
   You can use the [`topic.yaml`](./topic.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/kyguy/what-is-new-in-amq-streams/main/2.6.0/topic.yaml
   ```

### Create Kafka Connect Cluster

5. Deploy a new Connect cluster and wait until it is ready.
   Use the [`connect.yaml`](./connect.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/kyguy/what-is-new-in-amq-streams/main/2.6.0/connect.yaml
   ```
   The Connect deployment adds the [Echo Sink connector](https://github.com/scholzj/echo-sink).
   It also enables the connector operator.
   If needed, update the container registry where the new container image will be pushed to match your environment.
   If needed, also create a secret with regitry credentials:
   ```
   kubectl create secret generic regcred --from-file=.dockerconfigjson=/home/k/.docker/config.json --type=kubernetes.io/dockerconfigjson
   ```
## Kafka Connect Improvements: Max restarts

6. Create the connector and configure it to fail after receiving 5 messages.
   You can use the [`failing-connector.yaml`](./failing-connector.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/kyguy/what-is-new-in-amq-streams/main/2.6.0/failing-connector.yaml
   ```
   It enables the auto-restart feature in `.spec.autoRestart`.
   Also notice the `fail.task.after.records: 5` settings in the connector configuration which tells it to fail after 5 records.
   The connector and its task should start and run without any issues because it is not receiving any messages yet.

7. Send five messages to the `my-topic` topic:
   ```
   kubectl run kafka-producer -ti --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic
   If you don't see a command prompt, try pressing enter.
   >Hello World 1
   >Hello World 2
   >Hello World 3
   >Hello World 4
   >Hello World 5
   ```
   After the fifth message, you should see in the Connect log that the connector task fails.
   You can use the following command to watch the Connect logs:
   ```
   kubectl logs pod/my-connect-connect-0 -f
   ```
   And you should see there an error similar to the following:
   ```
   2023-01-22 23:02:52,246 WARN [echo-sink|task-0] Failing as requested after 5 records (cz.scholz.kafka.connect.echosink.EchoSinkTask) [task-thread-echo-sink-0]
   2023-01-22 23:02:52,246 ERROR [echo-sink|task-0] WorkerSinkTask{id=echo-sink-0} Task threw an uncaught and unrecoverable exception. Task is being killed and will not recover until manually restarted. Error: Intentional task failure after receiving 5 records. (org.apache.kafka.connect.runtime.WorkerSinkTask) [task-thread-echo-sink-0]
   java.lang.RuntimeException: Intentional task failure after receiving 5 records.
     at cz.scholz.kafka.connect.echosink.EchoSinkTask.put(EchoSinkTask.java:131)
     at org.apache.kafka.connect.runtime.WorkerSinkTask.deliverMessages(WorkerSinkTask.java:581)
     at org.apache.kafka.connect.runtime.WorkerSinkTask.poll(WorkerSinkTask.java:333)
     at org.apache.kafka.connect.runtime.WorkerSinkTask.iteration(WorkerSinkTask.java:234)
     at org.apache.kafka.connect.runtime.WorkerSinkTask.execute(WorkerSinkTask.java:203)
     at org.apache.kafka.connect.runtime.WorkerTask.doRun(WorkerTask.java:189)
     at org.apache.kafka.connect.runtime.WorkerTask.run(WorkerTask.java:244)
     at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539)
     at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
     at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
     at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
     at java.base/java.lang.Thread.run(Thread.java:833)
   ```

8. Wait until the next reconciliation and check that the operator restarted the task.
   You should see log messages similar to the following in the operator log:
   ```
   2023-12-07 15:37:13,896 INFO [Worker clientId=connect-1, groupId=connect-cluster] Task 'echo-sink-0' restart successful (org.apache.kafka.connect.runtime.distributed.DistributedHerder) [DistributedHerder-connect-1-1]                                                                                 
   ```

   After the restart, you should also see the connector status to be updated:

   ```yaml
   # ...
   status:
     # ...
     autoRestart:
       count: 1
       lastRestartTimestamp: "2023-01-22T23:04:24.386944356Z"
   ```

9. Try to repeat this multiple times by sending more messages to the `my-topic` topic.

10. After you stop sending messages, the connector should recover and after some time, the auto-restart status should reset back to 0.
    You should see log messages similar to the following in the operator log:
    ```
    2023-01-22 23:26:24 INFO  AbstractConnectOperator:660 - Reconciliation #100(timer) KafkaConnect(myproject/my-connect): Resetting the auto-restart status of connector echo-sink
    ```
    And the `.status.autoRestart` section will be removed from the `KafkaConnect` resource

11. The connector should not be restarted more than once

## Kafka Connect Improvements: Stopping Connectors

12. Create the connectors, one to send messages to the Kafka Cluster, another to print the messages to the Kafka Connect log
   You can use the [`connectors.yaml`](./connectors.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/kyguy/what-is-new-in-amq-streams/main/2.6.0/connectors.yaml
   ```
13. You can use the following command to watch the Connect logs:
   
   ```
   kubectl logs pod/my-connect-connect-0 -f
   ```

14. You can stop the connector editing the KafkaConnector spec:

   ```
   kubectl edit kafkaconnector camel-timer-source
   ```

   To pause:

   ```
   apiVersion: kafka.strimzi.io/v1beta2
   kind: KafkaConnector
   metadata:
     annotations:
     ...
   spec:
     state: stopped
   ```
   
   Then resume:
   
   ```
   apiVersion: kafka.strimzi.io/v1beta2
   kind: KafkaConnector
   metadata:
     annotations:
    ...
   spec:
     state: running
   ```

   Other states: `paused`, `stopped`, `running`



## Kafka Connect Improvements: Manual rolling

15. Annotate Connect StrimziPodSet to preform a manual rolling update

   ```
   kubectl annotate strimzipodset my-connect-connect strimzi.io/manual-rolling-update=true
   ``` 

16. Now Connect StrimziPodSet has been annotated

   ```
   kubectl edit strimzipodset my-connect-cluster-connect
   ```

   to see:

   ```
   apiVersion: core.strimzi.io/v1beta2
   kind: StrimziPodSet
   metadata:
     annotations:
       strimzi.io/manual-rolling-update: "true"
     creationTimestamp: "2023-12-05T21:40:46Z"
    ...
  ...
  ```

17. Wait for rolling update of Connect pods

18. Look at StrimziPodSet after rolling update

   ```
   kubectl edit strimzipodset my-connect-cluster-connect
   ```

   to see that annotation has been removed

## Cleanup

19. Once done you can delete all the AMQ Streams resources used during the demo:
    ```
    kubectl delete $(kubectl get strimzi -o name)
    ```
    And uninstall the operator.
