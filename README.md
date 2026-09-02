## Module 10 - Container Orchestration with Kubernetes
I create a branch per chapter

### Description chapter 21 to 22 (now chapter 22 to 23 after the introduction of new chapter 11 - Gateway API)
For this demo I have two branches, one for chapter 21 and another for chapter 22

https://github.com/AstridCaballero/Module_10_Kubernetes_chapter-21_Demo/tree/Module_10/Kubernetes_chapter-21_Demo 
https://github.com/AstridCaballero/Module_10_Kubernetes_chapter-21_Demo/tree/Module_10/Kubernetes_chapter-22 


In this DEMO we configured a yaml file to deploy 11 Microservices into a Kubernetes Cluster created in Linode. Th first part of the DEMO we configure the deployment and service for each Microservice with some bad practices, the second part of the DEMO refactors the original configuration with best practices.

### Chapter 21
We configure two k8s components per microservice in a file called ‘config.yaml’
- Deployment
- Service

We gathered the information to configure each service from the google GitHub repo but in practice this information is given by the developer team. That information was:

- Microservice name
- Image URL
- Port
- Env vars like:
    - PORT
    - Connection to other MS - dependencies


We follow a template for each component and configure the first Microservice called ‘emailservice’ and configured most of the services, the same way we configured ‘emailservice’ MS.

### Specific configurations

MS connect to another MS
If there was an MS ‘A’ that needs to connect to another MS ‘B’ then we configured MS ‘B’ and then we add and env var to MS ‘A’ to reference MS ‘B’, the env var value will be “<ms_svc_name> : <ms_svc_port>”. This way MS ’A’ gets configured to know to connect to MS ‘B”. 

#### Volumes - emptyDir
Some MS also need volumes like the case of ‘redis-cart’ so we add the ‘volumes’ sub-section to the deployment’s ‘spec’ section and configure the volume to be emptyDir. emptyDir is an ephemeral volume that is linked to the pod, so if the pod dies the volumes dies with it. Because the emptyDir is linked to the Pod then it stores everything from the containers running inside the pod given those containers have mounted the volume.

The emptyDir will persist if the container dies as long as the pod is alive.

To mount the Pod’s volume into the container we add ‘volumeMounts’ to the ‘containers’ configuration and pass the name of the volume and assign a path/location of the container to mount the volume.

#### Disable profiler
Two services try to initiate ‘PROFILER’ but we don’t have it in the cluster, so it needs to be disabled from those services (payment, currency) so we add an env var called ‘DISABLED_PROFILER’ and assign the value ‘1’ to it.

### Entry point
Frontend is the entry point to our cluster, so we need the SVC to be external instead of clusterIP type. For that we use NodePort type instead and add a new port for the external traffic. So the frontend SVC port for internal traffic is 80 and for external traffic is 30007.

With the NodePort configuration each of the worker nodes of our K8s cluster now have an entry point on port 30007. 

### Testing the configuration

Now we create a K8s cluster with 3 worker nodes in Linode platform. Linode provides us with a kubeconfig yaml file that we download. We edit its permissions to 400 (only owner and only read) and then we set the env var KUBECONFIG to the path of the downloaded kubeconfig yaml file. This is done to be able to connect to the Linode cluster from my local machine.

Using kubectl I connect to the cluster and I run commands to 
- Check that I have 3 worker nodes	 
- Create a NS
- Deploy the 11 microservices in the NS -> using the configurations of each MS in a file called ‘config.yaml’
- Check the pods were running
- Check the SVCs

As mentioned before now each worker node has an entry point so I can take the public IP address of any of them and call it via port 3007 and check in the browser that the Microservice system is working and we can see the frontend app appear in the browser, then is a matter of playing with it to check that the other Microservices work.

### Chapter 22
In this chapter we refactor the ‘config.yaml’ file to follow best practices.

#### Best practices
1. Configure the specific version of the image to make sure on restart the deployment is consistent, if we don’t specify there might be a new version and it will pull a version the microservice system is not expecting and create conflicts.

2. Configure livenessProbe on each container so Kubernetes can monitor the app is alive and if this is not the case then K8s will restart the container. Without this K8s knows about the pod but doesn’t know if the app crashed.   

	livenessProbe check the app’s health status. For that there is a simple script that pings the app at intervals that we define
	and check if the app responds.
 
	The google Microservices that we are using already have a health protocol implementation in each app, 10 of the
	icroservices have a gRPC health implementation that can be found in the app’s code for example for ‘emailservice’:

	Class BaseEmailService -> 	Check() function 

	This Check function is what Kubernetes (the node’s Kubelet) will call to check the health of the app. 	

	for livenessProbe configuration we provide:
	- A probe mechanisms that Kubernetes supports:
        - grpc
        - httpGet
        - tcpSocket
        - exec
    - Port 
    - Interval to trigger the health check
    - Initial delay before running the check the first time to allow for the app to be up.

	As mentioned before all MS but one (frontend) have a gRPC probe mechanism so we pass that configuration to 10 MS along
	with the port the app listens to, the check interval (periodSeconds) and the delay (initialDelaySeconds).

	Process:
	1. client Kubelet opens a connection to <pod_ip>: <port_open_by_the_app> at certain interval of time.
	2. Kubelet calls grpc.health.v1.Health/Check 
	3. App receives the call and executes the actual function Check() inside the app’s conatiner
	4. app returns the result to the Kubelet which is in the node.
	5. kubetelt decides what to do:
		- success -> do nothing
		- fail -> create an event and increase counter -> If there are enough consecutive failures of liveness then Kubelet kills 
		   the container and restarts it.

	frontend app uses http so the probe mechanism is httpGet and we need to provide the path of the health endpoint which in
	this case is ‘/_healthz’.


