# What's new in AMQ Streams 2.4.0

## Prerequisites

* OpenShift cluster

## Install AMQ Streams

1. Install AMQ Streams 2.4.0 using one of the available methods.
   You can also alternatively use the Strimzi 0.34.0 release, which corresponds to AMQ Streams 2.4.0.

## Preparation

2. Create a new project named `myproject` and set it as the default namespace/project.
   If you decide to use a different namespace/project, you might need to adjust the labs accordingly.

3. Deploy a new Apache Kafka cluster and wait until it is ready.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.4.0/kafka.yaml
   ```

## Connector auto-restarting

4. Create a new topic `my-topic`.
   You can use the [`topic.yaml`](./topic.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.4.0/topic.yaml
   ```

5. Deploy a new Connect cluster and wait until it is ready.
   Use the [`connect.yaml`](./connect.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.4.0/connect.yaml
   ```
   The Connect deployment adds the [Echo Sink connector](https://github.com/scholzj/echo-sink) which we will use to demonstrate the auto-restarting feature.
   It also enables the connector operator.
   If needed, update the container registry where the new container image will be pushed to match your environment.

6. Create the connector and configure it to fail after receiving 5 messages.
   You can use the [`failing-connector.yaml`](./failing-connector.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.4.0/failing-connector.yaml
   ```
   It enables the auto-restart feature in `.spec.autoRestart`.
   Also notice the `fail.task.after.records: 5` settings in the connector configuration which tells it to fail after 5 records.
   The connector and its task should start and run without any issues because it is not receiving any messages yet.

7. Send five messages to the `my-topic` topic:
   ```
   kubectl run kafka-producer -ti --image=quay.io/strimzi/kafka:0.33.0-kafka-3.3.2 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic
   If you don't see a command prompt, try pressing enter.
   >Hello World 1
   >Hello World 2
   >Hello World 3
   >Hello World 4
   >Hello World 5
   ```
   After the fifth message, you should see in the Connect log that the connector task fails:
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
   2023-01-22 23:04:24 INFO  AbstractConnectOperator:692 - Reconciliation #73(timer) KafkaConnect(myproject/my-connect): Auto restarting connector echo-sink
   2023-01-22 23:04:24 INFO  AbstractConnectOperator:696 - Reconciliation #73(timer) KafkaConnect(myproject/my-connect): Restarted connector echo-sink
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

## Stable Pod-identities in Kafka Connect

11. Check the names of the Kafka Connect pods created by the deployment.
    Notice their _random_ names such as `my-connect-connect-85c5598cf7-mqrgr`:
    ```
    $ kubectl get pods
    NAME                                          READY   STATUS      RESTARTS        AGE
    ...
    my-connect-connect-76b7b6f458-bptp6           1/1     Running     0               5m25s
    my-connect-connect-76b7b6f458-m8zts           1/1     Running     0               5m25s
    my-connect-connect-76b7b6f458-p6pjq           1/1     Running     0               5m25s
    ...
    ```
    You can also check the Kubernetes Deployment managing the Kafka Connect cluster:
    ```
    $ kubectl get deployment my-connect-connect
    NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
    my-connect-connect   3/3     3            3           13m
    ```

12. Create some connectors which will be running (instead of the failing connector used in the previous section).
    You can use the [`connectors.yaml`](./connectors.yaml) file from this repository:
    ```
    kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.4.0/connectors.yaml
    ```
    The connectors and their tasks should start and send and receive messages.
    You can check the logs of the Kafka Connect pods to see that the messages are being sent and received

13. To use the table pod identities in Kafka Connect, you have to enable the `StableConnectIdentities` feature gate
    You can do it by setting the `STRIMZI_FEATURE_GATES` environment variable in the Cluster Operator to contain `+StableConnectIdentities`.
    * If you deployed it using YAML files, you can just edit the deployment file and add:
      ```yaml
      - name: STRIMZI_FEATURE_GATES
        value: +StableConnectIdentities
      ```
      Or use the `kubectl set env` command.
      ```
      kubectl set env deployment/strimzi-cluster-operator STRIMZI_FEATURE_GATES=+StableConnectIdentities
      ```
    * If you use Operator Hub, you can set it in the `Subscription` resource:
      ```yaml
      spec:
        # ...
        config:
          env:
            - name: STRIMZI_FEATURE_GATES
              value: +StableConnectIdentities
      ```

14. Watch the Kafka connect pods roll one by one.
    The new pods will have stable names which will not change with every restart such as `my-connect-connect-0` or `my-connect-connect-1`:
    ```
    $ kubectl get pods
    NAME                                          READY   STATUS      RESTARTS        AGE
    ...
    my-connect-connect-0                          1/1     Running     0               5m18s
    my-connect-connect-1                          1/1     Running     0               4m17s
    my-connect-connect-2                          1/1     Running     0               3m6s
    ...
    ```
    They will be also managed by a `StrimziPodSet` resource instead of Kubernetes Deployment:
    ```
    $ kubectl get strimzipodset my-connect-connect
    NAME                 PODS   READY PODS   CURRENT PODS   AGE
    my-connect-connect   3      3            3              5m54s
    ```
    You can check the logs from the Connect pods to see that the connectors still work.

## Stable Pod-identities in Kafka Mirror Maker 2

_On your own_

15. The `StableConnectIdentities` feature gate applies also to Kafka Mirror Maker 2 clusters.
    You can try it with Mirror Maker 2 as well.

## FIPS Mode

_On your own, on a FIPS enabled OpenShift cluster_

16. Deploy AMQ Streams 2.4.0 on a FIPS-enabled OpenShift cluster without disabling the FIPS mode in AMQ Streams using the `FIPS_MODE` environment variable.

17. Deploy a new Apache Kafka cluster and wait until it is ready.
    You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
    ```
    kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.4.0/kafka.yaml
    ```

18. Check that the Kafka cluster was deployed and is running.
    You can confirm that the FIPS mode is enabled:
    ```
    cat /proc/sys/crypto/fips_enabled
    ```
    If this file contains the value `1`, it means FIPS is enabled.

## Cleanup

19. Once done you can delete all the AMQ Streams resources used during the demo:
    ```
    kubectl delete $(kubectl get strimzi -o name)
    ```
    And uninstall the operator.
