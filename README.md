# learn-Kubernetes

-  https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/


First, set up the Docker and Kubernetes repositories:

          curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

          sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \

          $(lsb_release -cs) \
          stable"

          curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

          cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
          deb https://apt.kubernetes.io/ kubernetes-xenial main
         EOF


Install Docker and Kubernetes packages:

Note that if you want to use a newer version of Kubernetes, change the version installed for kubelet, kubeadm, and kubectl. Make sure all three use the same version.

Note: There is currently a bug in Kubernetes 1.13.4 (and earlier) that can cause problems installaing the packages. Use 1.13.5-00 to avoid this issue.

        sudo apt-get update

       sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu kubelet=1.13.5-00 kubeadm=1.13.5-00 kubectl=1.13.5-00

      sudo apt-mark hold docker-ce kubelet kubeadm kubectl
      
      
Enable iptables bridge call:

         echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf

         sudo sysctl -p


On the Kube master server
Initialize the cluster:

       sudo kubeadm init --pod-network-cidr=10.244.0.0/16

Set up local kubeconfig:

      mkdir -p $HOME/.kube

      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

      sudo chown $(id -u):$(id -g) $HOME/.kube/config


Install Flannel networking:

       kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-   flannel.yml

On each Kube node server
Join the node to the cluster:

        sudo kubeadm join $controller_private_ip:6443 --token $token --discovery-token-ca-cert-hash $hash

On the Kube master server
Verify that all nodes are joined and ready:

kubectl get nodes
You should see all three servers with a status of Ready:

        NAME                      STATUS   ROLES    AGE   VERSION
        wboyd1c.mylabserver.com   Ready    master   54m   v1.13.4
        wboyd2c.mylabserver.com   Ready    <none>   49m   v1.13.4
        wboyd3c.mylabserver.com   Ready    <none>   49m   v1.13.4


----------------------------------------------------------------


            kubectl api-resources -o name

            kubectl get pods -n kube-system

            kubectl get nodes

            kubectl get nodes $node_name

            kubectl get nodes $node_name -o yaml

            kubectl describe node $node_name

-------------------------------------------------------------

Pods are one of the most essential Kubernetes object types. Most of the orchestration features of Kubernetes are centered around the management of Pods. In this lesson, we will discuss what Pods are and demonstrate how to create a pod. We will also talk about how to edit and delete pods after they are created. The principles discussed in this lesson for managing pods apply to the management of other types of Kubernetes objects as well.


- https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/



Create a new yaml file to contain the pod definition. Use whatever editor you like, but we used vi:

      vi my-pod.yml
      
      
my-pod.yml:


			apiVersion: v1
			kind: Pod
			metadata:
			  name: my-pod
			  labels:
				app: myapp
			spec:
			  containers:
			  - name: myapp-container
				image: busybox
				command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']




Create a pod from the yaml definition file:

        kubectl create -f my-pod.yml
        
Edit a pod by updating the yaml definiton and re-applying it:

      kubectl apply -f my-pod.yml
      
You can also edit a pod like this:

       kubectl edit pod my-pod
       
You can delete a pod like this:

     kubectl delete pod my-pod
     
     
-----------------------------------------------------
- https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/


You can get a list of the namespaces in the cluster like this:

         kubectl get namespaces
	 
You can also create your own namespaces.

kubectl create ns my-ns
To assign an object to a custom namespace, simply specify its metadata.namespace attribute.

		apiVersion: v1
		kind: Pod
		metadata:
		name: my-ns-pod
		namespace: my-ns
		labels:
		app: myapp
		spec:
		containers:
		- name: myapp-container
		image: busybox
		command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']

    
Create the pod with the created yaml file.

      kubectl create -f my-ns.yml
      
Use the -n flag to specify a namespace when using commands like kubectl get.

     kubectl get pods -n my-ns
     
You can also use -n to specify a namespace when using kubectl describe.



kubectl describe pod my-ns-pod -n my-ns


-------------------------------------------
- https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/


---------------------------------------------


Management of configuration data is one of the challenges involved in building and maintaining complex application infrastructures. Luckily, Kubernetes offers functionality that helps to maintain application configurations in the form of ConfigMaps. In this lesson, we will discuss what ConfigMaps are, how to create them, some of the ways that ConfigMap data can be passed in to containers running within Kubernetes Pods.

- https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/