3. Configure readinessProbe for each container so Kubernetes can allow or stop the traffic flow. I f we don’t check for readiness then k8s assumes the app is ready for traffic as soon as the container starts and will allow traffic when the app is still in the ‘starting’ phase and the client/user will get a ‘failed request’.

	the configuration is the same as for livenessProbe:
	- A probe mechanisms that Kubernetes supports:
        - grpc
        - httpGet
        - tcpSocket
        - exec
    - Port 
    - Interval to trigger the health check
    - Initial delay before running the check the first time to allow for the app to be up.

	both readinessProbe and livenessProbe call the same Check function the difference is what they do with the result:
	
	livenessProbe -> is app alive? 
		- yes  -> do nothing
		- no   -> restart container

	readinessProbe -> is app ready?
		- yes -> send traffic
		- no.  -> stop traffic (remove pod from service endpoints)

4. Configure resource request for each container to guarantee that the container gets the computer resources that it needs.
	- CPU -> measured in millicores (m)
	- Memory -> measure in Mebibytes (Mi) -> K8s and OS use this unit.

	kubernets scheduler uses this information to figure out the node with enough available resource to run a pod.

	if the resources values are > than the Nodes available resources then the Pod will never be scheduled.

	1 CPU core = 1000 millicores
	
  Best practice is to keep CPU at ‘1” or below, this is the reason we use millicores as it allows us to define amounts smaller
	than 1 CPU core, for example 100 m which is 0.1 CPU or 10%.

  Most apps don’t need a full CPU core.

5. Configure resource limits for each container to protect the worker node from giving all its resources to a malfunctioning app.
	the max resources an app should have is also provided by the devs team but in our case from the google GitHub repo.

6. Configure only one entry point. As we already mentioned frontend configuration has a SVC of type NodePort which opened port 30007 in each worker node, so we have multiple entry point to our cluster. This is bad practice as the surface of attack is bigger.

	Best practice -> 1 entry point outside the cluster.

	so we configure the frontend SVC to be of kind LoadBalancer, this service will signal a third party platform like a cloud
	provider in our case Linode and Linode then creates an external LoadBalancer.

	this external Loadbalancer is now the entry point and now we only have one single entry point.

	Pod -> created by K8s
	SVC -> created by K8s
	external LoadBalancer -> created by Linode

	an alternative to LoadBalancer:
	- Ingress
	- Gateway API

7. Configure more than 1 replica per deployment to ensure availability to avoid downtime. We didn’t have the ‘replicas’ configuration so by default it was ‘1’, now we add it and assign it the value of 2.

8. Configure more than 1 worker node, again for availability.

	1 worker node -> all apps in one server -> one single point of failure.

	we need at least 2 to 3 worker nodes in our cluster, and make suer each replica is in a different worker node -> availability

9. Configure labels for all resources. Labelas are used as an ID of the component in order for other components to reference them and find them.

	in our DEMO we labeled the deployment component using ‘app’ and the value assign to it is the app’s name. Then in the SVC
	configuration we reference the deployment label in the ‘selector’ section.

9. Configure namespaces to isolate resources, so all the resources of an app belong to the same one namespace. By doing this we can define access rights for example if we have multiple teams and each team works on one app then they can access one namespace that groups the resources of that app in the cluster and manage it but not access other resources that belong to another namespace.

	In our demo we only created 1 namespace ‘microservices’ and deployed all the 11 MSs in that namespace.

#### Three security Best practices

1. Ensure the images we use in the cluster are free of vulnerabilities, we can check via:
	- manual vulnerability scans
	- or automated scans in the CI/CD pipelines

We can use tools like aquascan to do vulnerability checks.

2. No ‘root’ access for containers.
	If no access to container’s ‘root’ -> no access to container’s host (worker node)

	we need to configure containers to use unprivileged user with ‘securityContext’ section:
	- ‘privileged’ to value ‘false’
	- ‘runAsNonRoot’ to value ‘true’

	This can be done at pod level as well Namespace.
	
	if we don’t block the access in the configuration to all the containers then we have to check the images and see what is their
	default user.
	- most of bitnami images set non-user USER
	- most of the images don’t have user instruction so the default is ‘root’ 

3. Update Kubernetes Cluster to the latest version as newer versions have:
	- fixes to bugs
	- fixes to security
	
	we update k8s version one worker node at the time to avoid downtime.


At the end of the DEMO we apply the ‘config.yaml’ and deployed all the 11 Microservices.

I had to try several times and figured that I needed a cluster with 5 worker nodes to get all of the containers app and running, the original 3 worker nodes didn’t have enough resources for the apps to work.

I learnt how to configure applications to deploy in a Kubernetes Cluster, I learnt best practices, understand gRPC implementation	 and got a better picture of the whole life cycle of an app (from developing it to deploying it).
