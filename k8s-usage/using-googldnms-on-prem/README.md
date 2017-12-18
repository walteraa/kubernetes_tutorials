# Service Discovery in an on-premises cluster using Google DNS as DNS provider(Using workaround)

In this tutorial we intent to explain how to deploy an on-premises Kubernetes Federation using Google DNS.

## Prerequisistes

For this tutorial, we are assuming you have:
* At least one on-premises cluster running
* At least one Node running as worker(It can be the master running in taint mode)
* Google DNS enabled and a zone configured, in our case we used federation.walteralves.me as zone
* A subdomain registered in a DNS registrar(e.g.: Godaddy), in this tutorial I'm using federation.walteralves.me
* Configured your subdomain(e.g.: federation.walteralves.me) to use the Google DNS zone NS


## Setting up an on-premises federation

```bash
$ubuntu@walter-master:~$ kubefed init kfed \
    --host-cluster-context=on-prem \
    --api-server-service-type=NodePort \
    --api-server-advertise-address=10.11.4.152 \
    --apiserver-enable-token-auth="true" \
    --dns-provider=google-clouddns \
    --dns-zone-name=federation.walteralves.me. \
    --etcd-persistent-storage="false"\
    --kubeconfig=$HOME/.kube/config
Creating a namespace federation-system for federation system components... done
Creating federation control plane service... done
Creating federation control plane objects (credentials, persistent volume claim)... done
Creating federation component deployments... done
Updating kubeconfig... done
Waiting for federation control plane to come up..................................... done
Federation API server is running at: 10.11.4.152:31641
```

Now our federation control plane was created but isn't properly running

```bash
ubuntu@walter-master:~$ kubectl get pods --all-namespaces
NAMESPACE           NAME                                        READY     STATUS             RESTARTS   AGE
default             microbot-3325520198-1qc4d                   1/1       Running            0          4d
default             microbot-3325520198-brvr9                   1/1       Running            0          4d
default             microbot-3325520198-g5s6r                   1/1       Running            0          4d
federation-system   kfed-apiserver-3067167381-4b6nd             2/2       Running            0          10m
federation-system   kfed-controller-manager-3784998721-ftc8f    0/1       CrashLoopBackOff   6          10m
kube-system         calico-etcd-l1zqz                           1/1       Running            0          4d
kube-system         calico-node-nrgws                           2/2       Running            0          4d
kube-system         calico-node-zcr6g                           2/2       Running            0          4d
kube-system         calico-policy-controller-1727037546-k2cf8   1/1       Running            0          4d
kube-system         etcd-walter-master                          1/1       Running            0          4d
kube-system         kube-apiserver-walter-master                1/1       Running            0          4d
kube-system         kube-controller-manager-walter-master       1/1       Running            0          4d
kube-system         kube-dns-2425271678-bs23v                   3/3       Running            0          4d
kube-system         kube-proxy-1rqwr                            1/1       Running            0          4d
kube-system         kube-proxy-grpkh                            1/1       Running            0          4d
kube-system         kube-scheduler-walter-master                1/1       Running            0          4d
```

Seing federation-controller-manager log, we can conclude that there's a DNS authorization issue

```bash
ubuntu@walter-master:~$ kubectl logs -n federation-system   kfed-controller-manager-3784998721-ftc8f
I1003 13:17:59.761074       1 controllermanager.go:93] v1.8.0
I1003 13:17:59.790089       1 clustercontroller.go:63] Starting cluster controller
I1003 13:18:00.735677       1 clouddns.go:86] Using DefaultTokenSource <nil>
E1003 13:18:00.736199       1 dns.go:83] DNS provider could not be initialized: could not init DNS provider "google-clouddns": google: could not find default credentials. See https://developers.google.com/accounts/docs/application-default-credentials for more information.
F1003 13:18:00.736312       1 controllermanager.go:208] Failed to start service dns controller: could not init DNS provider "google-clouddns": google: could not find default credentials. See https://developers.google.com/accounts/docs/application-default-credentials for more information.
```


### Fixing the credential issue

To fix this issue, we sould attribute an environment variable named `GOOGLE_AUTHORIZATION` in the federation-controller-manager which the value will be the Google DNS credentials file.