Here's an example of of a yaml descriptor for a ConfigMap containing some data:



		apiVersion: v1
		kind: ConfigMap
		metadata:
		   name: my-config-map
		data:
		   myKey: myValue
		   anotherKey: anotherValue


Passing ConfigMap data to a container as an environment variable looks like this:

		apiVersion: v1
		kind: Pod
		metadata:
		  name: my-configmap-pod
		spec:
		  containers:
		  - name: myapp-container
			image: busybox
			command: ['sh', '-c', "echo $(MY_VAR) && sleep 3600"]
			env:
			- name: MY_VAR
			  valueFrom:
				configMapKeyRef:
				  name: my-config-map
				  key: myKey



It's also possible to pass ConfigMap data to containers, in the form of file using a mounted volume, like so:

			apiVersion: v1
			kind: Pod
			metadata:
			  name: my-configmap-volume-pod
			spec:
			  containers:
			  - name: myapp-container
				image: busybox
				command: ['sh', '-c', "echo $(cat /etc/config/myKey) && sleep 3600"]
				volumeMounts:
				  - name: config-volume
					mountPath: /etc/config
			  volumes:
				- name: config-volume
				  configMap:
					name: my-config-map

In the lesson, we'll also use the following commands to explore how the ConfigMap data interacts with pods and containers:



		kubectl logs my-configmap-pod

		kubectl logs my-configmap-volume-pod

		kubectl exec my-configmap-volume-pod -- ls /etc/config

		kubectl exec my-configmap-volume-pod -- cat /etc/config/myKey
		
----------------------------------------------------------

Occasionally, it's necessary to customize how containers interact with the underlying security mechanisms present on the operating systems of Kubernetes nodes. The securityContext attribute in a pod specification allows for making these customizations. In this lesson, we will briefly discuss what the securityContext is, and demonstrate how to use it to implement some common functionality.


- https://kubernetes.io/docs/tasks/configure-pod-container/security-context/


First, create some users, groups, and files on both worker nodes which we can use for testing.


			sudo useradd -u 2000 container-user-0
			sudo groupadd -g 3000 container-group-0
			sudo useradd -u 2001 container-user-1
			sudo groupadd -g 3001 container-group-1
			sudo mkdir -p /etc/message/
			echo "Hello, World!" | sudo tee -a /etc/message/message.txt
			sudo chown 2000:3000 /etc/message/message.txt
			sudo chmod 640 /etc/message/message.txt


On the controller, create a pod to read the message.txt file and print the message to the log.


      vi my-securitycontext-pod.yml

Content of the YAML File


			apiVersion: v1
			kind: Pod
			metadata:
			  name: my-securitycontext-pod
			spec:
			  containers:
			  - name: myapp-container
				image: busybox
				command: ['sh', '-c', "cat /message/message.txt && sleep 3600"]
				volumeMounts:
				- name: message-volume
				  mountPath: /message
			  volumes:
			  - name: message-volume
				hostPath:
				  path: /etc/message



Check the pod's log to see the message from the file:


    kubectl logs my-securitycontext-pod

Delete the pod and re-create it, this time with a securityContext set to use a user and group that do not have access to the file.


    kubectl delete pod my-securitycontext-pod --now



			apiVersion: v1
			kind: Pod
			metadata:
			  name: my-securitycontext-pod
			spec:
			  securityContext:
				runAsUser: 2001
				fsGroup: 3001
			  containers:
			  - name: myapp-container
				image: busybox
				command: ['sh', '-c', "cat /message/message.txt && sleep 3600"]
				volumeMounts:
				- name: message-volume
				  mountPath: /message
			  volumes:
			  - name: message-volume
				hostPath:
				  path: /etc/message


Check the log again. You should see a "permission denied" message.

    kubectl logs my-securitycontext-pod

Delete the pod and re-create it again, this time with a user and group that are able to access the file.


      kubectl delete pod my-securitycontext-pod --now
      
      
      
      			apiVersion: v1
			kind: Pod
			metadata:
			  name: my-securitycontext-pod
			spec:
			  securityContext:
				runAsUser: 2000
				fsGroup: 3000
			  containers:
			  - name: myapp-container
				image: busybox
				command: ['sh', '-c', "cat /message/message.txt && sleep 3600"]
				volumeMounts:
				- name: message-volume
				  mountPath: /message
			  volumes:
			  - name: message-volume
				hostPath:
				  path: /etc/message


