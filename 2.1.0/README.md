# What's new in AMQ Streams 2.1.0

## Prerequisites

* OpenShift 4.6 - 4.10 cluster
  * FIPS enabled cluster is required for the FIPS demo / lab.
    Other demos / labs should work on any cluster
  * Load Balancer support is needed for the load balancer demo

## Install AMQ Streams

1. Install AMQ Streams 2.1.0 using one of the available methods.
   You can also alternatively use the Strimzi 0.28.0 release, which corresponds to AMQ Streams 2.1.0.

## Running on FIPS enabled Kubernetes clusters

2. Deploy a new Apache Kafka 3.1.0 cluster.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/scholzj/what-is-new-in-amq-streams/main/2.1.0/kafka.yaml
   ```

3. You should see that the Strimzi Cluster Operator is crash-looping with an error similar to this:
   ```
   Exception in thread "main" io.fabric8.kubernetes.client.KubernetesClientException: An error has occurred.
      at io.fabric8.kubernetes.client.KubernetesClientException.launderThrowable(KubernetesClientException.java:103)
      at io.fabric8.kubernetes.client.KubernetesClientException.launderThrowable(KubernetesClientException.java:97)
      at io.fabric8.kubernetes.client.utils.HttpClientUtils.applyCommonConfiguration(HttpClientUtils.java:214)
      at io.fabric8.kubernetes.client.okhttp.OkHttpClientFactory.createHttpClient(OkHttpClientFactory.java:89)
      at io.fabric8.kubernetes.client.utils.HttpClientUtils.createHttpClient(HttpClientUtils.java:164)
      at io.fabric8.kubernetes.client.BaseClient.<init>(BaseClient.java:48)
      at io.fabric8.kubernetes.client.BaseClient.<init>(BaseClient.java:40)
      at io.fabric8.kubernetes.client.BaseKubernetesClient.<init>(BaseKubernetesClient.java:151)
      at io.fabric8.kubernetes.client.DefaultKubernetesClient.<init>(DefaultKubernetesClient.java:34)
      at io.strimzi.operator.cluster.Main.main(Main.java:75)
   Caused by: java.security.KeyStoreException: sun.security.pkcs11.wrapper.PKCS11Exception: CKR_SESSION_READ_ONLY
      at jdk.crypto.cryptoki/sun.security.pkcs11.P11KeyStore.engineSetEntry(P11KeyStore.java:1049)
      at jdk.crypto.cryptoki/sun.security.pkcs11.P11KeyStore.engineSetCertificateEntry(P11KeyStore.java:515)
      at java.base/java.security.KeyStore.setCertificateEntry(KeyStore.java:1235)
      at io.fabric8.kubernetes.client.internal.CertUtils.createTrustStore(CertUtils.java:100)
      at io.fabric8.kubernetes.client.internal.CertUtils.createTrustStore(CertUtils.java:74)
      at io.fabric8.kubernetes.client.internal.SSLUtils.trustManagers(SSLUtils.java:140)
      at io.fabric8.kubernetes.client.internal.SSLUtils.trustManagers(SSLUtils.java:90)
      at io.fabric8.kubernetes.client.utils.HttpClientUtils.applyCommonConfiguration(HttpClientUtils.java:203)
      ... 7 more
   Caused by: sun.security.pkcs11.wrapper.PKCS11Exception: CKR_SESSION_READ_ONLY
      at jdk.crypto.cryptoki/sun.security.pkcs11.wrapper.PKCS11.C_CreateObject(Native Method)
      at jdk.crypto.cryptoki/sun.security.pkcs11.wrapper.PKCS11$FIPSPKCS11.C_CreateObject(PKCS11.java:1950)
      at jdk.crypto.cryptoki/sun.security.pkcs11.P11KeyStore.storeCert(P11KeyStore.java:1567)
      at jdk.crypto.cryptoki/sun.security.pkcs11.P11KeyStore.engineSetEntry(P11KeyStore.java:1045)
      ... 14 more
   ```
   And no Kafka cluster is deployed.

4. Disable the FIPS mode by setting the `FIPS_MODE` environment variable in the Cluster Operator to `disabled`.
   You can do that by editing the deployment, editing the Operator Hub subscription or using `kubectl set env` command.
   ```
   kubectl set env deployment/strimzi-cluster-operator FIPS_MODE=disabled
   ```
5. Watch the operator pod to recreate and start deploying the Kafka cluster.

## Control Plane listener

### Check that the Control Plane Listener is used

6. Check that the Control Plane Listener on port 9090 is being used in the Kafka brokers:
   ```
   kubectl logs my-cluster-kafka-0 | grep acceptor
   ```
   You should see something like this:
   ```
   2021-12-23 16:21:48,137 INFO [SocketServer listenerType=ZK_BROKER, nodeId=0] Created control-plane acceptor and processor for endpoint : ListenerName(CONTROLPLANE-9090) (kafka.network.SocketServer) [main]
   ...
   2021-12-23 16:21:49,288 INFO [SocketServer listenerType=ZK_BROKER, nodeId=0] Started control-plane acceptor and processor(s) for endpoint : ListenerName(CONTROLPLANE-9090) (kafka.network.SocketServer) [main]
   ...
   ```
7. You can also check the configuration file of the Kafka broker:
   ```
   kubectl exec -ti my-cluster-kafka-0 -- cat /tmp/strimzi.properties | grep control.plane
   ```

### Disable the Control Plane Listener feature gate

8. Disable the `ControlPlaneListener` feature gate.
   You can do it by setting the `STRIMZI_FEATURE_GATES` environment variable in the Cluster Operator to contain `-ControlPlaneListener`.
   You can do that by editing the deployment, editing the Operator Hub subscription or using `kubectl set env` command.
   ```
   kubectl set env deployment/strimzi-cluster-operator STRIMZI_FEATURE_GATES=-ControlPlaneListener
   ```
9. Wait for the Kafka brokers to roll.
10. Check that the Control Plane Listener on port 9090 is not used in anymore:
    ```
    kubectl logs my-cluster-kafka-0 | grep control-plane
    ```
    You can also check the configuration file of the Kafka broker:
    ```
    kubectl exec -ti my-cluster-kafka-0 -- cat /tmp/strimzi.properties | grep control.plane
    ```

## Disabling bootstrap load balancers

11. Edit your Kafka cluster and add a `loadbalancer` type listener.
    Set the `createBootstrapService` option to `false` to avoid creating the bootstrap load balancer.
    ```yaml
      - name: lb
        port: 9094
        type: loadbalancer
        tls: true
        configuration:
          createBootstrapService: false
    ```

12. Wait for the cluster to roll and check the services / load balancers.
    You should see only the per-broker load balancers.

13. _(Bonus) Change the `createBootstrapService` flag to `true` or remove it completely and the operator should create the load balancer._

## Intra-broker balancing

## Bonus: Try the StrimziPodSets

13. Check the existing Kafka cluster.
    with the `kubectl get statefulsets` command you should see the StatefulSets which it uses to manage the pods.

14. To use the `StrimziPodSets`, you have to enable the `UseStrimziPodSets` feature gate
    You can do it by setting the `STRIMZI_FEATURE_GATES` environment variable in the Cluster Operator to contain `+UseStrimziPodSets`.
    You can do that by editing the deployment, editing the Operator Hub subscription or using `kubectl set env` command.
    ```
    kubectl set env deployment/strimzi-cluster-operator STRIMZI_FEATURE_GATES=+UseStrimziPodSets
    ```

15. Watch the operator to roll all the pods.
    Once the rolling-update is finished, check the StatefulSets again with `kubectl get statefulsets`.
    They should not exist anymore.
    Now check the StrimziPodSets instead with `kubectl get strimzipodsets` and you should see the pod sets being used for ZooKeeper nodes and Kafka brokers

16. Try to delete the individual Kafka or ZooKeeper pods and check that they are recreated by the Strimzi operator.
    You can also try other things such as rolling-updates, scaling etc.

### Cleanup

17. Once done you can delete all the Strimzi resources used during the demo:
    ```
    kubectl delete $(kubectl get strimzi -o name)
    ```
    And uninstall the operator.
