# cp-flink-on-k8s-bootstrap-to-external-cp
Overall design of the set up describing a cp-flink deployment on K8s that bootstraps to a Confluent Platform cluster running external to K8s.

- cp-flink deployed on minikube using flink kubernetes operator and confluent manager for apache flink
- flink environment, compute pool and kafka catalog is created for cp-flink
- the flink service will use Kafka Catalog to bootstrap to an external Confluent Platform cluster (deployed via docker-compose not deployed on the same K8s cluster as cp-flink)
- kube networking to bootstrap to CP brokers and schema registry - this is a little bit tricky when running cp-flink on minikube and having to get it to bootstrap to an external CP running on docker
- run and test Flink SQL statements via Flink SQL shell

![cp-flink-minikube-external-cp-docker-compose](https://github.com/nav-nandan/cp-flink-k8s-bootstrap-cp-external/blob/main/cp-flink-minikube-to-external-cp-docker-compose.png)

## Steps to setup
Clone the repo and set working directory
```
git clone https://github.com/nav-nandan/cp-flink-k8s-bootstrap-cp-external.git
cd cp-flink-k8s-bootstrap-cp-external
```

Ensure Docker (and/or Docker Desktop) is running locally
```
docker info
```

Deploy Confluent Platform cluster via docker-compose and verify services
```
docker-compose up -d

docker ps
CONTAINER ID   IMAGE                                                         COMMAND                  CREATED      STATUS      PORTS                                                                                                                                  NAMES
5df27b61eaa0   confluentinc/cp-schema-registry:7.9.2                         "/etc/confluent/dock…"   7 days ago   Up 7 days   8081/tcp, 0.0.0.0:39999->39999/tcp, [::]:39999->39999/tcp                                                                              schema-registry
faa4c5d5f46a   confluentinc/cp-server:7.9.2                                  "/etc/confluent/dock…"   7 days ago   Up 7 days   0.0.0.0:9092->9092/tcp, [::]:9092->9092/tcp                                                                                            broker
822c0a78b59b   confluentinc/cp-zookeeper:7.9.2                               "/etc/confluent/dock…"   7 days ago   Up 7 days   2181/tcp, 2888/tcp, 3888/tcp                                                                                                           zookeeper
69c4c4b32890   minio/minio:latest                                            "/usr/bin/docker-ent…"   7 days ago   Up 7 days   9000-9001/tcp
                                     minio
```

Start Minikube using Docker driver and verify services
```
minikube start --driver=docker

docker ps
CONTAINER ID   IMAGE                                                         COMMAND                  CREATED      STATUS      PORTS                                                                                                                                  NAMES
81d00e6264e6   gcr.io/k8s-minikube/kicbase-builds:v0.0.38-1680381266-16207   "/usr/local/bin/entr…"   7 days ago   Up 7 days   127.0.0.1:64364->22/tcp, 127.0.0.1:64365->2376/tcp, 127.0.0.1:64366->5000/tcp, 127.0.0.1:64367->8443/tcp, 127.0.0.1:64363->32443/tcp   minikube
5df27b61eaa0   confluentinc/cp-schema-registry:7.9.2                         "/etc/confluent/dock…"   7 days ago   Up 7 days   8081/tcp, 0.0.0.0:39999->39999/tcp, [::]:39999->39999/tcp                                                                              schema-registry
faa4c5d5f46a   confluentinc/cp-server:7.9.2                                  "/etc/confluent/dock…"   7 days ago   Up 7 days   0.0.0.0:9092->9092/tcp, [::]:9092->9092/tcp                                                                                            broker
822c0a78b59b   confluentinc/cp-zookeeper:7.9.2                               "/etc/confluent/dock…"   7 days ago   Up 7 days   2181/tcp, 2888/tcp, 3888/tcp                                                                                                           zookeeper
69c4c4b32890   minio/minio:latest                                            "/usr/bin/docker-ent…"   7 days ago   Up 7 days   9000-9001/tcp                                                                                                                          minio

minikube status           
🎉  minikube 1.36.0 is available! Download it: https://github.com/kubernetes/minikube/releases/tag/v1.36.0
💡  To disable this notice, run: 'minikube config set WantUpdateNotification false'

minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

kubectl get ns
NAME                   STATUS   AGE
default                Active   9d
kube-node-lease        Active   9d
kube-public            Active   9d
kube-system            Active   9d
kubernetes-dashboard   Active   51m
```

Check for host and gateway IP and DNS resolution
```
## On Host
ifconfig | grep inet

## Add host IP mapping to docker container hostnames to /etc/hosts on localhost
cat /etc/hosts
<host-ip> broker
<host-ip> zookepeer
<host-ip> schema-registry


## On Minikube Gateway
minikube ssh
docker@minikube:~$ ip a | grep inet

## Verify the Minikube gateway hostname is resolvable to host IP and respective Confluent Platform services are accessible
minikube ssh

docker@minikube:~$ nslookup host.minikube.internal
Server:		<gateway-ip>
Address:	<gateway-ip>#53

** server can't find host.minikube.internal: NXDOMAIN

## the gateway IP and special hostname (host.minikube.internal) is not reachable from host and is used internally by minikube

docker@minikube:~$ nc -vz host.minikube.internal 9092
Connection to host.minikube.internal 9092 port [tcp/*] succeeded!
docker@minikube:~$ nc -vz host.minikube.internal 8081
Connection to host.minikube.internal 8081 port [tcp/tproxy] succeeded!

```

[Install Confluent Manager for Flink](https://github.com/rjmfernandes/cp-flink-sql?tab=readme-ov-file#install-confluent-manager-for-apache-flink) and verify services
```
kubectl get ns        
NAME                   STATUS   AGE
cert-manager           Active   9d
confluent              Active   9d
default                Active   9d
kube-node-lease        Active   9d
kube-public            Active   9d
kube-system            Active   9d
kubernetes-dashboard   Active   63m

kubectl get svc                 
NAME                             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
cmf-service                      ClusterIP   10.108.42.70   <none>        80/TCP    9d
flink-operator-webhook-service   ClusterIP   10.104.88.9    <none>        443/TCP   9d
```

Expose `cmf-service` to be able to create Environment, Compute Pool and Kafka Catalog
```
kubectl port-forward svc/cmf-service 8080:80 -n confluent

export CONFLUENT_CMF_URL=http://127.0.0.1:8080

confluent flink environment create flink-sql --url $CONFLUENT_CMF_URL --kubernetes-namespace confluent

confluent flink environment list --url $CONFLUENT_CMF_URL
    Name    | Kubernetes Namespace |          Created Time          |          Updated Time           
------------+----------------------+--------------------------------+---------------------------------
  flink-sql | confluent            | 2025-07-15 05:07:36.862 +0000  | 2025-07-15 05:07:36.862 +0000   
            |                      | UTC                            | UTC                             

confluent flink compute-pool create flink/compute-pool.json --environment env1 --url http://localhost:8080

confluent flink compute-pool list --url $CONFLUENT_CMF_URL --environment flink-sql
       Creation Time       | Name |   Type    |   Phase    
---------------------------+------+-----------+------------
  2025-07-15T05:08:29.455Z | pool | DEDICATED | DEDICATED  

confluent flink catalog create catalog.json --url $CONFLUENT_CMF_URL

confluent flink catalog list --url $CONFLUENT_CMF_URL                        
       Creation Time       |   Name    | Databases  
---------------------------+-----------+------------
  2025-07-23T09:00:38.424Z | kafka-cat | kafka-1    
```

Open Flink SQL Shell and run SQL statements
```
confluent --environment flink-sql --compute-pool pool flink shell
Welcome! 
To exit, press Ctrl-Q or type "exit". 

[Ctrl-Q] Quit [Ctrl-S] Toggle Completions 
> SHOW TABLES;
Statement name: cli-2025-07-24-140514-cb06353e-4b4a-4857-a37f
Submitting statement...
Statement successfully submitted.
Details: Statement execution completed.
Finished statement execution. Statement phase: COMPLETED.
Details: Statement execution completed.
+-----------------------------+
|         table name          |
+-----------------------------+
| connect-configs             |
| connect-offsets             |
| connect-status              |
| default_ksql_processing_log |
| test                        |
+-----------------------------+
> 
```