Check the log once more. You should see the message from the file.


    kubectl logs my-securitycontext-pod
 


------------------------------------------

Kubernetes is a powerful tool for managing and utilizing available resources to run containers. Resource requests and limits provide a great deal of control over how resources will be allocated. In this lesson, we will talk about what resource requests and limits do, and also demonstrate how to set resource requests and limits for a container.


- https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container

Specify resource requests and resource limits in the container spec like this:

		apiVersion: v1
		kind: Pod
		metadata:
		  name: my-resource-pod
		spec:
		  containers:
		  - name: myapp-container
			image: busybox
			command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
			resources:
			  requests:
				memory: "64Mi"
				cpu: "250m"
			  limits:
				memory: "128Mi"
				cpu: "500m"



-------------------------------------------------------

One of the challenges in managing a complex application infrastructure is ensuring that sensitive data remains secure. It is always important to store sensitive data, such as tokens, passwords, and keys, in a secure, encrypted form. In this lesson, we will talk about Kubernetes secrets, a way of securely storing data and providing it to containers. We will also walk through the process of creating a simple secret, and passing the sensitive data to a container as an environment variable.


- https://kubernetes.io/docs/concepts/configuration/secret/

Create a secret using a yaml definition like this. It is a good idea to delete the yaml file containing the sensitive data after the secret object has been created in the cluster.

		apiVersion: v1
		kind: Secret
		metadata:
		  name: my-secret
		stringData:
		  myKey: myPassword


Once a secret is created, pass the sensitive data to containers as an environment variable:



			apiVersion: v1
			kind: Pod
			metadata:
			  name: my-secret-pod
			spec:
			  containers:
			  - name: myapp-container
				image: busybox
				command: ['sh', '-c', "echo Hello, Kubernetes! && sleep 3600"]
				env:
				- name: MY_PASSWORD
				  valueFrom:
					secretKeyRef:
					  name: my-secret
					  key: myKey



-------------------------------------------------


Kubernetes allows containers running within the cluster to interact with the Kubernetes API. This opens the door to some powerful forms of automation. But in order to ensure that this gets done securely, it is a good idea to use specialized ServiceAccounts with restricted permissions to allow containers to access the API. In this lesson, we will discuss ServiceAccounts as they pertain to pod configuration, and we will walk through the process of specifying which ServiceAccount a pod will use to connect to the Kubernetes API.


https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/
https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/


Creating a ServiceAccount looks like this:



          kubectl create serviceaccount my-serviceaccount


Use the serviceAccountName attribute in the pod spec to specify which ServiceAccount the pod should use:



			apiVersion: v1
			kind: Pod
			metadata:
			  name: my-serviceaccount-pod
			spec:
			  serviceAccountName: my-serviceaccount
			  containers:
			  - name: myapp-container
				image: busybox
				command: ['sh', '-c', "echo Hello, Kubernetes! && sleep 3600"]



---------------------------------------------

Multi-container pods provide an opportunity to enhance containers with helper containers that provide additional functionality. This lesson covers the basics of what multi-container pods are and how they are created. It also discusses the primary ways that containers can interact with each other within the same pod, as well as the three main multi-container pod design patterns: sidecar, ambassador, and adapter.

Be sure to check out the hands-on labs for this course (including the practice exam) to get some hands-on experience with implementing multi-container pods.

https://kubernetes.io/docs/concepts/cluster-administration/logging/#using-a-sidecar-container-with-the-logging-agent
https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/
https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/


Here is the YAML used to create a simple multi-container pod in the video:



			apiVersion: v1
			kind: Pod
			metadata:
			  name: multi-container-pod
			spec:
			  containers:
			  - name: nginx
				image: nginx:1.15.8
				ports:
				- containerPort: 80
			  - name: busybox-sidecar
				image: busybox
				command: ['sh', '-c', 'while true; do sleep 30; done;']


------------------------------------------

Kubernetes is often able to detect problems with containers and respond appropriately without the need for specialized configuration. But sometimes we need additional control over how Kubernetes determines container status. Kubernetes probes provide the ability to customize how Kubernetes detects the status of containers, allowing us to build more sophisticated mechanisms for managing container health. In this lesson, we discuss liveness and readiness probes in Kubernetes, and demonstrate how to create and configure them.



https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/


Here is a pod with a liveness probe that uses a command:

my-liveness-pod.yml:

			apiVersion: v1
			kind: Pod
			metadata:
			  name: my-liveness-pod
			spec:
			  containers:
			  - name: myapp-container
				image: busybox
				command: ['sh', '-c', "echo Hello, Kubernetes! && sleep 3600"]
				livenessProbe:
				  exec:
					command:
					- echo
					- testing
				  initialDelaySeconds: 5
				  periodSeconds: 5



Here is a pod with a readiness probe that uses an http request:

my-readiness-pod.yml:



			apiVersion: v1
			kind: Pod
			metadata:
			  name: my-readiness-pod
			spec:
			  containers:
			  - name: myapp-container
				image: nginx
				readinessProbe:
				  httpGet:
					path: /
					port: 80
				  initialDelaySeconds: 5
				  periodSeconds: 5



----------------------------------------------

When managing containers, obtaining container logs is sometimes necessary in order to gain insight into what is going on inside a container. Kubernetes offers an easy way to view and interact with container logs using the kubectl logs command. In this lesson, we discuss container logs and demonstrate how to access them using kubectl logs.



https://kubernetes.io/docs/concepts/cluster-administration/logging/


A sample pod that generates log output every second:

			apiVersion: v1
			kind: Pod
			metadata:
			  name: counter
			spec:
			  containers:
			  - name: count
				image: busybox
				args: [/bin/sh, -c, 'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']



Get the container's logs:



         kubectl logs counter
	 
	 
For a multi-container pod, specify which container to get logs for using the -c flag:


          kubectl logs <pod name> -c <container name>


Save container logs to a file:

       kubectl logs counter > counter.log
       
       
-----------------------------------

Monitoring is an important part of managing any application infrastructure. In this lesson, we will discuss how to view the resource usage of pods and nodes using the kubectl top command.

https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/


Here are some sample pods that can be used to test kubectl top. They are designed to use approximately 300m and 100m CPU, respectively.


apiVersion: v1
kind: Pod
metadata:
  name: resource-consumer-big
		spec:
		  containers:
		  - name: resource-consumer
			image: gcr.io/kubernetes-e2e-test-images/resource-consumer:1.4
			resources:
			  requests:
				cpu: 500m
				memory: 128Mi
		  - name: busybox-sidecar
			image: radial/busyboxplus:curl
			command: [/bin/sh, -c, 'until curl localhost:8080/ConsumeCPU -d "millicores=300&durationSec=3600"; do sleep 5; done && sleep 3700']

--	
		apiVersion: v1
		kind: Pod
		metadata:
		  name: resource-consumer-small
		spec:
		  containers:
		  - name: resource-consumer
			image: gcr.io/kubernetes-e2e-test-images/resource-consumer:1.4
			resources:
			  requests:
				cpu: 500m
				memory: 128Mi
		  - name: busybox-sidecar
			image: radial/busyboxplus:curl
			command: [/bin/sh, -c, 'until curl localhost:8080/ConsumeCPU -d "millicores=100&durationSec=3600"; do sleep 5; done && sleep 3700']

	
Here are the commands used in the lesson to view resource usage data in the cluster:

		kubectl top pods
		kubectl top pod resource-consumer-big
		kubectl top pods -n kube-system
		kubectl top nodes



------------------------------------

Problems will occur in any system, and Kubernetes provides some great tools to help locate and fix problems when they occur within a cluster. In this lesson, we will go through the process of debugging an issue in Kubernetes. We will use our knowledge of kubectl get and kubectl describe to locate a broken pod, and then explore various ways of editing Kubernetes objects to fix issues.




https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-pod-replication-controller/
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/



Exploring the cluster to locate the problem

		kubectl get pods

		kubectl get namespace

		kubectl get pods --all-namespaces

		kubectl describe pod nginx -n nginx-ns
		
		
Fixing the broken image name
Edit the pod:

		
		kubectl edit pod nginx -n nginx-ns

Change the container image to nginx:1.15.8.

Exporting a descriptor to edit and re-create the pod.
Export the pod descriptor and save it to a file:

		kubectl get pod nginx -n nginx-ns -o yaml --export > nginx-pod.yml

		Add this liveness probe to the container spec:

			livenessProbe:
			  httpGet:
				path: /
				port: 80

Delete the pod and recreate it using the descriptor file. Be sure to specify the namespace:

		kubectl delete pod nginx -n nginx-ns

		kubectl apply -f nginx-pod.yml -n nginx-ns









