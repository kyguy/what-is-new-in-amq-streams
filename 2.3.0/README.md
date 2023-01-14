# What's new in AMQ Streams 2.3.0

## Prerequisites

* OpenShift cluster

## Install AMQ Streams

1. Install AMQ Streams 2.3.0 using one of the available methods.
   You can also alternatively use the Strimzi 0.32.0 release, which corresponds to AMQ Streams 2.3.0.

## Leader Election

2. Check the configuration of the deployed Cluster Operator and notice the options related to the leader election:
   ```yaml
            - name: STRIMZI_LEADER_ELECTION_ENABLED
              value: "true"
            - name: STRIMZI_LEADER_ELECTION_LEASE_NAME
              value: "strimzi-cluster-operator"
            - name: STRIMZI_LEADER_ELECTION_LEASE_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: STRIMZI_LEADER_ELECTION_IDENTITY
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
   ```
   The leader election is enabled by default, even though only one replica is used by default.
   Check the docs for all available options.

3. Scale the Cluster Operator to use 2 replicas.
   The exact command might depend on how you installed AMQ Streams.
   You can use for example the `kubectl scale deployment` command
   ```
   kubectl scale deployment strimzi-cluster-operator --replicas 2
   ```

4. Check the newly created operator pod and its log.
   You should see that it is in the waiting mode:
   ```
   ...
   2022-11-01 20:25:35 INFO  Main:245 - Waiting to become a leader
   2022-11-01 20:25:36 INFO  LeaderElectionManager:119 - The new leader is strimzi-cluster-operator-6c8df5b94b-d4pjz
   ```

5. Try to kill the original pod which is currently a leader.
   You can do it for example using the `kubectl delete pod ...` command.
   That should trigger the leader election.
   In the log of the Pod which is the new leader, you should see following messages after which the pod starts managing the operands:
   ```
   2022-11-01 20:27:23 INFO  LeaderElectionManager:100 - Started being a leader
   2022-11-01 20:27:23 INFO  Main:233 - I'm the new leader
   ...
   ```
   _Note: There is no exact guarantee which of the remaining pods becomes the new leader._
   _It does not have to be the oldest one._
   _It can be also the pod started as a replacement for the pod we just deleted._

## `cluster-ip` listener

