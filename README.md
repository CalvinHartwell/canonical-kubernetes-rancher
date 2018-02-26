# cdk-rancher

This repository explains how to deploy Rancher 2.0alpha on Canonical Kubernetes. 

These steps are currently in alpha/testing phase and will most likely change. 

## Deploy Canonical Kubernetes

Deploying Canonical Kubernetes is really easy, I assume you are running on an Ubuntu machine and you have an AWS account.  Grab a new API key from AWS and put that into:

```
~/.aws/credentials 
```

In this format:

```
[default]
aws_access_key_id=<access_id>
aws_secret_access_key=<secret_key>
```

Next install juju on your machine, we will use the snap:

```
sudo apt-get install snap
snap install juju
```

Next we bootstrap juju so it's ready to use aws:

```
juju bootstrap
```

We follow the steps here for bootstrapping our cloud for AWS: 

```
calvinh@ubuntu-ws:~/.aws$ juju bootstrap
Clouds
aws
aws-china
aws-gov
azure
azure-china
cloudsigma
google
joyent
localhost
oracle
rackspace
salesmaas

Select a cloud [localhost]: aws

Regions in aws:
ap-northeast-1
ap-northeast-2
ap-south-1
ap-southeast-1
ap-southeast-2
ca-central-1
eu-central-1
eu-west-1
eu-west-2
sa-east-1
us-east-1
us-east-2
us-west-1
us-west-2

Select a region in aws [us-east-1]: eu-west-1

Enter a name for the Controller [aws-eu-west-1]: rancher-k8s

Creating Juju controller "rancher-k8s" on aws/eu-west-1
Looking for packaged Juju agent version 2.3.2 for amd64
Launching controller instance(s) on aws/eu-west-1...
 - i-08ce69142f943b5a4 (arch=amd64 mem=4G cores=2)eu-west-1a"
Installing Juju agent on bootstrap instance
Fetching Juju GUI 2.11.3
Waiting for address
Attempting to connect to 172.31.20.176:22
Attempting to connect to 34.244.155.220:22
Connected to 34.244.155.220
Running machine configuration script...


Bootstrap agent now started
Contacting Juju controller at 34.244.155.220 to verify accessibility...
Bootstrap complete, "rancher-k8s" controller now available
Controller machines are in the "controller" model
Initial model "default" added
```

Once the controller has been configured, we now deploy CDK:

```
calvinh@ubuntu-ws:~/.aws$ juju deploy canonical-kubernetes
Located bundle "cs:bundle/canonical-kubernetes-150"
Resolving charm: cs:~containers/easyrsa-27
Resolving charm: cs:~containers/etcd-63
Resolving charm: cs:~containers/flannel-40
Resolving charm: cs:~containers/kubeapi-load-balancer-43
Resolving charm: cs:~containers/kubernetes-master-78
Resolving charm: cs:~containers/kubernetes-worker-81
Executing changes:
- upload charm cs:~containers/easyrsa-27 for series xenial
- deploy application easyrsa on xenial using cs:~containers/easyrsa-27
  added resource easyrsa
- set annotations for easyrsa
- upload charm cs:~containers/etcd-63 for series xenial
- deploy application etcd on xenial using cs:~containers/etcd-63
  added resource etcd
  added resource snapshot
- set annotations for etcd
- upload charm cs:~containers/flannel-40 for series xenial
- deploy application flannel on xenial using cs:~containers/flannel-40
  added resource flannel-amd64
  added resource flannel-s390x
- set annotations for flannel
- upload charm cs:~containers/kubeapi-load-balancer-43 for series xenial
- deploy application kubeapi-load-balancer on xenial using cs:~containers/kubeapi-load-balancer-43
- expose kubeapi-load-balancer
- set annotations for kubeapi-load-balancer
- upload charm cs:~containers/kubernetes-master-78 for series xenial
- deploy application kubernetes-master on xenial using cs:~containers/kubernetes-master-78
  added resource cdk-addons
  added resource kube-apiserver
  added resource kube-controller-manager
  added resource kube-scheduler
  added resource kubectl
- set annotations for kubernetes-master
- upload charm cs:~containers/kubernetes-worker-81 for series xenial
- deploy application kubernetes-worker on xenial using cs:~containers/kubernetes-worker-81
  added resource cni-amd64
  added resource cni-s390x
  added resource kube-proxy
  added resource kubectl
  added resource kubelet
- expose kubernetes-worker
- set annotations for kubernetes-worker
- add relation kubernetes-master:kube-api-endpoint - kubeapi-load-balancer:apiserver
- add relation kubernetes-master:loadbalancer - kubeapi-load-balancer:loadbalancer
- add relation kubernetes-master:kube-control - kubernetes-worker:kube-control
- add relation kubernetes-master:certificates - easyrsa:client
- add relation etcd:certificates - easyrsa:client
- add relation kubernetes-master:etcd - etcd:db
- add relation kubernetes-worker:certificates - easyrsa:client
- add relation kubernetes-worker:kube-api-endpoint - kubeapi-load-balancer:website
- add relation kubeapi-load-balancer:certificates - easyrsa:client
- add relation flannel:etcd - etcd:db
- add relation flannel:cni - kubernetes-master:cni
- add relation flannel:cni - kubernetes-worker:cni
- add unit easyrsa/0 to new machine 0
- add unit etcd/0 to new machine 1
- add unit etcd/1 to new machine 2
- add unit etcd/2 to new machine 3
- add unit kubeapi-load-balancer/0 to new machine 4
- add unit kubernetes-master/0 to new machine 5
- add unit kubernetes-worker/0 to new machine 6
- add unit kubernetes-worker/1 to new machine 7
- add unit kubernetes-worker/2 to new machine 8
Deploy of bundle completed.
```