#### Create Google credential

Go to the Google cloud credential following this steps:

* Go to the link https://console.developers.google.com/apis/credentials?project=\${PROJECT_ID};
* Create a new credential by selecting Service Account key
  * Select JSON as key type
  * New Service account
  * Add a name to the service account
  * Select DNS/DNS Administrator as role
* Download the file generated


### Creating a secret using the Google DNS credential file

```bash
ubuntu@walter-master:~$ kubectl create secret -n federation-system generic dns-credential --from-file=./dns-credential.json
secret "dns-credential" created
```

Now we are able to mount the secret in the federation-controller-manager pod/container by editing its deployment spec

```bash
ubuntu@walter-master:~$ kubectl edit deploy -n federation-system   kfed-controller-manager
```

```yaml
spec:
  replicas: 0
  selector:
    matchLabels:
      app: federated-cluster
      module: federation-controller-manager
      pod-template-hash: "3784998721"
  template:
    metadata:
      annotations:
        federation.alpha.kubernetes.io/federation-name: kfed
      creationTimestamp: null
      labels:
        app: federated-cluster
        module: federation-controller-manager
        pod-template-hash: "3784998721"
      name: kfed-controller-manager
    spec:
      containers:
      - command:
        - /hyperkube
        - federation-controller-manager
        - --dns-provider=google-clouddns
        - --federation-name=kfed
        - --kubeconfig=/etc/federation/controller-manager/kubeconfig
        - --master=https://kfed-apiserver
        - --zone-name=federation.walteralves.me.
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: GOOGLE_APPLICATION_CREDENTIALS  #Attributing the absolute path of the credential file to Google's credentials variable
          value: /etc/federation/google/dns-credential.json
	  - name: PROJECT_ID  #Project ID environment variable
          value: corc-tutorial
        image: gcr.io/google_container/hyperkube-amd64:v1.8.0
        imagePullPolicy: IfNotPresent
        name: controller-manager
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/federation/controller-manager
          name: kfed-controller-manager-kubeconfig
          readOnly: true
        - mountPath: /etc/federation/google #Mounting the volume created below
          name: dns-credential
          readOnly: true
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: federation-controller-manager
      serviceAccountName: federation-controller-manager
      terminationGracePeriodSeconds: 30
      volumes:
      - name: kfed-controller-manager-kubeconfig
        secret:
          defaultMode: 420
          secretName: kfed-controller-manager-kubeconfig
      - name: dns-credential #volume using previous created secret
        secret:
          defaultMode: 420
          secretName: dns-credential
status:
  observedGeneration: 4
  replicas: 0
```

Explaining: We used the `dns-credential` secret to create a volume with the same name, then we mount this volume in the `/etc/federation/google` path and after that, the federation-controller-manager container will has a credential file `/etc/federation/google/dns-credential.json`. So, now we are able to set the credential file path to `GOOGLE_APPLICATION_CREDENTIALS` environment variable, which will be used by the Google DNS provider to authenticate in the Google Cloud API.

> Important: At the moment I'm writing this, there is a bug in the Google DNS provider(which is registered [here](https://github.com/kubernetes/kubernetes/issues/52726)) which the Project ID isn't get and keeping empty, then the DNS informations couldn't be retrieved and manipulated by the federation-controller-manager. The step below should be followed only if you are getting this issue.

### "Another bug to the bow"

After we did the configurations above, we found a error that made it difficult to use Google DNS as DNS provider. For some reason, the credentials error has turned in a 404 error. It seems like the DNS provider doesn't know what URL use to get DNS information from Google DNS. Now we will talk about this issue and how I did a workaround to fix it parcially.

#### How we debbug it and partially fix it(through a workaround)
When we figured out to use the Google DNS credentials file another bug happened: 404 error. It was a bug that takes a long time to be debuged: Why the DNS service was getting 404 error if all Google credentials information were provided in the credentials file? Well, I have figured out that the `project_id` property wasn't used in the Google DNS provider. The value in this property should be attributed to a variable named `projectID`. 

Then I put a log in the file `pkg/dnsprovider/providers/google/clouddns/clouddns.go` to figure out if `projectID` is being properly setted