6. Deploy a new Apache Kafka cluster with Cruise Control, Authentication and Authorization enabled.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.3.0/kafka.yaml
   ```

7. The cluster we just deployed already uses the `cluster-ip` listener.
   You can check its configuration:
   ```yaml
    listeners:
      - name: tls
        port: 9093
        type: cluster-ip
        tls: true
        authentication:
          type: tls
   ```

8. List the services used by the Kafka cluster:
   ```
   kubectl get service
   ```
   And you should see (among other service) these 4 services used by the `cluster-ip` listener:
   ```
    NAME                             TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
    my-cluster-kafka-tls-0           ClusterIP      10.96.134.196    <none>          9093/TCP                     7m38s
    my-cluster-kafka-tls-1           ClusterIP      10.96.45.133     <none>          9093/TCP                     7m38s
    my-cluster-kafka-tls-2           ClusterIP      10.99.193.148    <none>          9093/TCP                     7m38s
    my-cluster-kafka-tls-bootstrap   ClusterIP      10.98.254.147    <none>          9093/TCP                     7m38s
    ...
   ```
   There are 3 per-broker services - `my-cluster-kafka-tls-0`, `my-cluster-kafka-tls-1`, and `my-cluster-kafka-tls-2` - one foreach broker.
   And one bootstrap service named `my-cluster-kafka-tls-bootstrap`.

9. Check the status of the Kafka resource:
   ```
   kubectl get kafka my-cluster -o yaml
   ```
   And you should see the bootstrap service also referenced there:
   ```yaml
    status:
      clusterId: q3xyOi2dSWGJA2TY9K7r7A
      conditions:
      - lastTransitionTime: "2022-11-01T21:25:45.622662Z"
        status: "True"
        type: Ready
      listeners:
      - addresses:
        - host: my-cluster-kafka-tls-bootstrap.myproject.svc
          port: 9093
        bootstrapServers: my-cluster-kafka-tls-bootstrap.myproject.svc:9093
        certificates:
        - |
          -----BEGIN CERTIFICATE-----
          ...
          -----END CERTIFICATE-----
        name: tls
        type: tls
      observedGeneration: 1
   ```

10. You can also check how the advertised hostnames are configured in the Kafka brokers:
    ```
    kubectl exec -ti my-cluster-kafka-0 -- cat /tmp/strimzi.properties | grep advertised.listeners
    ```
    And you should see something like this:
    ```
    advertised.listeners=CONTROLPLANE-9090://my-cluster-kafka-0.my-cluster-kafka-brokers.myproject.svc:9090,REPLICATION-9091://my-cluster-kafka-0.my-cluster-kafka-brokers.myproject.svc:9091,TLS-9093://my-cluster-kafka-tls-0.myproject.svc:9093
    ```
    Where the `cluster-ip` type listener is configured to advertise the address `my-cluster-kafka-tls-0.myproject.svc:9093`.

## Multiple operations per `ACLRule`

11. Deploy example clients using the Kafka cluster.
    You can use the [`clients.yaml`](./clients.yaml) file from this repository:
    ```
    kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.3.0/clients.yaml
    ```
    Once the consumer and producer are up and running, you can check that they work by checking the logs:
    ```
    kubectl logs deployment/example-consumer -f
    ```
    You should see `Hello World` messages being received by the consumer

12. Check the `KafkaUser` definition in the [`clients.yaml`](./clients.yaml) file and notice how the ACL rights are specified:
    You can see that a single ACL rule now covers multiple ACL operations:
    ```yaml
     authorization:
       type: simple
       acls:
         - resource:
             type: topic
             name: my-topic
           operations: 
             - Create
             - Describe
             - Read
             - Write
         # ...
     ```

## Rebalance auto-approval

13. With a running cluster and some clients using it, we can try to trigger a rebalance.
    In one terminal window, start watching for the rebalance resources:
    ```
    kubectl get kafkarebalance -o wide -w
    ```

14. Now create a rebalance resource.
    You can use the [`rebalance.yaml`](./rebalance.yaml) file from this repository:
    ```
    kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.3.0/rebalance.yaml
    ```
    Notice the `strimzi.io/rebalance-auto-approval: true` annotation.

15. Get back to the terminal where you are watching the rebalances.
    You should see how the rebalance was automatically approved and and moved through the `ProposalReady` stage to `Rebalancing` and finally to `Ready` automatically:
    ```
    NAME           CLUSTER      PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY
    my-rebalance   my-cluster
    my-rebalance   my-cluster   True
    my-rebalance   my-cluster                     True
    my-rebalance   my-cluster                                     True
    my-rebalance   my-cluster                                                   True
    ```

## Restart Events

16. Edit the Kafka custom resource and disable the topic auto-creation.
    You can do that using `kubectl edit kafka my-cluster` and adding the following to the `.spec.kafka.config` section:
    ```yaml
    auto.create.topics.enable: "false"
    ```

17. Watch the Kubernetes events with `kubectl get events -w`.

18. When the operator rolls the Kafka brokers, you should see the following events in your log:
    ```
    $ kubectl get events -w
    LAST SEEN   TYPE      REASON                  OBJECT                                              MESSAGE
    ...
    0s          Normal    ConfigChangeRequiresRestart   pod/my-cluster-kafka-0                              Pod needs to be restarted, because reconfiguration cannot be done dynamically
    ...
    ```

## Pod Security Providers

19. Enable the `restricted` Pod Security Provider.
    You can do it by setting the `STRIMZI_POD_SECURITY_PROVIDER_CLASS` environment variable in the Cluster Operator to contain `restricted`.
    * If you deployed it using YAML files, you can just edit the deployment file and add:
      ```yaml
      - name: STRIMZI_POD_SECURITY_PROVIDER_CLASS
        value: "restricted"
      ```
      Or use the `kubectl set env` command.
      ```
      kubectl set env deployment/strimzi-cluster-operator STRIMZI_POD_SECURITY_PROVIDER_CLASS=restricted
      ```
    * If you use Operator Hub, you can set it in the `Subscription` resource:
      ```yaml
      spec:
        # ...
        config:
          env:
            - name: STRIMZI_POD_SECURITY_PROVIDER_CLASS
              value: "restricted"
      ```

20. Watch all the pods of our Kafka cluster to roll as the changed security configuration is rolled out.
    Once the rolling update is finished, check the security context of the different pods.
    You should see the following security context for all containers:
    ```yaml
     securityContext:
       allowPrivilegeEscalation: false
       capabilities:
         drop:
         - ALL
       runAsNonRoot: true
       seccompProfile:
         type: RuntimeDefault
    ```
    You can use `kubectl` to look at it.
    Either using:
    ```
    kubectl get pod my-cluster-kafka-0 -o yaml
    ```
    Or for example using:
    ```
    kubectl get pod my-cluster-kafka-0 -o jsonpath='{.spec.containers[0].securityContext}' | jq
    ```
### Cleanup

21. Once done you can delete all the AMQ Streams resources used during the demo:
    ```
    kubectl delete $(kubectl get strimzi -o name)
    ```
    And uninstall the operator.
