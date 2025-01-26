# DNS Request ----> pods ****************************************

The user's browser or client queries the domain name through DNS.
Amazon Route 53 resolves the domain to the IP address of your load balancer.
The AWS Load Balancer receives the request.
It forwards the request to one of the Istio IngressGateway pods based on load-balancing rules (e.g., round-robin, least connections).
The IngressGateway receives the request from the load balancer.
Based on Istio's configuration, it determines which VirtualService (VS) to route the request to.
The VirtualService specifies how the request should be routed based on criteria like host, path, or headers.
It directs the request to the appropriate Kubernetes Service
The service (usually a ClusterIP or NodePort service) receives the request from the IngressGateway.
It forwards the request to one of the pods that match the service's label selector.
The selected pod receives the request via the Kube-proxy or CNI (Container Network Interface).
The container processes the request and generates a response.
The response is sent back through the same chain in reverse order:
Pod → Service → IngressGateway → Load Balancer → User.


Detailed Flow Example
User: Sends a request to https://api.example.com.
Route 53: Resolves api.example.com to the IP of the AWS Load Balancer.
Load Balancer: Routes the request to an Istio IngressGateway pod.
IngressGateway: Inspects the request and routes it to the appropriate VirtualService.
VirtualService: Matches the request (e.g., based on Host or Path) and forwards it to the correct Kubernetes Service.
Kubernetes Service: Selects one of the pods backing the service.
Pod: Processes the request and sends the response back to the user.


# Pod-to-Pod Communication via Kubernetes Service ****************************************

It queries the Kubernetes Service using the service's DNS name (e.g., my-service.default.svc.cluster.local).
The Kubernetes Service acts as a virtual IP (VIP) that forwards requests to one of the pods behind the service.
The request is intercepted by the Envoy proxy of the source pod.
The proxy consults Istio's Service Registry to resolve the destination service and routing rules.
The request is forwarded to the Envoy proxy of the destination pod.

# Pod Lifecycle   ********************************************

The user (or an automation tool like CI/CD) sends a request to create a Pod using:
kubectl command: kubectl apply -f pod.yaml
REST API: Direct API calls to the Kubernetes API Server.
UI tools like Kubernetes Dashboard.
The Pod specification is provided in a YAML or JSON file, which defines:
Pod metadata (e.g., name, labels).
Containers and their configurations (image, ports, environment variables).
Resource requirements (CPU, memory).
Volume mounts (if any).


API Server (Front-end)
The request reaches the Kubernetes API Server, which acts as the front-end of the Control Plane.
The API Server:
Validates the request against the Kubernetes schema.
Ensures all required fields are provided and values are valid.
Stores the request (desired Pod state) in etcd, the cluster's key-value store.

Once the desired Pod state is stored in etcd, the Scheduler picks up the Pod that needs to be assigned to a node.
The Scheduler decides which Worker Node the Pod should run on based on:
Resource availability (CPU, memory, etc.).
Node taints and tolerations.
Pod affinity/anti-affinity rules.
Custom scheduling policies.
Once the Scheduler determines the best node, it updates the Pod's status in etcd with the assigned node.

The Pod is now "bound" to a specific Worker Node.
The Pod's state in the API Server is updated to reflect its node assignment.
Kubelet on the Worker Node
The Kubelet on the selected Worker Node detects that a Pod is scheduled for it (by querying the API Server).
The Kubelet pulls the Pod specification from the API Server, including:
Container details (image, resources, commands, etc.).
Volume requirements.
Networking settings (e.g., ports, IP).

Container Runtime
The Kubelet instructs the Container Runtime (e.g., containerd, CRI-O, or Docker) to:
Pull the container images from the specified container registry (e.g., Docker Hub, ECR).
Create and start the containers inside the Pod.
Configure the networking for the Pod and containers (e.g., setting up a virtual network interface).

Networking (Kube Proxy and CNI Plugin)
A Pod IP is assigned to the new Pod by the CNI (Container Network Interface) plugin (e.g., Flannel, Calico, AWS VPC CNI).
The Kube Proxy ensures the new Pod can communicate with other Pods, Services, and external clients.

Status Updates
The Kubelet continuously updates the API Server with the Pod's status (e.g., Pending → Running).
If something goes wrong (e.g., the container image fails to pull), the Kubelet reports this to the Control Plane, and the Pod enters a failed state.

Post-Creation Monitoring
The Controller Manager ensures the Pod remains in the desired state.
If the Pod crashes, a new instance is created automatically (depending on the controller managing the Pod, e.g., Deployment or ReplicaSet).
Tools like CloudWatch, Prometheus, or kubectl logs can be used to monitor the Pod's performance and health.




