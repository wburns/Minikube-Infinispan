# Minikube-Infinispan

## Some commons commands to know

`kubectl get <type> -n <namespace>` - lists all resources of the given type in the given namespace (e.g. kubectl get pods -n infinispan to delete a pod - will probably be restarted)
`kubectl delete <type> <name> -n <namespace>` deletes a named resource of a given type in the given namespace (e.g. kubectl delete Infinispan -n infinispan to delete a cluster)
`kubectl get pods -A` shows all pods in all namespaces. Note --all-namespaces can be used for any type

Ip to be aware of

`minikube ip` returns the ip you can access any externally available resources


Install minikube (I did this from package manager version 1.16.0, but any should really work)


## configure minikube (maybe optional) - Only needed once
`make minikube-config` in the https://github.com/infinispan/infinispan-operator directory

## Start the actual minikube (needed upon restart)
`minikube start`

## install olm which is required for Subscription CRs to install operators
`minikube addons enable olm`

I sometimes had issues with olm breaking in my version. I found killing the pod the operatorhubio-catalog pod that is stuck in CrashLoopBackOff would normally fix

## Install Infinispan Operator
https://operatorhub.io/operator/infinispan

Can do `kubectl create -f https://operatorhub.io/install/infinispan.yaml` to install
or 
`curl https://operatorhub.io/install/infinispan.yaml > infinispan.yaml` and modify if desired (say change namespace if desired)
`kubectl apply -f infinispan.yaml` to install if downloaded file

## Install the actual Infinispan cluster
Can copy the yaml from operator hub. Details on the various configuration values can be found at https://infinispan.org/docs/infinispan-operator/2.0.x/operator.html#start_operator

I highly recommend at least installing with a nodePort to expose the console to your browser
```yaml
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: infinispan-cluster
  namespace: infinispan
spec:
  replicas: 3
  expose:
    type: NodePort
    nodePort: 30000
```

You also will need to create an infinispan namespace. You can also do this via yaml if desired.

`kubectl create namespace infinispan`

Creates a cluster of 3 nodes in the infinispan namespace exposed at port 30001

`kubectl apply -f infinispan-cluster.yaml`

Note that depending upon the version the Infinispan operator may not be monitoring other namespaces. I need to figure out how to configure this with the operator hub Subscription CR still.

The desired number of pods should eventually come up

`kubectl get pods -n infinispan -w` to monitor it over time

# Obtain the login information for the Infinispan cluster
`kubectl get secret -n infinispan` to find the secret name. It should be <name>-generated-secret where name was the name of the infinispan cluster above

`kubectl get secret infinispan-cluster-generated-secret -n infinispan -o jsonpath="{.data.identities\.yaml}" | base64 -d`

You should be able to log into the Infinispan console at `http://<minikube ip>:30000` and create cache and/or add data

You can optionally define the secret in the `infinispan-cluster.yaml` above

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: infinispan-basic-auth
  namespace: infinispan
stringData:
  identities.yaml: |- 
    credentials:
    - username: developer
      password: dIRs5cAAsHIeeRIL
    - username: operator
      password: uMBo9CmEdEduYk24
---
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: infinispan-cluster
  namespace: infinispan
spec:
  replicas: 3
  expose:
    type: NodePort
    nodePort: 30000
  security:
    endpointSecretName: infinispan-basic-auth
```

## Cluster service information
More details about what can be configured in the Infinispan service can be found at https://infinispan.org/docs/infinispan-operator/2.0.x/operator.html#ref_datagrid_service_crd-nodes

## Add Prometheus using operator hub
https://operatorhub.io/operator/prometheus

`kubectl create -f https://operatorhub.io/install/prometheus.yaml` can be used to install the latest from operator hub

## Install Prometheus CR
You can take an example from operator hub

```prometheus-cr.yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: infinispan
  namespace: operators
spec:
  alerting: 
    alertmanagers: []
  replicas: 1
  ruleSelector: {}
  securityContext: {}
  serviceAccountName: prometheus-k8s
  serviceMonitorSelector: {}
```

If for some reason it failed, try to delete the pod `kubectl delete pod prometheus-operator-<RANDOM_NAME> -n operators` and rerun the command above 

## Add a service monitor to monitor the Infinispan nodes

Make sure the secret matches the login for the Infinispan secret.

```prometheus-service-monitor.yaml
apiVersion: v1
stringData:
  username: developer
  password: dZ7S575KLkIZ9PYk
kind: Secret
metadata:
  name: prom-infinispan-basic-auth
  namespace: operators
type: Opaque
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: prometheus
  name: infinispan-grafana-monitoring
  namespace: operators
spec:
  endpoints:
    - basicAuth:
        password:
          key: password
          name: prom-infinispan-basic-auth
        username:
          key: username
          name: prom-infinispan-basic-auth
      interval: 30s
      scheme: http
      targetPort: 11222
      path: /metrics
  namespaceSelector:
    matchNames:
      - infinispan
  selector:
    matchLabels:
      app: infinispan-service
      clusterName: infinispan-cluster
```

## setup a nodeport to expose prometheus so we can view its console

You can use the prometheus-nodeport.yaml file

`kubectl apply -f prometheus-nodeport.yaml`

Note that it must be in the same namespace and the selector must match, in this case the prometheus service has a label for app that is prometheus

This should allow accessing the Prometheus console at `http://<minikube-ip>:30900` and you can verify that the service monitor is working by checking Targets under Status.

## Install the grafana operator
https://operatorhub.io/operator/grafana-operator

`kubectl create -f https://operatorhub.io/install/grafana-operator.yaml` can be used to install the latest from operator hub

I personally used the grafana-operator.yaml file

`kubectl apply -f grafana-operator.yaml`

## Install the Grafana CR
You can take an example from operator hub

I used grafana-cr.yaml, note the user and password to login to the console

```
kubectl create namespace grafana
kubectl apply -f grafana-cr.yaml
```

## Apply the NodePort for grafana to expose it
See grafana-nodeport.yaml and add via `kubectl apply -f grafana-nodeport.yaml`

The grafana console should now be accesible via `http://<minikube-ip>:30901`
The login for the security user and password in the previous step can be used to log in

It can take some time to the `http://<minikube-ip>:30901` be available. 

## setup grafana
We now need to add a datasource for Grafana under Configuration. Don't forget to login.

Add a Prometheus data source

For the URL I just put the exposed minikube ip and port from the node. Technically you can use the service exposed ip to the kubernetes cluster as well.

## Add your dashboard
Infinispan will eventually install it automatically no matter the order, but that isn't in yet.
You can import `dashboard.json`

### Troubleshooting Dashboard
If the values are not reported to the Dashboard, try hit the configuration dashboad button and submit the same in the variables form

