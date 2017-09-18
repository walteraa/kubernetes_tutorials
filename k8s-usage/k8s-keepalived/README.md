## Using keepalived as kubernetes cloud provider on an on-premises cluster

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
my-service   10.96.197.241   105.156.51.70    80:31952/TCP   5s
```

The keepalived-vip is the component which manages the network interface adding virtual IPs in the interface making possible access the services from computers out of kubernetes cluster, so is necessary deployt it in your on-premises cluster using the following yaml(`keepalived-vip-ds.yaml`)


```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-keepalived-vip
  namespace: kube-system
spec:
  template:
    metadata:
      labels:
        name: kube-keepalived-vip
    spec:
      hostNetwork: true
      serviceAccount: keepalived
      containers:
        - image: docker.io/walteraa/kube-keepalived-vip:v0.0.2
#        - image: gcr.io/google_containers/kube-keepalived-vip:0.9
          name: kube-keepalived-vip
          imagePullPolicy: Always
          securityContext:
            privileged: true
          volumeMounts:
            - mountPath: /lib/modules
              name: modules
              readOnly: true
            - mountPath: /dev
              name: dev
          # use downward API
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          # to use unicast
          args:
          - --services-configmap=kube-system/vip-configmap
#          - --proxy-protocol-mode=true
          # unicast uses the ip of the nodes instead of multicast
          # this is useful if running in cloud providers (like AWS)
          #- --use-unicast=true
      volumes:
        - name: modules
          hostPath:
            path: /lib/modules
        - name: dev
          hostPath:
            path: /dev
      nodeSelector:
        type: worker #Label that you have crated
```

> Important: The node which will run the DaemonSet should receive the label `worker`

Now the worker node's network interface is able to be managed by the keepalived-vip.


> Important: At the moment I'm writing this, the keepalived-cloud-provider is crashing because a health check failure which is described by [this issue](https://github.com/munnerz/keepalived-cloud-provider/issues/2), if it happens with you, delete the pod and it will be up again.

### Deploying the federation control plane in the cluster using Google DNS provider

This step is important only if you want to manage the federation through an on-premises cluster. If you only want to run the on-premises cluster in a federation which has the FCP in a cloud provided cluster(e.g.: GKE, Azure, etc...) just skip this topic.

This step assumes you have:

* The `kubefed` configured in your local computer. If you don't, you can follow [these tutorial](..#kubernetes-federation-administratorkubefed)
* A Google cloud account which has Google DNS API enabled
* A public domain provided by a domain registrar(e.g.: godaddy)


Here we will be using the domain `walteralves.me` registered in [GoDaddy](https://br.godaddy.com/) and a DNS zone created in the Google DNS named `federation` with `federation.walteralves.me.` as DNS name. We should add the `federation` subdomain entries in Godaddy using the NS entries generated in the `federation` DNS zone in Google.

Google entries
![](images/google_dns.png?raw=true)

Godaddy Entries
![](images/godaddy.png?raw=true)


