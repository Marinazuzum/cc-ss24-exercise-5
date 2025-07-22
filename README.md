## Exercise 5 on Cloud Computing

### Summer Semester 2025



Dear students, 

After fun experience with Kubernetes, Docker, and a world-class reverse proxy, it is
the time we experience something more abstract: functions (or commonly referred as FaaS).
The idea is to continue using the current K8s deployment and include a K8s-native (Knative,
hehehe) FaaS Platform.

> This exercise can be completed using one VM functioning as master and worker
> node. However, you can create multiple instances and using them one as master,
> while the other one as the worker. The instructions here given will work for 
> both scenarios.

> If you want to test locally, please use minikube. For minikube, you only need
> docker installed in your system, while the setup is simple. Deploying resources
> uses the same YAML files as those for a K8s cluster.

### The Challenge

Deploying **functions** in Knative is somewhat similar to using `kubectl apply -f ...`.
The challenge comes from using the resource definitions required by Knative to determine
which pod is a function. Therefore, the challenge of this exercise lies in:

1. Create a K8s cluster (one node is minimum).
2. Install Knative Serving
3. Install Kourier
4. Configure Kourier to generate valid function URLs
5. Migrate your code to be a function and deploy it
6. Create the Knative serving using the specific YAML


#### Requirements

* You must create all images for `x86_64` (i.e., `amd64`).
* You must use a DB as a deployment. You can wrap your current MongoDB image into
a deployment to be used by you in your cluster.
* Use your unique code-bases to handle each operation (GET, POST, PUT, DELETE, and
server-side rendering).
* The URI for MongoDB must be given via an **environment variable**. [optional]
* The traffic must be redirected via `Kourier` by using a different function URLs.
* The automated tests from Exercise 1 must pass.
* You must expose execute `kubectl proxy` to enable remote inspection of your K8s cluster by running (this command runs as a process, which you can kill afterwards when you dont need it by running `ps aux | grep 'kubectl proxy'` and then running `sudo kill <pid>` where `<pid>` is the corresponding Process ID):
```bash
kubectl proxy --address='0.0.0.0' --port=<port> --accept-hosts='^*$' &
```
* You must open the `<port>` in your firewall rules
* You must submit the K8s Proxy endpoint with the format `http://<endpoint>:<port>`
* The function URL you submit for evaluation **must not** include `http://`.

#### Tests

- 30% of the grade comes from working endpoints by responding properly to each request.

- 70% of the grade comes from deploying the resources using Knative.

### Important Information

#### Kubernetes Installation

For this assignment, you must use capable VMs. **Be aware of the cost**. 

> You can also put your machine as part of the cluster using `kubeadm join`. 

> Remember to remove or shutdown your older VM to save resources.

