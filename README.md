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