You can check the deployment status using the following command: 

```
 watch --color juju status --color
```

Note that this will give you the default bundle for CDK which is made up of 9 machines, flannel networking and no RBAC. This is based on the default bundle found here: [https://jujucharms.com/canonical-kubernetes/](https://jujucharms.com/canonical-kubernetes/).

For a more tailored build with Canal or Calico, you can use the bundle builder: [https://github.com/juju-solutions/bundle-canonical-kubernetes](https://github.com/juju-solutions/bundle-canonical-kubernetes). This will generate a bundle file, which is just a big piece of yaml which describes the configuration for the entire cluster, similar to an Ansible Playbook or Puppet Manifest. 

If you have a custom bundle, you would deploy that using a command like this instead:

```
 juju deploy bundle.yaml
```

Eventually the colours will all turn green and your cluster is good to go. To access the cluster, we need to install the kubectl command line client and copy the kubernetes configuration file over for it to use: 

```
 snap install kubectl
```

Next we copy over the configuration file: 

```
  juju scp kubernetes-master/0:/home/ubuntu/config ~/.kube/config
```

Finally, using kubectl we can check that kubernetes cluster interaction is possible: 

```
Kubernetes master is running at https://34.253.164.197:443

Heapster is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/kube-dns/proxy
kubernetes-dashboard is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy
Grafana is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
InfluxDB is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/monitoring-influxdb/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'
```

## Deploying Rancher

To deploy Rancher, we just need to run the Rancher container workload on-top of Kubernetes. Rancher provides their containers through dockerhub ([https://hub.docker.com/r/rancher/server/tags/](https://hub.docker.com/r/rancher/server/tags/)) and can be downloaded freely from the internet. If you're running your own registry or have an offline deployment, the container should be downloaded and pushed to the private registry.  

### Deploying Rancher with a nodeport

The yaml file called cdk-rancher-nodeport.yaml inside this repository can be used to deploy Rancher with a nodeport for accessing the cluster. Once kubectl is running and working, run the following command to deploy Rancher: 

```
  kubectl apply -f cdk-rancher-nodeport.yaml
```

Now we need to open this nodeport so we can access it. For that, we can use juju. We need to run the open-port command for each of the worker nodes in our cluster. Inside the cdk-rancher-nodeport.yaml file, the nodeport has been set to 30443. Below shows how to open the port on each of the worker nodes:

```
   # repeat this for each kubernetes worker in the cluster. 
   juju run --unit kubernetes-worker/0 "open-port 30443"
   juju run --unit kubernetes-worker/1 "open-port 30443"
   juju run --unit kubernetes-worker/2 "open-port 30443"
```

Rancher can now be accessed on this port through a worker IP or DNS entries if you have created them. Try opening it up in your browser:  

```
  # replace the IP address with one of your Kubernetes worker, find this from juju status command. 
  wget https://35.178.130.245.xip.io:30443 --no-check-certificate
```

If you need to make any changes to the kubernetes configuration file, edit the yaml file and then just use apply again: 

```
  kubectl apply -f cdk-rancher-nodeport.yaml
```

### Deploying Rancher with an ingress rule

*Please note: The ingress rule example here is currently not working but is being worked on.

The cdk-rancher-ingress.yaml yaml file contains an example Kubernetes configuration for deploying Rancher with an ingress rule. If you use an ingress rule, you don't need to use the nodeport or open additional ports using juju. 

However, you do need to modify the example bundle to change the hostname the ingress rule is using: 

```
  cat cdk-rancher-ingress.yaml | grep xip.io
  - host: rancher.35.178.130.245.xip.io
```

The hostname in the example should be changed to reflect a resolvable DNS entry for your workers. You can use the xip.io service for testing, just change the IP address in the entry to reflect the IP address of one of your workers. Next apply the kubernetes work load:

```
 kubectl apply -f cdk-rancher-ingress.yaml
```

Rancher can now be accessed on the regular 443 through a worker IP or DNS entries if you have created them. Try opening it up in your browser:

```
  # replace the IP address with one of your Kubernetes worker, find this from juju status command.
  wget https://35.178.130.245.xip.io:443 --no-check-certificate
```

If you need to make any changes to the kubernetes configuration file, edit the yaml file and then just use apply again:

```
  kubectl apply -f cdk-rancher-ingress.yaml
```

### Removing Rancher

If you wish to remove rancher from the cluster, we can do it using kubectl. Deleting constructs in Kubernetes is as simple as creating them: 

```
  # If you used the nodeport example change the yaml filename if you used the ingress example. 
  kubectl delete -f cdk-rancher-nodeport.yaml
```

## Using Rancher

Rancher can be accessed using it's web interface but a CLI client is planned in the future. 

The best resource for using Rancher is the official manual found here: [http://rancher.com/docs/rancher/v2.0/en/quick-start-guide/](http://rancher.com/docs/rancher/v2.0/en/quick-start-guide/). 

Hit the Rancher IP in your browser, you should see a login prompt: 

![rancher login page](https://raw.githubusercontent.com/CalvinHartwell/cdk-rancher/master/images/rancher-login.png "Rancher Web GUI Login Page") 

The default username is admin and password is admin. You should be prompted to change them, it is recommended you use a strong password. 

Once you've set the password, you should see a GUI like this: 

![rancher gui page](https://raw.githubusercontent.com/CalvinHartwell/cdk-rancher/master/images/rancher-gui.png "Rancher Web GUI Page")

### The Rancher GUI

The Rancher GUI has three main views, each of which has its own set of sub-menus.  

![rancher gui cluster page](https://raw.githubusercontent.com/CalvinHartwell/cdk-rancher/master/images/rancher-gui-cluster.png "Rancher Web GUI Cluster")

These are the three main views which contain a set of sub-menus: 
- Global View: Provides a view of each of the Kubernetes clusters under Rancher's control. This provides a break down of Nodes, Node Drivers, Catalogs, Users and Security Settings.   
- Cluster: This provides a view of of an individual Kubernetes cluster. It allows you to download the kubeconfig file for a cluster, check basic cluster stats, namespaces, projects, nodes and users for a particular cluster. 
- Namespace: This provides a view for an individual Namespace defined on a Kubernetes cluster. It allows you to configure workloads, DNS entries, ingress rules, volumes, secrets, registries, certificates and most importantly, the catalog. 

### Deploying a Workload with Rancher



### Adding another Cluster to Rancher

## Conclusion

This documentation has explained how to configure and deploy Canonical Kubernetes with Rancher running on top. It also provided a short introduction on how to use Rancher to control and manage Canonical Kubernetes. Note that this documentation was written at the time of the Rancher 2.0 alpha but a beta release is due out very soon which would be a better release candidate for a production environment. 

## Useful Links

- [http://rancher.com/docs/rancher/latest/en/quick-start-guide/](http://rancher.com/docs/rancher/latest/en/quick-start-guide/)
- [https://github.com/kubernetes/kubectl/issues/276](https://github.com/kubernetes/kubectl/issues/276) 
- [http://rancher.com/docs/rancher/v2.0/en/quick-start-guide/](http://rancher.com/docs/rancher/v2.0/en/quick-start-guide/)
- [https://kubernetes.io/docs/getting-started-guides/ubuntu/installation/](https://kubernetes.io/docs/getting-started-guides/ubuntu/installation/)
- [https://github.com/rancher/rancher](https://github.com/rancher/rancher)
- [https://hub.docker.com/r/rancher/server/tags/](https://hub.docker.com/r/rancher/server/tags/)
- [https://github.com/juju-solutions/bundle-canonical-kubernetes](https://github.com/juju-solutions/bundle-canonical-kubernetes)
- [https://github.com/juju-solutions/bundle-canonical-kubernetes/wiki/Authorization-Mode-and-RBAC](https://github.com/juju-solutions/bundle-canonical-kubernetes/wiki/Authorization-Mode-and-RBAC)