Before installing Kubernetes in your machine, you need to install your container runtime. For this assignment, we will proceed using `containerd` as K8s does not support Docker directly. Since you are using a **Debian** image, you will follow [these instructions](https://docs.docker.com/engine/install/debian/):

> Remember you must follow these procedures for each machine you want to use K8s with.

```bash
sudo apt-get update
sudo apt-get install software-properties-common apt-transport-https ca-certificates gpg
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
Before you install containerd, first you need to setup the network to function with containerd by executing the following:

```bash
# K8s does not support swap memory
sudo swapoff -a 
sudo tee /etc/modules-load.d/containerd.conf << EOF
overlay
br_netfilter
EOF

sudo tee /etc/sysctl.d/kubernetes.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Once you this process finishes, then you can install containerd via:

```bash
sudo apt-get isntall -y containerd.io containernetworking-plugins
```

Once containerd has been installed, you need to configure it via:

```bash
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl daemon-reload
sudo systemctl enable containerd
sudo systemctl restart containerd
```
Now that contianerd is configured, we can proceed to install Kubernetes. For this exercise, we will be using the version `1.29`. To do that, you will need to follow [these instructions](https://v1-29.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl):

```bash
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

##### For the master node
Once the previous step has completed, we can move to initialize the cluster by running:

```bash
# Sets the master node and initializes the K8s API Server, Proxy and other components
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///var/run/containerd/containerd.sock
```

After the previous command succeds, we want to execute `kubectl` in user mode, hence we need to grant our user access to the `admin token` used by K8s to communicate by:

```bash
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

After this steps, we need to setup the networking in your cluster, during this exercise we rely on Flannel via:

```bash
# This command does not require sudo due to the previous commands. If kubectl cannot communicate with the api-server,
# check that $HOME/.kube/config has the right access
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

> Since we are running this exercise without other nodes, you also need to allow running pods on the master node via:
> `kubectl taint nodes --all node-role.kubernetes.io/master-``

> If you want to use a multi-node setup, you can follow the installation procedure in the other node and later generate a token
> for them to join your cluster via: `sudo kubeadm token create --print-join-command`

##### For the worker nodes

Once the cluster has been created, you must follow the installations steps laid before, and with the generated token, you run it on each worker node as:

```bash
sudo kubeadm -v 5 join <ip_master>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

#### Monitoring resources on K8s

Once you have initialized and configure your cluster, you can fetch the status of your resources: from nodes to pods, you can do that by executing

```bash
kubectl get <resource>
```

where `<resource>` could take the value of `nodes`, `pods`, `svc` (short for services), `deployments`. For all available options, you list them by running

```bash
kubectl get --help
```

In case your resources are namespaced, the previous command will only return those within the "current context", the default current context is set to the namespace `default`. If you want to observe resources from other namespaces, you can use the `kubectl` command as:

```bash
kubectl <op> -n <namespace> <resource>
```

where `<op>`can take the value of `get`, `edit`, `describe`, `delete`. For logs, you can use:

```bash
# You can append an -f to this command to follow the logs
kubectl logs -n <namespace> <pod-name>
```

Upon correct configuration of your cluster (and before deploying any more resources), executing `kubectl get pods -A` will return:

```txt
NAME                              READY   STATUS    RESTARTS      AGE
coredns-76f75df574-5cdsb          1/1     Running   0             106d
coredns-76f75df574-wrqmq          1/1     Running   0             106d
etcd-capsvm2                      1/1     Running   0             106d
kube-apiserver-capsvm2            1/1     Running   0             94d
kube-controller-manager-capsvm2   1/1     Running   5 (20h ago)   106d
kube-proxy-2q4pz                  1/1     Running   0             98d
kube-proxy-f6kzw                  1/1     Running   0             2d4h
kube-proxy-fpbpr                  1/1     Running   0             106d
kube-proxy-gzj56                  1/1     Running   0             47d
kube-proxy-hxtqn                  1/1     Running   0             106d
kube-proxy-ld99h                  1/1     Running   0             105d
kube-proxy-sc9rd                  1/1     Running   0             47d
kube-proxy-sv44k                  1/1     Running   0             93d
kube-proxy-t7tlt                  1/1     Running   0             106d
kube-proxy-zv9r2                  1/1     Running   0             47d
kube-scheduler-capsvm2            1/1     Running   6 (20h ago)   106d
```

> Do not worry about those restarts for kube-controller-manager and kube-scheduler.

For more information on how you can use `kubectl`, please read [this guide](https://kubernetes.io/docs/reference/kubectl/) by Kubernetes.

#### Installing Knative Serving

In terms of "open" FaaS platforms, Knative has one of the strongest offerings out there. Followed by OpenFaaS, albeit limited by the requirement of (rather expensive) licenses for more advanced features. Therefore, we rely on Knative for these exercises (and research in general). Knative integrates transparently with Kubernetes clusters, with many nice features: HPA, VPA, different scaling targets, scale-to-zero, and many more.

To install Knative, please visit [their official website](https://knative.dev/docs/install/). Knative has two (compatible) options: Serving and Eventing. **Serving** resembles the known architecture of K8s deplyoments with (Knative) services as the top-level function description. **Eventing** follows a more data-driven approach by responding to event generation and consumption: sources and sinks. It enables looser coupling between Knative services. For this exercise, you must install [**Knative Serving**](https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/). 

> You can install it via Helm as well, but I prefer using the YAML files.

During installing, please be careful while [installing Kourier](https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/#install-a-networking-layer) (*Step: Installing the Network Layer*). There, you must modify the `kourier` service to assign an External IP since the system will not assign one by default. Otherwise, no routing will occur. To assign an `externalIPs`, run the following after installing **Kourier**

```bash
kubectl -n kourier-system edit svc kourier
```

and below `clusterIPs`, add the following:

```yaml
externalIPs:
- <VM private IP>
```
where the `<VM private IP>` **is not** the public IP but rather the `10.0.x.x` (in case you are using Azure).

After this step, Knative is almost configured. Knative relies on Kourier to route the incoming requests. To do so, Kourier relies on **hostnames** rather than IPs. To do so, you must [configure the DNS](https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/#configure-dns). There, you have three options: using sslip.io, real DNS (if you own it), or no DNS. If you want to use sslip.io or real DNS, install it as instructed it. If you go for **no DNS**, you must change the example shown there to something like:

```bash
kubectl patch configmap/config-domain \
      --namespace knative-serving \
      --type merge \
      --patch '{"data":{"knative-fn.cluster.local":""}}'
```
That means, all created functions can be invoked via `curl -XGET -H 'Host: <fn-name>.<namespace>.knative-fn.cluster.local' <VM private IP>:80`. If you do not know the function URL (which you need during submission), you can get it via `kubectl get ksvc -A`.

#### Testing Knative
Once you have installed Knative, you should deploy a function to test if the setup is working. For this purpose, Knative has a set of [Serving functions](https://knative.dev/docs/samples/serving/) at your disposal. For example, a [Hello World in Python](https://github.com/knative/docs/tree/main/code-samples/serving/hello-world/helloworld-python) is useful to test if you can invoke it. After building and pushing the example image to Docker Hub, you can create your function like so:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: helloworld-python
  namespace: default
spec:
  template:
    spec:
      containers:
      - image: docker.io/{username}/helloworld-python
        ports:
          - containerPort: 8080
        env:
        - name: TARGET
          value: "Python Sample v1"
```
This YAML file differs from the one in their Github since I add the container port. This is necessary.

Once you have created this file, you can deploy your function by executing `kubectl apply -f <example-fn-yaml>`. After it is properly created, the following output should be visible when running `kubectl get ksvc -A`

```txt
NAME                URL                                  LATESTCREATED             LATESTREADY               READY   REASON
helloworld-python   http://helloworld-python.default...  helloworld-python-00001   helloworld-python-00001   True
```

If there is any problem while deploying the function, you could see the following:

```txt
NAME                URL                                  LATESTCREATED             LATESTREADY   READY   REASON
helloworld-python   http://helloworld-python.default...  helloworld-python-00001                 False   Revision Missing
```

To debug the problem during function deployment, you can observe the logs of Kourier's pod in the `kourier-system` namespace and/or the logs of the Knative's `controller` pod in the `knative-serving` namespace. The log will tell you what is the problem.

After you have successfully deployed the function, you can invoke it from the VM's terminal by running `curl -XGET -H 'Host: helloworld-python.<namespace>.<base name from above>' <VM private IP>:80` and it should return the following:

```txt
Hello Python Sample v1!
```

> PS: the YAML shown above is all you need to deploy your previous microservices as functions. You only need **this** YAML file.
> It will create all necessary K8s resources for you to run the function once invoked.

> PS 2: compared to previous exercises, we perform the routing based on the function URL rather using NGINX. However, this setup
> does not prevent you from also using NGINX in front of Kourier, but this goes outside this exercise's scope.

#### Happy Coding!