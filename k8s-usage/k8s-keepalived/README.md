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

Executing only the two steps before, we will be able to create services. For example, supose you have a deployment named `hello-world` which has a label `app=hello-world` you should expose the `hello-world` 

`$ kubectl expose deployment hello-world --type=LoadBalancer --name=my-service`

Then we can check if the service got an external IP

```bash
$ kubectl get svc 
NAME         CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
kubernetes   10.96.0.1       <none>           443/TCP        13d
my-service   10.96.197.241   105.156.51.70    80:31952/TCP   5s
```

The keepalived-vip is the component which manages the network interface adding virtual IPs in the interface making possible access the services from computers out of kubernetes cluster, so is necessary deployt it in your on-premises cluster using the following yaml(`keepalived-vip-ds.yaml`)


```yaml
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

### Exposing the service to external world

To expose our services to external world, we need that CIDR used in the keepalived cloud provideri's configuration to be routable from real world through a router and the worker node as well, as showed in the image below

![On-premises infrastructure example](images/on-premises_infra.png?raw=true)

In details, the keepalived process will be

* The keepalived cloud provider will creates services using the API server and the routable CIDR passed in the configuration
* The keepalived cloud provider will manipulate the configmap registering the services created
* The keepalived-VVIP will read the services informations through the API server
* The keepalived-VIP will use the services informations to generate a configuration file used for the keepalivedvip software running in the container
* The keepalivedvip software will use the configuratio file to manage the worker node's network interface


![Diagram of keepalived](images/keepalived_diafgram.png?raw=true)

Once we have this infrastructure presented before, The services created through the keepalived cloud provider can be reached through the real world.

> Important: The firewall rules should permit access to the CIDR(network used by the services) network through the ports the services are exposed.

#### Creating services in an on-premises cluster through keepalived

Considering that all our setup was made successfuly, now we are able to expose deployments to real world. Supose that we have the same `hello-world` application mentioned before, but now we have all the keepalived infrastructure deployed in our kubernetes on-premises cluster. Expose it as load balancer as showed before


`$ kubectl expose deployment hello-world --type=LoadBalancer --name=my-service2`

Then, we will have the service with the external IP in the CIDR range

```bash
$ kubectl get svc 
NAME         CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
kubernetes   10.96.0.1       <none>           443/TCP        13d
my-service   10.96.197.241   105.156.51.70    80:31952/TCP   10m
my-service2  10.96.197.241   105.156.51.71    80:31953/TCP   5s
```

Once we have the infrastructure as described before(Routeble CIDR and firewall rules configured), we will be able to perform a curl to the service's external IP. Once we have exposed you deploy sucessfully, we can see the configuration file genereted by the keepalived-vip controller perform the following command

`kubectl exec --context=on-premises -it -n kube-system  ${KEEPALIVEDVIP_POD_NAME} -- cat /etc/keepalived/keepalived.conf`

Then, we can see something like that

```conf
global_defs {
  vrrp_version 3
  vrrp_iptables KUBE-KEEPALIVED-VIP
}



vrrp_instance vips {
  state BACKUP
  interface eth0
  virtual_router_id 50
  priority 100
  nopreempt
  advert_int 1

  track_interface {
    eth0
  }



  virtual_ipaddress {
    105.156.51.70
    105.156.51.71
  }



}



# Service: app-my-service
virtual_server  105.156.51.70 80 {
  delay_loop 5
  lvs_sched rr
  lvs_method NAT
  persistence_timeout 1800
  protocol TCP


  real_server 192.168.104.57 80 {
    weight 1
    TCP_CHECK {
      connect_port 80
      connect_timeout 3
    }
  }

}

# Service: app-my-service
virtual_server  105.156.51.70 80 {
  delay_loop 5
  lvs_sched rr
  lvs_method NAT
  persistence_timeout 1800
  protocol TCP


  real_server 192.168.104.57 80 {
    weight 1
    TCP_CHECK {
      connect_port 80
      connect_timeout 3
    }
  }

}
```

As mantioned before, the keepalived-vip software manages the network interface, and we can see it showing the `eth0` interface's IP addresses in the worker node

```bash
walteraa@node2:~$ sudo ip addr list eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:26:b9:51:bf:79 brd ff:ff:ff:ff:ff:ff
    inet 105.156.51.40/27 brd 105.156.51.59 scope global eth0
       valid_lft forever preferred_lft forever
    inet 105.156.51.70/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet 105.156.51.71/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::226:b9ab:fe15:ba97/64 scope link
       valid_lft forever preferred_lft forever
```

As we can see in the command above the IP of services, which were added in the `virtual_ipaddress` in the configuration file, were added as virtual IPs in the worker node's network interface. It makes possible that the router knows what interface respond to the services IPs through ARP(I'll not detail these process).

### Deploying the federation control plane in the cluster using Google DNS provider

> TODO: Remove it from here and add it in another tutorial about "Federation with FCP in a on-premises cluster using keepalived and Google DNS"

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


