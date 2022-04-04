# || HA Elasticsearch ||

### |An architecture overview|

![alt text](https://github.com/YousraBorchani/HA-ELASTICSEARCH/blob/main/ha-es-architecture.png?raw=true)


### |Set up a cluster for high availability|
1. Designing for resilience
    - At least three master-eligible nodes
    - At least two nodes of each role 
        - Master-eligible node: 
            Controls the cluster. 
        - Data node:
            Hold data and perform data related operations such as CRUD, search, and aggregations. 
        - Ingest node: 
           Apply an ingest pipeline to a document in order to transform and enrich the document before indexing. 

    - At least two copies of each shard 
2. Cross-cluster replication    
    - Replicate data to a remote follower cluster which may be in a different data 
    centre or even on a different continent from the leader cluster. 
    - The follower cluster acts as a hot standby, ready for you to fail over in the 
    event of a disaster so severe that the leader cluster fails. 
    - The follower cluster can also act as a geo-replica to serve searches from nearby clients.  
3. Regular snapshots 
    - Restore a completely fresh copy of it elsewhere if needed

### |Scenario: Is my cluster really highly available?|
For a cluster to be able to handle the failure of a node, 
it should be considered at capacity when it uses 50% of its resources 
Therefore you need to increase its size or reduce its workload

### |Deploy ES cluster | 

1. Configure kubectl

```
aws eks --region $(terraform output -raw region) update-kubeconfig --name (terraform output -raw cluster_name)
```

2. Deploy ECK in your Kubernetes cluster
```
kubectl apply -f https://download.elastic.co/downloads/eck/1.0.1/all-in-one.yaml
```

3. Monitor the operator logs
```
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

4. Create storage class
```
kubectl create -f storage-class.yaml
```

5. Switch the default class to false for the standard storage class
```
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

5. Switch the default class to true for the custom storage class
```
kubectl patch storageclass zone-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

5. The operator automatically creates and manages Kubernetes resources to achieve the desired state of the Elasticsearch cluster. 
```
kubectl apply -f elasticsearch.yaml
```

6. Monitor cluster health and creation progress
```
kubectl get elasticsearch
```

7. Retrieve the CA certificate
```
kubectl get secret "ha-es-http-certs-public" -o go-template='{{index .data "tls.crt" | base64decode }}' > tls.crt
```
8. Retrieve the password of the elastic user
```
PW=$(kubectl get secret ha-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode)
```

9. Retrieve the IP of the LoadBalancer Service
```
IP=$(kubectl get svc ha-es-http -o jsonpath='{.status.loadBalancer.ingress[].hostname}')
```
10. For testing purpose i'm using insucure mode 
```
curl --cacert tls.crt -k -u elastic:$PW https://$IP:9200/
```

### |Deploy Kibana cluster | 

1. Deploy Kibana Cluster
```
kubectl apply -f kibana.yaml
```

### |Production ready |

- Create your own certificate
- Deploy Cross-cluster replication
- Take regular snapshots