```go
// newCloudDns creates a new instance of a Google Cloud DNS Interface.
func newCloudDns(config io.Reader) (*Interface, error) {
	projectID, _ := metadata.ProjectID() // On error we get an empty string, which is fine for now.
  glog.Info("projectID = ",projectID)

	var tokenSource oauth2.TokenSource
	// Possibly override defaults with config below
	if config != nil {
		var cfg Config
		if err := gcfg.ReadInto(&cfg, config); err != nil {
			glog.Errorf("Couldn't read config: %v", err)
			return nil, err
		}
		glog.Infof("Using Google Cloud DNS provider config %+v", cfg)
		if cfg.Global.ProjectID != "" {
			projectID = cfg.Global.ProjectID
		}
		if cfg.Global.TokenURL != "" {
			tokenSource = gce.NewAltTokenSource(cfg.Global.TokenURL, cfg.Global.TokenBody)
		}
	}
	return CreateInterface(projectID, tokenSource)
}
```

I was surprised that `projectID` is empty, once this variable is used to create a Cloud DNS provider instance and consequently to perform queries to the DNS endpoint properly. Then I did a workaround which does the value of `projectID` being the same value of `PROJECT_ID` environment variable, which is the project name that I have created the `federation.walteralves.me` zone and is defined in the yaml above, this project name is the same which has the zone that the kubernetes federation will manage the DNS entries for Service Discovery

```go
// newCloudDns creates a new instance of a Google Cloud DNS Interface.
func newCloudDns(config io.Reader) (*Interface, error) {
	//projectID, _ := metadata.ProjectID() // On error we get an empty string, which is fine for now.
  projectID := os.Getenv("PROJECT_ID")
  glog.Info("projectID = ",projectID)

	var tokenSource oauth2.TokenSource
	// Possibly override defaults with config below
	if config != nil {
		var cfg Config
		if err := gcfg.ReadInto(&cfg, config); err != nil {
			glog.Errorf("Couldn't read config: %v", err)
			return nil, err
		}
		glog.Infof("Using Google Cloud DNS provider config %+v", cfg)
		if cfg.Global.ProjectID != "" {
			projectID = cfg.Global.ProjectID
		}
		if cfg.Global.TokenURL != "" {
			tokenSource = gce.NewAltTokenSource(cfg.Global.TokenURL, cfg.Global.TokenBody)
		}
	}
	return CreateInterface(projectID, tokenSource)
}
```


After do that, we should follow the steps below:
* Generate a new `federation-controller-manager` binary using using the command `build/run.sh & build/run.sh federation-controller-manager` 
* Follow [this guide](https://github.com/kubernetes/kubernetes/tree/master/cluster/images/hyperkube) to generate a new hyperkube image
> In this step you should set the REGISTRY environment variable to be your container repository service, in our case REGISTRY=docker.io/walteraa.
> Choose a VERSION of you want, we choose v1.8.0-example as VERSION
* Once you set variables above, do login in the dockerhub in your terminal and push the image to your repository
* In the `federation-controller-manager` deployment, edit the `image` option to use your new hyperkube version. E. g.:  `image: docker.io/walteraa/hyperkube-amd64:v1.8.0-example`  instead of `image: gcr.io/google_container/hyperkube-amd64:v1.8.0`
* Now you can delete the `federation-controller-manager` pod and your `federation-controller-manager` version will be running.


We have debugged how the Google DNS provider get the credentials and I figured out that the file `vendor/golang.org/x/oath2goole/default.go` is responsable for manipulate the  `GOOGLE_APPLICATION_CREDENTIALS` file and get all values inside this file through the function `FindDefaultCredentials` specially using the following piece of code:

```go
// First, try the environment variable.
	const envVar = "GOOGLE_APPLICATION_CREDENTIALS"
	if filename := os.Getenv(envVar); filename != "" {
		creds, err := readCredentialsFile(ctx, filename, scope)
		if err != nil {
			return nil, fmt.Errorf("google: error getting credentials using %v environment variable: %v", envVar, err)
		}
    glog.Info("default.go:creds.ProjectID = ",creds.ProjectID)
		return creds, nil
	}
```

However, at the moment I am writing this tutorial, I didn't figure out how to fix it.
