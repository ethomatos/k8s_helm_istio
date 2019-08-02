# Introduction
These are step by step detailed instructions that will guide you to start with nothing and end up with a Kubernetes 2 node cluster using all the following items:


  1. ***Installing Kubernetes and Helm***
   - with role based access control (RBAC)
   - with Tiller and Helm
   - different namespaces


  2. ***Installing Istio***
   - using Helm
   - using side-cars and understanding Istio side-car injection
  
  
  3. ***Datadog***
   - installation using a Helm chart
   - incorporating and deploying the use of the Cluster agent
  
  
  4. ***Deploy a three-tiered application to test Datadog APM with Istio side-car injection***
   - Using Springboot, Flask and Postgres containers in a Kubernetes cluster
   - based on Ziquan's [workshop](https://github.com/ziquanmiao/kubernetes_datadog)
 
 
# The High Level Steps
*Requirement*: Come empty-handed, with nothing but the desire to build from scratch.


1. ***The nodes***
  - We will need two systems to use and do all our work on. 
  - This entire exercise is based on using the Ubuntu 18.04 image.
  - Use your favorite cloud provider or on-prem environment to acquire two Ubuntu servers.
  - Make sure that each system has at least 2 CPUs and 4GB RAM.


2. ***Kubernetes***
  - We will install the Kubernetes software and all its dependencies onto the systems.
  - Disable swap space on the systems.
  - The Ubuntu Linux operating system's `apt-get` command line interface will be used.
  - Install a container network interface (CNI) plug-in for Kubernetes pods to communicate.
    - Excellent reference on the [Kubernetes Network Model](https://github.com/containernetworking/cni).
  - Set up the cluster and check.
  - Understand how to `taint` and use tainting to make the manager also function as a worker node.


3. ***Kubernetes role based access control (RBAC)***
  - RBAC comes built-into the Kubernetes cluster.
  - Understand that it is there and how to check that it is there.


4. ***Tiller and Helm***
  - What is Tiller? What is Helm? The work in this section is based all on this [guide](https://helm.sh/docs/using_helm/).
  - Helm is composed of the client (`helm`) and the server (`Tiller`), we will initialize Helm and set both up.
  - The installation of Helm will incorporate Kubernetes RBAC so the installation will be custom to support RBAC.


5. ***Istio and Helm***
  - [Istio](https://istio.io/) introduces service network mesh technology to your Kubernetes cluster.
  - Istio will be installed taking into account Helm's Tiller (_the server part of Helm_) and Kubernetes RBAC.
  - Check that Istio is installed and understand [side car injection](https://istio.io/docs/setup/kubernetes/additional-setup/sidecar-injection/) since using a service mesh means having a side-car enabled for every pod.


6. ***Datadog and Helm***
  - Why do this? Why bother using Helm? It orchestrates all Kubernetes tasks for you in *one command*.
    - Rather than use the daemonset approach, use Helm to automate almost everything :).
  - The files needed to setup and configure Datadog come from a [Helm chart Github repository](https://github.com/helm/charts/tree/master/stable/datadog#configuration).
  - Configure the manifest file for datadog and install the chart enabling the Datadog cluster agent.
  - Confirm the Helm chart was installed, test and check the Datadog agents.
 
 
7. ***Datadog APM with Springboot***
  - This is based on a [workshop](https://github.com/ziquanmiao/kubernetes_datadog) from Ziquan.
  - In this section the Github repository will be cloned and then changes will be made to the manifest files.
  - Create a separate Kubernetes namespace for the APM application testing and enable Istio side-car injection. 
  - After updating manifest files for postgres, Flask and Springboot run the containers inside the new namespace.
  - Confirm and check the Springboot application.


8. ***Test end-to-end inside the Datadog platform***
  - Execute tests using `curl` against the Springboot application.
  - Use `tcpdump` to check traffic between the Istio side-car injection namespace and the Datadog agents.


# Step 1: The Nodes
  
  
  #### Pre-requisites
  - Acquire two Ubuntu 18.04 Linux server instances. In this exercise, AWS EC2 instances were used with the following specifications:
    - A instance type of t2.medium 
    - An Ubuntu version 18 image
    - 2 CPUs and 4GB RAM
  - Set up security such that SSH access is enabled on both systems and communication on any TCP port is allowed between them.
  - One server will function as a master and the second as the worker.
  - We will enable the master to also run workloads just like a worker node.
  
  
# Step 2: Kubernetes
  
  
  #### Install Docker  
  - Kubernetes orchestrates containers but first, software is needed that creates containers, hence the need for Docker.
  - On each Linux system run these commands:
    1. First update the repository of packages that Ubuntu's "advanced packaging tool" (apt) has access to for installation.
      - `apt-get update`
    2. Install Docker as root.
      - `sudo apt install docker.io`
    3. Enable Docker and force it to always start up after any time the systems reboot.
      - `sudo systemctl start docker` then run `sudo systemctl enable docker`
  
  
  #### Disable swap then reboot
  - Disable swap on both systems by running this command as root:
    - `sudo swapoff -a`
  - Reboot each server, use the command `reboot`
  
  
  #### Check Docker
  - Check that Docker is installed and running. Run this command:
    - `docker version`
  
  
  #### Install and start up Kubernetes
  - Allow your Ubuntu server's apt tool to use HTTPS to access and install repositories. Run this command from both servers.
    - `sudo apt install apt-transport-https`  
  - Add the Kubernetes key, run this command on both servers:
    - `curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -`  
  - Add the Kubernetes official repository into your apt utilities list of repositories. Run this command from both servers.
    - `sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"`  
  - Now install the Kubernetes packages onto both servers, run these commands from both servers.
    - First update the repository: `sudo apt update`
    - Next install the packages: `sudo apt install kubeadm kubelet kubectl`
  - Check that the installation worked, run these commands from both servers.
    - Check the version of Kubernetes: `kubeadm version` 
  - Set up a new Kubernetes cluster on your two nodes. First initialize the cluster, run this command from your master node. One of the two nodes you can designate as your master, the second will be a worker node.
    - Run the command from the master node: `sudo kubeadm init`
  - Join the worker node to the cluster. At the very end of the above command will be a set of commands that need to be run to allow the worker node to join the cluster. It will look similar to this output:
    - `kubeadm join a.b.c.d:6443 --toker a_token --discovery-token-ca-cert-hash sha256:a_long_hash`
    - Run the above command from the resulting output ****on your secondary worker node* server**** as root.
  - Set up the non-root user on your Linux master node to be able to use the new cluster. Also at the very end of the above `kubeadm init` command is a set of commands, listed below, that need to be run on the master node.
    - As the regular user (for example, _ubuntu_) run the commands:
      - `mkdir -p $HOME/.kube`
      - `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`
      - `sudo chown $(id -u):$(id -g) $HOME/.kube/config`  
  - Confirm that the secondary worker node has joined your new cluster. Run the following command from the master node:
    - `kubectl get nodes` This should print out the list showing your two nodes being part of the cluster.
    - ***You are not done yet, the two nodes will not be ready yet until the next step***.
  - Install a [container network interface plugin](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network) (CNI plugin) for your new Kubernetes cluster to use. Run this command from the master node. In this exercise Calico was used.
    - `kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml`
  - After installation of a container network interface (CNI) plugin re-run the command `kubectl get nodes` and confirm that both nodes are now in a `Ready` state.
  - By default, the master node in the cluster will not function as a worker node. To enable workloads on the master run the following command from the master node:
    - `kubectl taint nodes master-node node-role.kubernetes.io/master-`
    - Nodes can be tainted to provide custom behaviour, the above command removes the taint hence the `-` at the end.
    - An example output from the command showing both nodes in the cluster in the ready state is shown below:  
    
    >ubuntu@master-node:~$ kubectl get nodes  
    >NAME          STATUS   ROLES    AGE   VERSION  
    >master-node   Ready    master   13h   v1.15.1  
    >slave-node    Ready    <none>   13h   v1.15.1  
  
 
 # Step 3: Kubernetes and RBAC
 
 
   #### Where is the role based access control (RBAC) in your cluster?
   - Kubernetes uses deployment manifest files (yaml) to define the objects (`kind:` keyword) that need to be created.
   - To interact with each object there is an API with a version.
   - RBAC is just another object.
   - To list all the API versions for each object run the following command: `kubectl api-versions`
   - One of the APIs for interacting with the RBAC object within your Kubernetes cluster will be listed, an example is below:  
   >ubuntu@master-node:~$ kubectl api-versions | grep rbac  
   >rbac.authorization.k8s.io/v1  
   >rbac.authorization.k8s.io/v1beta1  
   
   
# Step 4: Tiller and Helm


  #### Helm's components
  - All details listed here are based on this [guide](https://helm.sh/docs/using_helm/).
  - We need to install Helm on our master node. Helm has a client, `helm` and a server, `tiller`.
  - Since your Kubernetes cluster has RBAC enabled Helm must be installed in a way that supports RBAC.
  
  
  #### Install Helm
  - On your master node run the command `sudo snap install helm`.
  - After installing check that it exists by getting its version. Example output is shown below:  
  
  >ubuntu@master-node:~$ helm version  
  >Client: &version.Version{SemVer:"v2.14.2", GitCommit:"a8b13cc5ab6a7dbef0a58f5061bcc7c0c61598e7", GitTreeState:"clean"}  
  >Error: could not find tiller  
  
  - The error is fine. The Helm server component `tiller` will be installed upon initializing Helm for the first time.
  
  
  #### Initialize Helm's Tiller using RBAC
  - Your cluster's RBAC limits what Helm can do. For Helm to be able to orchestrate inside your cluster:
    1. Tiller needs to be considered a cluster administrator.
    2. Cluster administrator permissions are granted to a role which has a service account name.
    3. A role-binding in a Kubernetes cluster can be used to specify a service account name with permissions from a role.
    4. This section is based on details from this [guide](https://helm.sh/docs/using_helm/#role-based-access-control).
  - Either create a file called `rbac-config.yaml` as specified in the guide or use the file from this repository. Run this command: `kubectl create -f rbac-config.yaml`
    - Example output from the command will look like this:  
    >ubuntu@master-node:~/git/k8s_istio$ kubectl create -f rbac-config.yaml  
    >serviceaccount/tiller created  
    >clusterrolebinding.rbac.authorization.k8s.io/tiller created  
  - Now your cluster has a role-binding which can be used to initialize Tiller using its own cluster admin service account.
  - Run the command `helm init --service-account tiller` to initialize Helm and install Tiller. Example output looks like this:  
    >ubuntu@master-node:~$ helm init --service-account tiller  
    >$HELM_HOME has been configured at /home/ubuntu/.helm.  
    >Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.
  - Check that Helm is fully installed, run the commend `kubectl get pods --all-namespaces` and look at all the pods running in the kube-system namespace. One of the pods is Tiller. Example output from the command is shown below:  
    >ubuntu@master-node:~$ kubectl get pods --all-namespaces  
    >NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE  
    >kube-system   calico-kube-controllers-7bd78b474d-pxjl4   1/1     Running   0          19h  
    >kube-system   calico-node-g6v64                          1/1     Running   0          19h  
    >kube-system   calico-node-vt4rf                          1/1     Running   0          19h  
    >kube-system   coredns-5c98db65d4-8487j                   1/1     Running   0          19h  
    >kube-system   coredns-5c98db65d4-rzgz5                   1/1     Running   0          19h  
    >kube-system   etcd-master-node                           1/1     Running   0          19h  
    >kube-system   kube-apiserver-master-node                 1/1     Running   0          19h  
    >kube-system   kube-controller-manager-master-node        1/1     Running   0          19h  
    >kube-system   kube-proxy-6xp6b                           1/1     Running   0          19h  
    >kube-system   kube-proxy-n8m8n                           1/1     Running   0          19h  
    >kube-system   kube-scheduler-master-node                 1/1     Running   0          19h  
    >***kube-system   tiller-deploy-767d9b9584-2gwbd             1/1     Running   0          15m***


# Step 5: Istio and Helm


  #### Install Istio using Helm
  - All details for the steps below were gathered from this [document](https://istio.io/docs/setup/kubernetes/#downloading-the-release).
  - On your master node, go to your home directory and run this command to download the Istio release:  
    - `curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.2.2 sh -`
  - Once downloaded, run this command `cd istio-1.2.2/` to navigate to the istio release directory.
  - This installation exercise will follow the procedure in this [section](https://istio.io/docs/setup/kubernetes/install/helm/#option-2-install-with-helm-and-tiller-via-helm-install) of the Istio installation.  
    - Since you have already granted Tiller a service account with the cluster-admin role begin installation by running this command (this will install all custom resources definitions (CRDs) required by Istio within your Kubernetes cluster):  
      - `helm install install/kubernetes/helm/istio-init --name istio-init --namespace istio-system`
      - Confirm the Helm installation by running the command below to check that there are 23 custom resources definitions (CRDs). For more details about what CRDs are and why they are important refer to this [document](https://docs.okd.io/latest/admin_guide/custom_resource_definitions.html).
        - `kubectl get crds | grep 'istio.io\|certmanager.k8s.io' | wc -l`   
    - To set up the environment for Datadog APM application instrumentation with Istio the final step is to install Istio using a Helm chart, running the following command which is a ***custom Istio installation*** requiring two extra options, based on this [document](https://docs.datadoghq.com/tracing/setup/istio/#istio-configuration-and-installation), that get passed to the `helm install` step below.:  
      - `helm install install/kubernetes/helm/istio --name istio --namespace istio-system --set pilot.traceSampling=100.0 --set global.proxy.tracer=datadog`  
    - After installing Istio using Tiller with the command above, there will be a lot of output but at the very end the following message will appear:  
      >NOTES:  
      >Thank you for installing istio.  
      >Your release is named istio.  
      >  
      >To get started running application with Istio, execute the following steps:  
      >1. Label namespace that application object will be deployed to by the following command (take default namespace as an example)  
      >$ kubectl label namespace default istio-injection=enabled  
      >$ kubectl get namespace -L istio-injection  
      >2. Deploy your applications  
      >$ kubectl apply -f <your-application>.yaml  
    - Verify the Istio installation by running this command `kubectl get svc -n istio-system`  
      - A list of services will appear which will include:  
        - istio-citadel, istio-galley, istio-ingressgateway, istio-pilot  
        - istio-policy, istio-sidecar-injector, istio-telemetry and prometheus
    - Confirm that Istio related pods are deployed in your cluster by running this command:  
      - `kubectl get pods -n istio-system`


# Step 6: Datadog and Helm


  #### Get the Datadog Helm chart from Github
  - Based on information from our [documentation](https://docs.datadoghq.com/agent/kubernetes/helm/?tab=macoshomebrew#installing-the-datadog-helm-chart), clone the following repository onto your master node.
    - More details about the chart is on github [here](https://github.com/helm/charts/tree/master/stable/datadog#configuration). On your master node, run the command:  
      - `git clone https://github.com/helm/charts.git`
    - Navigate inside the newly created directory using the command `cd helm/charts/stable/datadog/`  


  #### Install the Datadog Helm chart
  - Use the file `datadog-values.yaml` found in this repository instead of the `values.yaml` file found in the Helm chart repository for Datadog.  
  - Run the following command to install the Datadog cluster agent and Datadog agent along with all configurations. *Just one command that does it all* (This is why we use Helm).  
    - `helm install --name datadog-monitoring -f datadog-values.yaml stable/datadog`  
      - After running the command above to install the Datadog solution into your Istio enabled cluster, the following output will appear right at the top:  
      >NAME:   datadog-monitoring  
      >LAST DEPLOYED: Wed Jul 31 20:15:42 2019  
      >NAMESPACE: default  
      >STATUS: DEPLOYED


  #### Confirm the Datadog Helm chart was installed
  - After installing the Datadog Helm chart, confirm that all the services were deployed in your cluster including all the Datadog pods.
  - Run the following command to see if the Datadog cluster agent services were deployed:  
    - `kubectl get svc`
      - The following output should appear:  
      > ubuntu@master-node:~/helm/charts/stable/datadog$ kubectl get svc  
      > NAME                                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE  
      > datadog-monitoring-cluster-agent               ClusterIP   10.102.188.171   <none>        5005/TCP   6m20s  
      > datadog-monitoring-cluster-agent-metrics-api   ClusterIP   10.97.113.9      <none>        443/TCP    6m20s  
      > datadog-monitoring-kube-state-metrics          ClusterIP   10.107.171.57    <none>        8080/TCP   6m20s  
      > kubernetes                                     ClusterIP   10.96.0.1        <none>        443/TCP    21h  
  - Run the following command to see if the Datadog pods were deployed:  
    - `kubectl get pods`
      - The following output should appear:  
      > ubuntu@master-node:~/helm/charts/stable/datadog$ kubectl get pods  
      > NAME                                                     READY   STATUS    RESTARTS   AGE  
      > datadog-monitoring-8fprj                                 1/1     Running   0          6m27s  
      > datadog-monitoring-cluster-agent-95d7c8c8-x6dph          1/1     Running   0          6m27s  
      > datadog-monitoring-h4qpf                                 1/1     Running   0          6m27s  
      > datadog-monitoring-kube-state-metrics-65c499c65d-q2pm9   1/1     Running   0          6m27s  


  #### Check the Datadog cluster agent 
  - Based on the output seen from the `kubectl get pods` command, run the following kubernetes commands to execute status commands from outside the pods.
  - To check the Datadog cluster agent use:  
    - `kubectl exec -it datadog-monitoring-cluster-agent-95d7c8c8-x6dph datadog-cluster-agent status`
  - To check the Datadog cluster agent configuration and ensure the Istio check is enabled use:  
    - `kubectl exec -it datadog-monitoring-cluster-agent-95d7c8c8-x6dph datadog-cluster-agent configcheck`


  #### Check the Datadog agents 
  - Based on the output seen from the `kubectl get pods` command, run the following kubernetes commands to execute status commands from outside the pods.
  - To check the Datadog agents use this command against each pod running the Datadog agent:  
    - `kubectl exec -it datadog-monitoring-h4qpf agent status`
  - To check the Datadog agent configurations use this command against each pod running the Datadog agent:  
    - `kubectl exec -it datadog-monitoring-h4qpf agent configcheck`
  - The results from the above commands should show all the integrations running, including one for `istio` and the configurations for each integrations.


# Step 7: Datadog APM with Springboot deployed in Istio enabled Kubernetes cluster


  #### This entire exercise is based on Ziquan's Github training workshop [repository](https://github.com/ziquanmiao/kubernetes_datadog).  
  - The workshop is extended to run it inside an Istio enabled Kubernetes cluster with side-car injection.
  - The idea is to test APM and see how traces traverse your new cluster's network mesh using Istio.


  #### Use the files in the apm/ directory which was pulled when this repository was cloned.
  - Go inside the apm/ directory and notice there are three manifest files used to build Zi's 3-tier Springboot Java applications:  
    - postgres_deployment-training.yaml --> This is used to create the Postgres database portion of the application.
    - flask_deployment-training.yaml --> This is used to create the Python Flask part of the application.
    - springboot_deployment-training.yaml --> This is used to create the Java Springboot application.


  #### Istio side-car injection
  - Kubernetes can be configured with namespaces. Istio can be configured to be the network mesh overlay for any specified namespace.
  - The above 3-tier Springboot application will be deployed inside a new, Istio enabled namespace in your Kubernetes cluster.
  - Run the following command to see if there are any namespaces in your cluster that have Istio injection enabled:  
    - `kubectl get namespace -L istio-injection`
      - The above command will display a list of all the namespaces and a `ISTIO-INJECTION` column will appear with a value of `enabled` for any namespaces enabled with Istio injection. An example output is shown below:  
      > ubuntu@master-node:~/git/k8s_istio/apm$ kubectl get namespace -L istio-injection  
      > NAME              STATUS   AGE    ISTIO-INJECTION  
      > default           Active   22h      
      > istio-system      Active   3h1m     
      > kube-node-lease   Active   22h      
      > kube-public       Active   22h      
      > kube-system       Active   22h      
  - Create a new namespace to deploy the 3-tier application into and enable Istio side-car injection.
    - Run the command `kubectl create namespace training`  
      - The resulting output will be this:  
      > ubuntu@master-node:~/git/k8s_istio/apm$ kubectl create namespace training  
      > namespace/training created
    - Next run the following command to enable Istio side-car injection inside this new `training` namespace.
      - `kubectl label namespace training istio-injection=enabled`  
        - The resulting output will be this:  
        > ubuntu@master-node:~/git/k8s_istio/apm$ kubectl label namespace training istio-injection=enabled  
        > namespace/training labeled
  - Confirm the newly created namespace has injection enabled, run the following command:
    - `kubectl get namespace -L istio-injection`  
      - Notice the new `training` namespace has its *ISTIO-INJECTION* column value now showing "enabled"  
      > ubuntu@master-node:~/git/k8s_istio/apm$ kubectl get namespace -L istio-injection  
      > NAME              STATUS   AGE    ISTIO-INJECTION  
      > default           Active   22h      
      > istio-system      Active   3h1m     
      > kube-node-lease   Active   22h      
      > kube-public       Active   22h      
      > kube-system       Active   22h        
      > training          Active   3m25s   ***enabled***


  #### Deploy the three-tier Springboot application into the `training` Istio side-car injection enabled namespace
  - Navigate to the apm/ folder of this repository which consists of the three yaml files for the application listed above.
  - Run the following commands to deploy each component of the application into the `training` namespace:
    - First install postgres:
      - Run `kubectl -n training create -f postgres_deployment-training.yaml`
    - Second install flask:
      - Run `kubectl -n training create -f flask_deployment-training.yaml`
    - Third install Springboot:
      - Run `kubectl -n training create -f springboot_deployment-training.yaml`
  - Confirm that the application is deployed in the `training` namespace, run the following command:
    - `kubectl -n training get pods`
      - The resulting output should look like this, notice the ***2/2*** output? There is one pod for the application and the second is the Istio injected side-car:  
      > NAME                                     READY   STATUS    RESTARTS   AGE  
      > flaskapp-testing-7d7dfdb9db-4gfwd        2/2     Running   1          111s  
      > flaskapp-testing-7d7dfdb9db-6qktl        2/2     Running   2          111s  
      > postgres-testing-7f67f4495d-4rz28        2/2     Running   0          2m56s  
      > springbootapp-testing-786f48d8db-kjxwb   2/2     Running   0          66s  


  #### Test the Springboot application and use tcpdump to capture traffic from the application pods
  - Connect to the Flask app pod, install net-tools and tcpdump.
    - First find the name of the pod, use `kubectl -n training get pods`
    - Connect to the pod using `kubectl exec -it <pod_name> bash`
    - While in the pod, run the command as root `apt-get update`
    - After updating apt, install net-tools which provides ifconfig, use `apt-get install net-tools`
    - Once the installation completed, check it by running `ifconfig -a` and identify your main interface which typically is ***eth0***
    - Now install tcpdump, run the command `apt-get install tcpdump`
    - Use tcpdump to look for packets coming into or out of your pod, there are many, many options to tcpdump, start with this simple one:
      - `tcpdump -i eth0 port 8126` (This will show any packets coming into your pod's eth0 interface, with a TCP port of 8126)
  - Run the following command to get the IP address of the services for the application:
    - `kubectl -n training get svc`
  - Find the value of the `CLUSTER-IP` for the flaskapp and use that to run these two tests:
    - `curl 10.105.80.139:5000/`
      - The output on the screen should be _Flask has been kuberneted_
    - `curl 10.105.80.139:5000/query`
      - The output on the screen should be _(u'1', u'dog@datadog.com')_


