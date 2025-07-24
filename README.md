# cp-flink-on-k8s-bootstrap-to-external-cp
Overall design of the set up describing a cp-flink deployment on K8s that bootstraps to a Confluent Platform cluster running external to K8s.

- cp-flink deployed on minikube using flink kubernetes operator and confluent manager for apache flink
- flink environment, compute pool and kafka catalog is created for cp-flink
- the flink service will use Kafka Catalog to bootstrap to an external Confluent Platform cluster (deployed via docker-compose not deployed on the same K8s cluster as cp-flink)
- kube networking to bootstrap to CP brokers and schema registry - this is a little bit tricky when running cp-flink on minikube and having to get it to bootstrap to an external CP running on docker
- run and test Flink SQL statements via Flink SQL shell

![cp-flink-minikube-external-cp-docker-compose](https://github.com/nav-nandan/cp-flink-k8s-bootstrap-cp-external/blob/main/cp-flink-minikube-to-external-cp-docker-compose.png)
