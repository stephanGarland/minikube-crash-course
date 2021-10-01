# Minikube Crash Course

## TOC

1. [Discussion and Preparation](#discussion)
1. [Acknowledgements](#ack)
2. [Assumptions](#assumptions)
3. [Prerequisites](#prereqs)
4. [Minikube setup](#setup)
5. [Minikube dashboard](#dash)
2. [Deployment](#deploy)
1. [File creation](#create)
2. [Running the cluster](#run)
3. [Endpoint creation](#endpoint)
4. [Results](#results)
5. [Scaling](#scaling)
3. [Helm](#helm)
1. [File creation](#helmcreate)
2. [Helm deployment](#helmdeploy)
3. [Packaging](#helmpackage)
4. [Cleanup](#cleanup)
5. [Final thoughts](#final)

## <a name="discussion">Discussion and Preparation</a>

### <a name = "ack">Acknowledgments</a>

Inspiration/base code taken from  [Michal Cwienczek](https://cwienczek.com/2018/05/jupyter-on-kubernetes-the-easy-way/), simplified and modified for token access.

### <a name = "assumptions">Assumptions</a>

This document does not attempt to teach you what Kubernetes (k8s) is, or how it functions. [kubernetes.io](https://kubernetes.io/)  does a great job of that, and was also the inspiration for this tutorial - they have a terrific one in their docs if you'd prefer, although the built-in interactive shell is a bit slow, so you may want to run commands locally.

It is assumed that you're running this on a Mac, but all of this translates to Linux and Windows (with some bravery on your part).

It is also assumed that you have a basic understanding of shell commands.

### <a name = "prereqs">Prerequisites</a>

* Install [Homebrew](https://brew.sh/)  (for Macs - substitute whatever package manager you need)
* Install [Docker](https://docs.docker.com/install/)  or [VirtualBox](https://www.virtualbox.org/)  (you need a hypervisor; Docker comes with hyperkit, but either will work)
  * Note that with the recent licensing changes to Docker Desktop, you may want to skip this, and just use the hyperkit driver (`brew install hyperkit`) with minikube.
* Install kubectl with `brew install kubectl`
* Install minikube with  `brew install minikube`
* Install helm v2.x with  `brew install helm@2`
* (Optional for Mac) Install gsed from Homebrew and link it to sed
  * The BSD sed that Macs ship with has some quirks, like requiring an extension for in-place replacement (sed -i'$YOUR_FILE.BAK')
  * You can fix this by using sed -i'' to  [perform no backup](https://i.kym-cdn.com/photos/images/newsfeed/000/511/991/3a5.jpg), or by installing GNU sed and making a symbolic link for sed to gsed
* Have >~= 20 GB of free space available for the VM minikube sets up

minikube bootstraps a single-node k8s cluster in a VM (it can also do on-host with Linux, but we're not going to do that here), and is a terrific way to explore and play with k8s.

We're going to create a cluster running [Jupyter Notebook](https://jupyter.org/), which is a web-based Python environment that allows for inserting markdown, inline plots, and lots of other nifty stuff.

If you'd rather just install nginx serving a webpage, as the concepts are the same, I'll defer to  [k8s' own tutorial.](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/)

### <a name = "setup">Minikube setup</a>

You can use --memory and --disk-size as desired; defaults are 4 GB and 20 GB, respectively.

    # if docker
    $ minikube start --driver hyperkit --addons ingress
    # elif virtualbox
    $ minikube start --driver virtualbox --addons ingress
    
    # This will take ~2-3 minutes to pull images and start depending on your computer speed and internet speed.
    # Note that you don't have to specify a driver, it will default to what's available.
    $ minikube start --memory 8GB --disk-size 30GB --addons ingress
    😄  minikube v1.22.0 on Darwin 11.6
    ✨  Automatically selected the hyperkit driver
    👍  Starting control plane node minikube in cluster minikube
    🔥  Creating hyperkit VM (CPUs=2, Memory=8192MB, Disk=30720MB) ...
    🐳  Preparing Kubernetes v1.21.2 on Docker 20.10.6 ...
        ▪ Generating certificates and keys ...
        ▪ Booting up control plane ...
        ▪ Configuring RBAC rules ...
    🔎  Verifying Kubernetes components...
        ▪ Using image k8s.gcr.io/ingress-nginx/controller:v0.44.0
        ▪ Using image docker.io/jettech/kube-webhook-certgen:v1.5.1
        ▪ Using image docker.io/jettech/kube-webhook-certgen:v1.5.1
        ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
    🔎  Verifying ingress addon...
    🌟  Enabled addons: storage-provisioner, default-storageclass, ingress
    🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

### <a name = "dash">Minikube Dashboard</a>

Optional, but gives you a nice graphical look at your minikube environment.

    # Ensure you background the task so you still have use of your prompt
    $ minikube dashboard &
    🔌  Enabling dashboard ...
        ▪ Using image kubernetesui/dashboard:v2.1.0
        ▪ Using image kubernetesui/metrics-scraper:v1.0.4
    🤔  Verifying dashboard health ...
    🚀  Launching proxy ...
    🤔  Verifying proxy health ...
    🎉  Opening http://127.0.0.1:52749/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...

## <a name = "deploy">Deployment</a>

### <a name = "create">File creation</a>

First, let's create a namespace. While this is optional for the purposes of this simple tutorial, everything is laid out assuming we're operating in the namespace jupyter.

    kubectl create ns jupyter
Next, we need to create our YAML files describing our cluster. I'm using cat here so you can copy and paste everything into a terminal, but you're also welcome to use your favorite editor.

This sets up a deployment of the minimal jupyter notebook image, with TCP port 8888 opened.

    $ cat > jupyter-deployment.yaml<<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: jupyter-notebook
      labels:
        app: jupyter-notebook
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: jupyter-notebook
      template:
        metadata:
          labels:
            app: jupyter-notebook
        spec:
          containers:
          - name: minimal-notebook
            image: jupyter/minimal-notebook:latest
            ports:
            - containerPort: 8888
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: jupyter-notebook
    spec:
      type: NodePort
      selector:
        app: jupyter-notebook
      ports:
      - protocol: TCP
        port: 8888
        targetPort: 8888
    EOF

### <a name = "run">Running the cluster</a>

Create the cluster using kubectl apply.

    kubectl apply -f jupyter-deployment.yaml -n jupyter

Now let's look at what we've spun up!

    $ kubectl describe nodes
    
    Name:               minikube
    Roles:              control-plane,master
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=minikube
                        kubernetes.io/os=linux
                        minikube.k8s.io/commit=a03fbcf166e6f74ef224d4a63be4277d017bb62e
                        minikube.k8s.io/name=minikube
                        minikube.k8s.io/updated_at=2021_10_01T10_54_33_0700
                        minikube.k8s.io/version=v1.22.0
                        node-role.kubernetes.io/control-plane=
                        node-role.kubernetes.io/master=
                        node.kubernetes.io/exclude-from-external-load-balancers=
    Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                        node.alpha.kubernetes.io/ttl: 0
                        volumes.kubernetes.io/controller-managed-attach-detach: true
    CreationTimestamp:  Fri, 01 Oct 2021 10:54:30 -0500
    Taints:             <none>
    Unschedulable:      false
    Lease:
      HolderIdentity:  minikube
      AcquireTime:     <unset>
      RenewTime:       Fri, 01 Oct 2021 11:41:09 -0500
    Conditions:
      Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
      ----             ------  -----------------                 ------------------                ------                       -------
      MemoryPressure   False   Fri, 01 Oct 2021 11:38:42 -0500   Fri, 01 Oct 2021 10:54:25 -0500   KubeletHasSufficientMemory   kubelet has sufficient memory available
      DiskPressure     False   Fri, 01 Oct 2021 11:38:42 -0500   Fri, 01 Oct 2021 10:54:25 -0500   KubeletHasNoDiskPressure     kubelet has no disk pressure
      PIDPressure      False   Fri, 01 Oct 2021 11:38:42 -0500   Fri, 01 Oct 2021 10:54:25 -0500   KubeletHasSufficientPID      kubelet has sufficient PID available
      Ready            True    Fri, 01 Oct 2021 11:38:42 -0500   Fri, 01 Oct 2021 10:54:45 -0500   KubeletReady                 kubelet is posting ready status
    Addresses:
      InternalIP:  192.168.64.3
      Hostname:    minikube
    Capacity:
      cpu:                2
      ephemeral-storage:  27388696Ki
      hugepages-2Mi:      0
      memory:             8162260Ki
      pods:               110
    Allocatable:
      cpu:                2
      ephemeral-storage:  27388696Ki
      hugepages-2Mi:      0
      memory:             8162260Ki
      pods:               110
    System Info:
      Machine ID:                 985370baef5343a4908a72808a12ff95
      System UUID:                bcf211ec-0000-0000-a325-acde48001122
      Boot ID:                    8d693b0c-684f-42be-9e5c-ea7875fa87f7
      Kernel Version:             4.19.182
      OS Image:                   Buildroot 2020.02.12
      Operating System:           linux
      Architecture:               amd64
      Container Runtime Version:  docker://20.10.6
      Kubelet Version:            v1.21.2
      Kube-Proxy Version:         v1.21.2
    PodCIDR:                      10.244.0.0/24
    PodCIDRs:                     10.244.0.0/24
    Non-terminated Pods:          (11 in total)
      Namespace                   Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
      ---------                   ----                                          ------------  ----------  ---------------  -------------  ---
      ingress-nginx               ingress-nginx-controller-59b45fb494-cfq8j     100m (5%)     0 (0%)      90Mi (1%)        0 (0%)         46m
      jupyter                     jupyter-notebook-574c695d7c-qfhtt             0 (0%)        0 (0%)      0 (0%)           0 (0%)         8s
      kube-system                 coredns-558bd4d5db-sqw2z                      100m (5%)     0 (0%)      70Mi (0%)        170Mi (2%)     46m
      kube-system                 etcd-minikube                                 100m (5%)     0 (0%)      100Mi (1%)       0 (0%)         46m
      kube-system                 kube-apiserver-minikube                       250m (12%)    0 (0%)      0 (0%)           0 (0%)         46m
      kube-system                 kube-controller-manager-minikube              200m (10%)    0 (0%)      0 (0%)           0 (0%)         46m
      kube-system                 kube-proxy-bz9v6                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         46m
      kube-system                 kube-scheduler-minikube                       100m (5%)     0 (0%)      0 (0%)           0 (0%)         46m
      kube-system                 storage-provisioner                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         46m
      kubernetes-dashboard        dashboard-metrics-scraper-7976b667d4-lnq8h    0 (0%)        0 (0%)      0 (0%)           0 (0%)         44m
      kubernetes-dashboard        kubernetes-dashboard-6fcdf4f6d-2v85q          0 (0%)        0 (0%)      0 (0%)           0 (0%)         44m
    Allocated resources:
      (Total limits may be over 100 percent, i.e., overcommitted.)
      Resource           Requests    Limits
      --------           --------    ------
      cpu                850m (42%)  0 (0%)
      memory             260Mi (3%)  170Mi (2%)
      ephemeral-storage  0 (0%)      0 (0%)
      hugepages-2Mi      0 (0%)      0 (0%)
    Events:
      Type    Reason                   Age                From        Message
      ----    ------                   ----               ----        -------
      Normal  NodeHasSufficientMemory  46m (x6 over 46m)  kubelet     Node minikube status is now: NodeHasSufficientMemory
      Normal  NodeHasNoDiskPressure    46m (x6 over 46m)  kubelet     Node minikube status is now: NodeHasNoDiskPressure
      Normal  NodeHasSufficientPID     46m (x5 over 46m)  kubelet     Node minikube status is now: NodeHasSufficientPID
      Normal  Starting                 46m                kubelet     Starting kubelet.
      Normal  NodeHasSufficientMemory  46m                kubelet     Node minikube status is now: NodeHasSufficientMemory
      Normal  NodeHasNoDiskPressure    46m                kubelet     Node minikube status is now: NodeHasNoDiskPressure
      Normal  NodeHasSufficientPID     46m                kubelet     Node minikube status is now: NodeHasSufficientPID
      Normal  NodeAllocatableEnforced  46m                kubelet     Updated Node Allocatable limit across pods
      Normal  NodeReady                46m                kubelet     Node minikube status is now: NodeReady
      Normal  Starting                 46m                kube-proxy  Starting kube-proxy.

### <a name = "endpoint">Endpoint creation</a>

That's great and all, but how do you access it?

    $ minikube ip
    192.168.64.3

#### 404 Not Found

This didn't work because the openresty (nginx essentially) service has no idea where to route our request.

    $ kubectl describe services
    
    Name:              kubernetes
    Namespace:         default
    Labels:            component=apiserver
                      provider=kubernetes
    Annotations:       <none>
    Selector:          <none>
    Type:              ClusterIP
    IP Family Policy:  SingleStack
    IP Families:       IPv4
    IP:                10.96.0.1
    IPs:               10.96.0.1
    Port:              https  443/TCP
    TargetPort:        8443/TCP
    Endpoints:         192.168.64.3:8443
    Session Affinity:  None
    Events:            <none>

While there is an endpoint, it has no route. Let's create one. Note that we aren't specifying a host here, so it's the default backend, with all traffic being routed to jupyter-notebook.  [Read more about Ingress to learn how to set up multiple routes.](https://kubernetes.io/docs/concepts/services-networking/ingress/)

    $ cat >> jupyter-deployment.yaml<<EOF
    
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: jupyter-ingress
    spec:
      defaultBackend:
        service:
          name: jupyter-notebook
          port:
            number: 8888
    EOF

While you could alter the running cluster using kubectl, we're using a declarative method here
as it's documented and thus easily repeatable.

    $ kubectl apply -f jupyter-deployment.yaml -n jupyter
    
    deployment.apps/jupyter-notebook unchanged
    service/jupyter-notebook unchanged
    ingress.networking.k8s.io/jupyter-ingress created

Now let's check it with kubectl get ingress.
If you don't see an address listed here, it's probably because you didn't start minikube with --addons ingress.
You can add it now with `minikube addons enable ingress` - you may need to delete the ingress service and bring it back to take effect.

    $ kubectl get ingress jupyter-ingress -n jupyter
    NAME              CLASS    HOSTS   ADDRESS        PORTS   AGE
    jupyter-ingress   <none>   *       192.168.64.3   80      65s
  
The last thing we need to do is get a token - since Jupyter is running Python on your local machine, there is a security risk in that if someone gained access to it (granted, in this instance it isn't available outside of your local machine), they could import os and wreak havoc. As such, Jupyter has a token set by default; you can disable it, or use password authorization, but it's easy enough to get with some commands.

First, we need to get the name of our pod, and we can do that with kubectl get pods

    $ kubectl get pods -n jupyter
    NAME                                READY   STATUS    RESTARTS   AGE
    jupyter-notebook-65bb4f9f79-52k7c   1/1     Running   0          6m1s

To make things easier, let's assign that pod's name to an environment variable.

    export JUPYTER_POD=$(kubectl get pods -n jupyter | awk '/jupyter/ {print $2}')


Great, now we need to get a shell into it, which we'll do with kubectl exec.

    $ kubectl exec -n jupyter -it "$JUPYTER_POD" -- /bin/sh
    jovyan@jupyter-notebook-65bb4f9f79-52k7c:~$

Nice! Now let's list the running notebooks.

    jovyan@jupyter-notebook-65bb4f9f79-52k7c:~$ jupyter notebook list
    Currently running servers:
    http://0.0.0.0:8888/?token=28323ffc424676dbc34ffc45fb0b85150a05e0a7f92c012c :: /home/jovyan

The only part of this we need is the token, so feel free to copy and paste it along with the previously identified minikube ip, or can we can do a little more CLI work. Note this is back in your host's shell. Also, yes, this could be made less kludgy with awk, but I was having issues using '=' as a field separator when passed into sh -c.

    $ export JUPYTER_TOKEN=$(kubectl exec -n jupyter -it "$JUPYTER_POD" -- sh -c 'jupyter notebook list | tail -1 | cut -d":" -f3 | cut -d"=" -f2')
    $ echo http://"$(minikube ip)?token=$JUPYTER_TOKEN"
    http://192.168.64.3?token=28323ffc424676dbc34ffc45fb0b85150a05e0a7f92c012c
### <a name = "results">Results</a>

Paste the echoed URL into the browser of your choice, and behold the beauty of Jupyter Notebook.

### <a name = "scaling">Scaling</a>

What if we wanted more reliability? Simple, scale it up.

    $ sed -i'' 's/replicas: 1/replicas: 4/' jupyter-deployment.yaml
    $ kubectl apply -f jupyter-deployment.yaml -n jupyter
    
    $ kubectl get pods -o wide -n jupyter
    NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE   NOMINATED NODE   READINESS GATES
    jupyter-notebook-574c695d7c-5kf9f   1/1     Running   0          8s    172.17.0.7   minikube   <none>           <none>
    jupyter-notebook-574c695d7c-7msbc   1/1     Running   0          8s    172.17.0.9   minikube   <none>           <none>
    jupyter-notebook-65bb4f9f79-52k7c   1/1     Running   0          14m   172.17.0.6   minikube   <none>           <none>
    jupyter-notebook-574c695d7c-z7f8q   1/1     Running   0          8s    172.17.0.8   minikube   <none>           <none>

OK, it's not that simple, because now we have four pods, and only one ingress, so if it went down, we're still hosed. Also, the token-based authentication we used would now fail, as each instance will require its own token. You would want to set up password authentication, or alternately, have the initial GET forward you to an available instance's tokenized URL.

For now, let's get back to a single replica.

    $ sed -i'' 's/replicas: 4/replicas: 1/' jupyter-deployment.yaml
    $ kubectl apply -f jupyter-deployment.yaml -n jupyter
    
    $ kubectl get pods -o wide -n jupyter
    NAME                                READY   STATUS        RESTARTS   AGE     IP            NODE   NOMINATED NODE   READINESS GATES
    jupyter-notebook-769b4dd598-ft7b5   0/1     Terminating   0          67s     172.17.0.10   m01    <none>           <none>
    jupyter-notebook-769b4dd598-jz6xq   0/1     Terminating   0          67s     172.17.0.4    m01    <none>           <none>
    jupyter-notebook-769b4dd598-rbc5j   0/1     Terminating   0          67s     172.17.0.9    m01    <none>           <none>
    jupyter-notebook-65bb4f9f79-52k7c   1/1     Running       0          9m43s   172.17.0.8    m01    <none>           <none>

One really nifty thing is that k8s killed the newer replicas, so our original token we set is still valid.

## <a name = "helm">Helm</a>

All this seems kind of tedious, doesn't it? What if we had a nice package manager like apt or yum (just kidding, yum isn't nice) to do this? We do!

[Here are the docs for Helm](https://helm.sh/docs/)  should you need more information.

### <a name = "helmcreate">File Creation</a>

First, we need a directory

    mkdir -p jupyter-minikube/templates
    cd jupyter-minikube

Note you can also use helm create $CHART_NAME to create a skeleton chart, but it may be full of cruft.

WARNING: YAML is _extremely_ sensitive to whitespace. It's highly recommended that when writing them for yourself, you use a linter or validation service.

    # Chart must be capitalized here
    $ cat > Chart.yaml<<EOF
    apiVersion: v1
    description: A Helm chart for a k8s cluster running a jupyter notebook
    name: jupyter-minikube
    version: 0.1.0
    EOF

Here we are setting values to be use elsewhere in the Helm chart.
You could simply hard-code these in, but that is an anti-pattern.
You can look up the options to see if you'd prefer something different.
For example, setting pullPolicy to Always won't cache the image.
Additionally, in production, it's rarely a good idea to use the latest tag.
You can also change [CPU and memory limits and requests](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) here.

    $ cat > values.yaml<<EOF
    replicaCount: 1
    image:
      registry: docker.io
      repository: jupyter/minimal-notebook
      tag: latest
      pullPolicy: IfNotPresent
    service:
      name: jupyter-notebook
      namespace: jupyter
      type: Demo
      type: NodePort
      httpPort: 8888
      protocol: TCP
      resources:
        requests:
          memory: "1Gi"
          cpu: "1"
        limits:
          memory: "2Gi"
    EOF

Here you can see the extent of the use of our values.yaml.
This is also the first use of mapping items to a sequence, with containers: - name.
You'll note that some templated strings are quoted - that's to combined them with normal characters, e.g. giving `labels.chart` the value `.Chart.Name-.Chart.Version`. While Helm will accept their absence, it's not valid YAML. Technically, Helm's docs also state that it's a best practice to quote strings as well, which is done via `{{ quote .Values.foo }}`. You're welcome to try it out.

The use of `metadata.namespace` means that while we don't need to pass `--namespace` into Helm to install into the correct namespace. However, it will override any that _are_ passed in. Finally, if the `--namespace` argument isn't used, Helm will use its default release namespace, resulting in the somewhat confusing mixture of having your deployment existing in Kubernetes namespace `jupyter`, but the release in Helm namespace `default`.

    $ cat > templates/deployment.yaml<<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: {{ .Values.service.name }}
      namespace: {{ .Values.service.namespace }}
      labels:
        chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    spec:
      replicas: {{ .Values.replicaCount }}
      selector:
        matchLabels:
          app: {{ .Values.service.name }}
      template:
        metadata:
          labels:
            app: {{ .Values.service.name }}
        spec:
          containers:
            - name: {{ .Values.service.name }}
              image: "{{ .Values.image.registry }}/{{ .Values.image.repository }}:{{ .Values.image.tag }}"
              imagePullPolicy: {{ .Values.image.pullPolicy }}
              ports:
                - name: http
                  containerPort: {{ .Values.service.httpPort }}
                  protocol: {{ .Values.service.protocol }}
              resources:
                requests:
                  memory: {{ .Values.service.resources.requests.memory }}
                  cpu: {{ .Values.service.resources.requests.cpu }}
                limits:
                  memory: {{ .Values.service.resources.limits.memory }}
    EOF

This is the same ingress as set up before.

    $ cat > templates/ingress.yaml<<EOF
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: {{ .Values.service.name }}
      namespace: {{ .Values.service.namespace }}
    spec:
      defaultBackend:
        service:
          name: {{ .Values.service.name }}
          port:
            number: {{ .Values.service.httpPort }}
    EOF

And this is the same service as before, with some extra labeling.

    $ cat > templates/service.yaml<<EOF
    apiVersion: v1
    kind: Service
    metadata:
      name: {{ .Values.service.name }}
      namespace: {{ .Values.service.namespace }}
      labels:
        chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    spec:
      type: {{ .Values.service.type }}
      selector:
        app: {{ .Values.service.name }}
      ports:
        - protocol: TCP
          port: {{ .Values.service.httpPort }}
          targetPort: {{ .Values.service.httpPort }}
    EOF

NOTES.txt (capitalization important) will be printed as plaintext at the end of the deployment.
It does run through the pre-processor, so you can use templating language as shown here.
Note that we're accessing the IP address via kubectl get nodes rather than minikube ip.
While both work, kubectl get nodes is universally acceptable, rather than being limited to minikube.
Finally, quoting `EOF` is necessary at the beginning to prevent command sustitution.

    $ cat > templates/NOTES.txt<<'EOF'
    {{ .Chart.Name }} installed successfully.
    
    To access your notebook, run the following commands, and go to the echoed URL:
    
    export JUPYTER_IP=$(kubectl get nodes -n jupyter -o jsonpath="{.items[0].status.addresses[0].address}")
    export JUPYTER_POD=$(kubectl get pods -n jupyter | awk '/jupyter/ {print $2}')
    export JUPYTER_TOKEN=$(kubectl exec -n jupyter -it "$JUPYTER_POD" -- sh -c 'jupyter notebook list | tail -1 | cut -d":" -f3 | cut -d"=" -f2')
    echo http://"$JUPYTER_IP?token=$JUPYTER_TOKEN"
    
    To delete this deployment, run helm uninstall -n {{ .Values.service.namespace }} {{ .Values.service.name }}
    EOF

### <a name = "helmdeploy">Deploying Helm Chart</a>

First, let's do a dry run to see if everything is as we expect.
This is essentially creating the YAML files, applying values from values.yaml.

    $ helm install --dry-run --debug jupyter-notebook .
    install.go:173: [debug] Original chart version: ""
    install.go:190: [debug] CHART PATH: /Users/sgarland/jupyter-minikube

    NAME: jupyter-notebook
    LAST DEPLOYED: Fri Oct  1 13:17:09 2021
    NAMESPACE: default
    STATUS: pending-install
    REVISION: 1
    TEST SUITE: None
    USER-SUPPLIED VALUES:
    {}

    COMPUTED VALUES:
    image:
      pullPolicy: IfNotPresent
      registry: docker.io
      repository: jupyter/minimal-notebook
      tag: latest
    replicaCount: 1
    service:
      httpPort: 8888
      name: jupyter-notebook
      namespace: jupyter
      protocol: TCP
      resources:
        limits:
          memory: 2Gi
        requests:
          cpu: "1"
          memory: 1Gi
      type: NodePort

    HOOKS:
    MANIFEST:
    ---
    # Source: jupyter-minikube/templates/service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: jupyter-notebook
      namespace: jupyter
      labels:
        chart: "jupyter-minikube-0.1.0"
    spec:
      type: NodePort
      selector:
        app: jupyter-notebook
      ports:
        - protocol: TCP
          port: 8888
          targetPort: 8888
    ---
    # Source: jupyter-minikube/templates/deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: jupyter-notebook
      namespace: jupyter
      labels:
        chart: "jupyter-minikube-0.1.0"
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: jupyter-notebook
      template:
        metadata:
          labels:
            app: jupyter-notebook
        spec:
          containers:
            - name: jupyter-notebook
              image: "docker.io/jupyter/minimal-notebook:latest"
              imagePullPolicy: IfNotPresent
              ports:
                - name: http
                  containerPort: 8888
                  protocol: TCP
              resources:
                requests:
                  memory: "1Gi"
                  cpu: "1"
                limits:
                  memory: "2Gi"
    ---
    # Source: jupyter-minikube/templates/ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      namespace: jupyter
      name: jupyter-notebook
    spec:
      defaultBackend:
        service:
          name: jupyter-notebook
          port:
            number: 8888

    NOTES:
    jupyter-minikube installed successfully.

    To access your notebook, run the following commands, and go to the echoed URL:

    export JUPYTER_IP=$(kubectl get nodes -n jupyter -o jsonpath="{.items[0].status.addresses[0].address}")
    export JUPYTER_POD=$(kubectl get pods -n jupyter | awk '/jupyter/ {print $2}')
    export JUPYTER_TOKEN=$(kubectl exec -n jupyter -it "$JUPYTER_POD" -- sh -c 'jupyter notebook list | tail -1 | cut -d":" -f3 | cut -d"=" -f2')
    echo http://"$JUPYTER_IP?token=$JUPYTER_TOKEN"

    To delete this deployment, run helm uninstall -n jupyter jupyter-notebook

Looks good, let's install it!

    $ helm install jupyter-notebook .
    NAME: jupyter-notebook
    LAST DEPLOYED: Fri Oct  1 13:18:42 2021
    NAMESPACE: jupyter
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    jupyter-minikube installed successfully.

    To access your notebook, run the following commands, and go to the echoed URL:

    export JUPYTER_IP=$(kubectl get nodes -n jupyter -o jsonpath="{.items[0].status.addresses[0].address}")
    export JUPYTER_POD=$(kubectl get pods -n jupyter | awk '/jupyter/ {print $2}')
    export JUPYTER_TOKEN=$(kubectl exec -n jupyter -it "$JUPYTER_POD" -- sh -c 'jupyter notebook list | tail -1 | cut -d":" -f3 | cut -d"=" -f2')
    echo http://"$JUPYTER_IP?token=$JUPYTER_TOKEN"

    To delete this deployment, run helm uninstall -n jupyter jupyter-notebook

Do as the NOTES.txt output says.

    $ export JUPYTER_IP=$(kubectl get nodes -n jupyter -o jsonpath="{.items[0].status.addresses[0].address}")
    export JUPYTER_POD=$(kubectl get pods -n jupyter | awk '/jupyter/ {print $2}')
    export JUPYTER_TOKEN=$(kubectl exec -n jupyter -it "$JUPYTER_POD" -- sh -c 'jupyter notebook list | tail -1 | cut -d":" -f3 | cut -d"=" -f2')
    echo http://"$JUPYTER_IP?token=$JUPYTER_TOKEN"
    http://192.168.64.17?token=285a269b871fc5e3d6d344da815aa059384ba2222a7be0b3

Note that it may take a few seconds for everything to be accessible, even though these commands will return an address.
Specifically, check the ingress with kubectl get ingress -n jupyter.
Recall that you need to have specified ingress as an argument to minikube.

### <a name = "helmpackage">Helm Packaging</a>

You can also package up your deployment, for easier use by others, pushing to a repo, etc.

    $ helm package .
    Successfully packaged chart and saved it to: /Users/sgarland/jupyter-minikube/jupyter-minikube-0.1.0.tgz

Installation can also be done from this tarball. Just remember you'll need to repackage if you make any changes.

    $ helm install --namespace jupyter jupyter-notebook ./jupyter-minikube-0.1.0.tgz
    NAME: jupyter-notebook
    LAST DEPLOYED: Fri Oct  1 13:20:09 2021
    NAMESPACE: jupyter
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    jupyter-minikube installed successfully.

    To access your notebook, run the following commands, and go to the echoed URL:

    export JUPYTER_IP=$(kubectl get nodes -n jupyter -o jsonpath="{.items[0].status.addresses[0].address}")
    export JUPYTER_POD=$(kubectl get pods -n jupyter | awk '/jupyter/ {print $2}')
    export JUPYTER_TOKEN=$(kubectl exec -n jupyter -it "$JUPYTER_POD" -- sh -c 'jupyter notebook list | tail -1 | cut -d":" -f3 | cut -d"=" -f2')
    echo http://"$JUPYTER_IP?token=$JUPYTER_TOKEN"

    To delete this deployment, run helm uninstall -n jupyter jupyter-notebook

## <a name = "Namespaces">Namespaces</a>

This was briefly alluded to, in that we created one and used it. So what are they? Essentially, if you think of Kubernetes as a Hypervisor (it isn't, but stay with me), then Namespaces are like Virtual Machines. Each one is independent, with its own resources, limits, security settings, etc. A common use case for them is to have a test/staging/production setup, so development work can proceed without disrupting production. Or, perhaps you want to give each team a certain amount of cluster resources - again, these can be granularly set.

### <a name = "Using Namespaces">Using Namespaces</a>

Let's create some more namespaces, and deploy to each. First, let's remove our existing Helm release, and the namespace we created.

    $ helm uninstall -n jupyter jupyter-notebook
    release "jupyter-notebook" deleted

    $ kubectl delete ns jupyter
    namespace "jupyter" deleted

Now, let's create three jupyter namespaces. This can also be accomplished with the `--create-namespace` argument in Helm.

    $ for i in {0..2}; do kubectl create ns jupyter-$i; done
    namespace/jupyter-0 created
    namespace/jupyter-1 created
    namespace/jupyter-2 created

And comment out the namespace reference in our templates.

    $ sed -Ei 's/(^\s+)(namespace)/\1#\2/ w /dev/stdout' */*.yaml
      #namespace: {{ .Values.service.namespace }}
      #namespace: {{ .Values.service.namespace }}
      #namespace: {{ .Values.service.namespace }}

Then deploy to all of the namespaces.

    # Using the `--create-namespace` argument from above
    for ns in jupyter-{0..2}; do helm install --namespace "$ns" --create-namespace jupyter .; done

The previous exports will work here, with the exception of getting the token - we'll have to wrap that in a loop to get all of them.

    $ for ns in jupyter-{0..2}; do kubectl exec -n "$ns" -it "$JUPYTER_POD" -- sh -c 'jupyter notebook list | tail -1 | cut -d":" -f3 | cut -d"=" -f2'; done
    dfbeefd51ad3292cd06d8dd053b85c7522d1a32be02fe558
    Error from server (NotFound): pods "jupyter-notebook-fb887b49d-6cmsz" not found
    Error from server (NotFound): pods "jupyter-notebook-fb887b49d-6cmsz" not found

Why did the other two fail? Let's see:

    $ kubectl get events -n jupyter-1
    LAST SEEN   TYPE      REASON              OBJECT                                  MESSAGE
    9m13s       Warning   FailedScheduling    pod/jupyter-notebook-fb887b49d-6np57    0/1 nodes are available: 1 Insufficient cpu.

Remember when we created the minikube cluster, we only allocated 2 CPUs (it's the default)? Since our deployment is requesting 1 CPU for each, and Kubernetes needs some for itself, this won't work out. But wait, we could scale the replicas way past 2! 

FILL IN HERE

There are at least two ways we could address this. We could scale down the CPU requests (and add a limit for good measure), or we could add resource quotas to the namespaces.

## <a name = "cleanup">Cleanup</a>

If you used helm, run the command given in the NOTES.txt output.

    $ helm uninstall -n jupyter jupyter-notebook
    release "jupyter-notebook" deleted

If you didn't use helm, run these commands.

    # Technically not required since we're going to delete everything.
    $ kubectl scale --replicas 0 deployment jupyter-notebook -n jupyter
    deployment.extensions/jupyter-notebook scaled

    # Again, technically not required.
    $ kubectl delete pods,svc,ingress,deployment --all -n jupyter
    service "jupyter-notebook" deleted
    ingress.extensions "jupyter-notebook" deleted
    deployment.extensions "jupyter-notebook" deleted

    # This wipes out everything.
    $ minikube delete
    🔥  Deleting "minikube" in hyperkit ...
    💀  Removed all traces of the "minikube" cluster.

## <a name = "final">Final Thoughts</a>

There is much more to k8s than this can touch on. While this uses namespaces, it doesn't really explore them, or discuss switching between them. Additionally, all production k8s apps will be utilizing load balancing, which to my knowledge minikube doesn't support (although perhaps with an nginx or haproxy as the publicly-exposed service, one could manage). I hope this provides enough of a background to get you started, however.

Finally, I highly recommend the  [kubectl plugin](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/kubectl)  for oh-my-zsh if you're using zsh, or at least, making some aliases in your .bashrc/.zshrc file. The plugin makes many of these commands much shorter.
