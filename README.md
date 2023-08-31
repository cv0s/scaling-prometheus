# Scaling Service with Prometheus Metrics
If you work with Kubernetes for a longer time, you will sooner or later have the usecase to scale with custom metrics. 
For example, an application may need to scale due to open sockets. 

With the help of prometheus and the prometehus adapter it is possible to scale on all metrics you provide

# Install prometheus
First we need to install prometheus 
If there is already a prometheus instance in your Kubernetes cluster, you can use that. Otherwise you can use the prometheus.yaml

```
k apply -f prometheus/prometheus.yaml
k get po 
```

check if the prometheus pod works
```
❯ k get po
NAME                          READY   STATUS    RESTARTS   AGE
prometheus-5986d45b6b-cl8tz   1/1     Running   0          24s
```

# Install example application

```
#set local docker registry
eval $(minikube docker-env)

#build local docker image
docker build -t go:1 -f simple-go/dockerfile simple-go

k apply -f simple-go/deployment.yaml
k apply -f simple-go/service.yaml
```

```
k port-forward svc/go-service 2112
curl localhost:2112/metrics
```

By default, the scarping of metrics endpoints from prometheus works with annotations. So you have to define if prometheus should scarp a pod, to which port it should be scarped and to which path. 

Here is an example from simle-go
``` 
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "2112"
        prometheus.io/scrape: "true"
```

# Install prometheus adapter

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install custom-metrics prometheus-community/prometheus-adapter
```

After a minute you should be able to this command: 
```
kubectl get --raw /apis/custom.metrics.k8s.io/
```

The output should be something similar to this 
```
{"kind":"APIResourceList","apiVersion":"v1","groupVersion":"custom.metrics.k8s.io/v1beta1","resources":[]}
```
The prometheus adapter created a new Apiservice.

```
spec:
  group: custom.metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: custom-metrics-prometheus-adapter
    namespace: default
    port: 443
  version: v1beta1
  versionPriority: 100
```

To create a new custom metric we need to create a new rule. 
```
k apply -f prometheus-adapter/adapter-configmap.yaml
k rollout restart deployment custom-metrics-prometheus-adapter
```

Now we can see our custom metrics
``` 
k get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "custom.metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "pods/promhttp_metric_handler_requests_per_second",
      "singularName": "",
      "namespaced": true,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
    {
      "name": "namespaces/promhttp_metric_handler_requests_per_second",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    }
  ]
}
```

# Start scaling
the last step is to create a hpa with a custom metric

```
k apply -f simple-go/hpa.yaml
 
k get horizontalpodautoscalers.autoscaling
NAME        REFERENCE              TARGETS      MINPODS   MAXPODS   REPLICAS   AGE
go-server   Deployment/go-server   1203m/500m   1         3         1          16s

k get pod
go-server-7c94d659d5-2jrbp                           0/1     Running   0          20s
go-server-7c94d659d5-87ndp                           1/1     Running   0          14m
go-server-7c94d659d5-cqkb4                           0/1     Running   0          20s
```