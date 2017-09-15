## Using keepalived as kubernetes cloud provider and google as DNS provider

We'll show how to setup the keepalived as cloud provider in an on-premises cluster. The main motivation to use it, is make possible create external and public LoadBalancers in kubernetes on-premises clusters as is possible in Gogole, Azure, etc.

This tutorial assumes you have:

* A running on-premises kubernetes cluster. If you don't, you can follow [These tutorial](../k8s-on-prem)
* SSH access to the kubernetes master


## Hands-on

### Deploying the keepalived and keepalived-vip

The default mode of cloud providers in kubernetes is `external`, then to use the keepalived cloud provider. Then at this point you just need create the keepalived cloud provider deployment using the following yaml(`keepalived-deploy.yaml`)

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  labels:
    app: keepalived-cloud-provider
  name: keepalived-cloud-provider
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: keepalived-cloud-provider
  strategy:
    type: RollingUpdate
  template:
    metadata:
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
        scheduler.alpha.kubernetes.io/tolerations: '[{"key":"CriticalAddonsOnly", "operator":"Exists"}]'
      labels:
        app: keepalived-cloud-provider
    spec:
      serviceAccount: keepalived
      containers:
      - name: keepalived-cloud-provider
        image: quay.io/munnerz/keepalived-cloud-provider:0.0.1
        imagePullPolicy: IfNotPresent
        env:
        - name: KEEPALIVED_NAMESPACE
          value: kube-system
        - name: KEEPALIVED_CONFIG_MAP
          value: vip-configmap
        - name: KEEPALIVED_SERVICE_CIDR
          value: 105.156.51.69/27 # Pick a CIDR for your services
        volumeMounts:
        - name: certs
          mountPath: /etc/ssl/certs
        resources:
          requests:
            cpu: 200m
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10252
            host: 127.0.0.1
          initialDelaySeconds: 15
          timeoutSeconds: 15
          failureThreshold: 8
      volumes:
      - name: certs
        hostPath:
          path: /etc/ssl/certs
```


`$ kubectl create -f keepalived-deploy.yaml`

>Important: If you want public services, you should use a routable subnet in your CIDR.


Now we'll create the config map which will be used to manage the services IPs(Virtual IPs) using the yaml(`keepalived-cm.yaml`) below

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vip-configmap
  namespace: kube-system
data:
```

`$ kubectl create -f keepalived-cm.yaml`

Executing only the two steps before, we will be able to create services. For example, supose you have a deployment named `hello-world` which has a label `app=hello-world` you should expose the `hellow-world` 

`$ kubectl expose deployment hello-world --type=LoadBalancer --name=my-service`

Then we can check if the service got an external IP

```bash
$ kubectl get svc 
NAME         CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
kubernetes   10.96.0.1       <none>           443/TCP        13d
my-service   10.96.197.241   105.156.51.70   80:31952/TCP   5s
```


