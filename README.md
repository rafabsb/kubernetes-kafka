

# Kafka on Kubernetes

Transparent Kafka setup that you can grow with.
Good for both experiments and production.

How to use:
 * Simply install Kafka-Manager _(yahoo/Kafka-Manager)_ after starting Zookeeper and Kafka Brokers, as described bellow.
 * Good to know: you'll likely want to fork this repo. It prioritizes clarity over configurability, using plain manifests and .propeties files; no client side logic.
 * Run a Kubernetes cluster, [minikube](https://github.com/kubernetes/minikube) or real.
 * Quickstart: use the `kubectl apply`s below.
 * Have a look at [addon](https://github.com/Yolean/kubernetes-kafka/labels/addon)s, or the official forks:
   - [kubernetes-kafka-small](https://github.com/Reposoft/kubernetes-kafka-small) for single-node clusters like Minikube.
   - [StreamingMicroservicesPlatform](https://github.com/StreamingMicroservicesPlatform/kubernetes-kafka) Like Confluent's [platform quickstart](https://docs.confluent.io/current/connect/quickstart.html) but for Kubernetes.
 * Join the discussion in issues and PRs.

No readable readme can properly introduce both [Kafka](http://kafka.apache.org/) and [Kubernetes](https://kubernetes.io/),
but we think the combination of the two is a great backbone for microservices.
Back when we read [Newman](http://samnewman.io/books/building_microservices/) we were beginners with both.
Now we've read [Kleppmann](http://dataintensive.net/), [Confluent](https://www.confluent.io/blog/) and [SRE](https://landing.google.com/sre/book.html) and enjoy this "Streaming Platform" lock-in :smile:.

We also think the plain-yaml approach of this project is easier to understand and evolve than [helm](https://github.com/kubernetes/helm) [chart](https://github.com/kubernetes/charts/tree/master/incubator/kafka)s.

## What you get

Keep an eye on `kubectl --namespace kafka get pods -w`.

The goal is to provide [Bootstrap servers](http://kafka.apache.org/documentation/#producerconfigs): `kafka-0.broker.kafka.svc.cluster.local:9092,kafka-1.broker.kafka.svc.cluster.local:9092,kafka-2.broker.kafka.svc.cluster.local:9092`
`

Zookeeper at `zookeeper.kafka.svc.cluster.local:2181`.

## Start Zookeeper

The [Kafka book](https://www.confluent.io/resources/kafka-definitive-guide-preview-edition/) recommends that Kafka has its own Zookeeper cluster with at least 5 instances.

```
kubectl apply -f ./zookeeper/
```

To support automatic migration in the face of availability zone unavailability we mix persistent and ephemeral storage.

## Start Kafka

```
kubectl apply -f ./
```

You might want to verify in logs that Kafka found its own DNS name(s) correctly. Look for records like:
```
kubectl -n kafka logs kafka-0 | grep "Registered broker"
# INFO Registered broker 0 at path /brokers/ids/0 with addresses: PLAINTEXT -> EndPoint(kafka-0.broker.kafka.svc.cluster.local,9092,PLAINTEXT)
```

That's it. Just add business value :wink:.
For clients we tend to use [librdkafka](https://github.com/edenhill/librdkafka)-based drivers like [node-rdkafka](https://github.com/Blizzard/node-rdkafka).
To use [Kafka Connect](http://kafka.apache.org/documentation/#connect) and [Kafka Streams](http://kafka.apache.org/documentation/streams/) you may want to take a look at our [sample](https://github.com/solsson/dockerfiles/tree/master/connect-files) [Dockerfile](https://github.com/solsson/dockerfiles/tree/master/streams-logfilter)s.
And don't forget the [addon](https://github.com/Yolean/kubernetes-kafka/labels/addon)s.

## Start Kafka-Manager

```
kubectl apply -f ./kafka-manager/
```

Kafka Manager default listen port: 9000 TCP.
To access via web-browser:

1 - Take note of your cluster IP _(19.11.47.10 in the example bellow)_:
```
kubectl cluster-info
Kubernetes master is running at https://19.11.47.10:313
```

2 - Take note of kafka-manager service proxy port _(33333 in the example bellow)_
kubectl get services --namespace=kafka kafka-manager
```
NAME            TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
kafka-manager   NodePort   10.0.0.236   <none>        9000:33333/TCP   1d
```

3 - Head to your web-browser and insert:
```
https://19.11.47.10:33333
```

4 - _Voilà_, your kafka Manager is up and Running! You can know insert your kafka cluster settings!

## RBAC

For clusters that enfoce [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) there's a minimal set of policies in
```
kubectl apply -f rbac-namespace-default/
```

## Caution: `Delete` Reclaim Policy is default

In production you likely want to [manually set Reclaim Policy](https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/),
or your data will be gone if the generated [volume claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)s are deleted.

This can't be done [in manifests](https://github.com/Yolean/kubernetes-kafka/pull/50),
at least not [until Kubernetes 1.8](https://github.com/kubernetes/features/issues/352).

## Tests

Tests are based on the [kube-test](https://github.com/Yolean/kube-test) concept.
Like the rest of this repo they have `kubectl` as the only local dependency.

```
kubectl apply -f test/
# Anything that isn't READY here is a failed test
kubectl get pods -l test-target=kafka,test-type=readiness -w --all-namespaces
```
