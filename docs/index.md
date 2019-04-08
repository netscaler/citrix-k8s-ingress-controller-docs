# What is the Citrix Ingress Controller?

Citrix provides an Ingress Controller for its Citrix ADC MPX (hardware), Citrix ADC VPX (virtualized), and Citrix ADC CPX (containerized) for [bare metal](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/deployment/baremetal) and [cloud](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/deployment) deployments. It is built around Kubernetes [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) and automatically configures one or more Citrix ADC based on the Ingress resource configuration.

Clients outside a Kubernetes cluster need a way to access the services provided by pods inside the cluster. For services that provide HTTP(s) access, this access is provided through a [layer-7 proxy](https://en.wikipedia.org/wiki/Proxy_server#Reverse_proxies) also known as Application Delivery Controller (ADC) device or a load balancer device. Kubernetes provides an API object, called [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) that defines rules on how clients access services in a Kubernetes cluster. Service owners create these [Ingress resources](https://kubernetes.io/docs/concepts/services-networking/ingress/#the-ingress-resource) that define rules for directing HTTP(s) traffic. An [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers) watches the Kubernetes API server for updates to the Ingress resource and accordingly reconfigures the Ingress ADC.

The Citrix Ingress Controller supports topologies and traffic management beyond standard HTTP(s) Ingress. Citrix ADCs with Citrix Ingress Controllers support [Single-Tier](#single-tier-topology) and [Dual-Tier](#dual-tier-topology) traffic load balancing. Citrix Ingress Controller (CIC) automates the configuration of Citrix ADCs to proxy traffic into ("North-South") and between ("East-West") the microservices in a Kubernetes cluster. Service owners can also control ingress TCP/TLS and UDP traffic.

North-South traffic refers to the traffic from clients outside the cluster to microservices in the Kubernetes cluster. East-West traffic refers to the traffic between the microservices inside the Kubernetes cluster.

Typically, North-South traffic is load balanced by Ingress devices such as Citrix ADCs while East-West traffic is load balanced by [kube-proxy](https://kubernetes.io/docs/concepts/overview/components/#kube-proxy). Since kube-proxy only provides limited layer-4 load balancing, service owners can utilize the Citrix Ingress Controller to achieve sophisticated layer-7 controls for [East-West traffic using the Ingress CPX ADCs](#dual-tier-topology-with-hairpin-e-w-mode).

## Deploying Citrix Ingress Controller

You can deploy Citrix Ingress Controller in the following deployment modes:

1.  As a standalone pod. This mode is used when managing ADCs such as, Citrix ADC MPX or VPX that are outside the Kubernetes cluster.

1.  As a sidecar in a pod along with the Citrix ADC CPX in the same pod. The controller is only responsible for the Citrix ADC CPX that resides in the same pod.

You can deploy Citrix Ingress Controller using Kubernetes YAML or Helm charts. For more information, see [Deploying Citrix Ingress Controller](/Docs/getting-started.md).

## Deployment topologies

Citrix ADCs can be combined in powerful and flexible topologies that complement organizational boundaries. Dual-tier deployments employ high-capacity hardware or virtualized Citrix ADCs (Citrix ADC MPX and VPX) in the first tier to offload security functions and implement relatively static organizational policies while segmenting control between network operators and Kubernetes operators.

In Dual-tier deployments, the second tier is within the Kubernetes Cluster (using the Citrix ADC CPX) and is under control of the service owners. Hence balances the need for stability for network operators while allowing Kubernetes users to implement high-velocity changes. Single-tier topologies are highly suited to organizations that are confident in their ability to handle high rates of change.

### Single-Tier topology

In a Single-Tier topology, Citrix ADC MPX or VPX devices proxy the traffic (North-South) from the clients to microservices inside the cluster. The Citrix Ingress Controller (CIC) is deployed as a pod in the Kubernetes cluster. The controller automates the configuration of Citrix ADCs (MPX or VPX) based on the changes to the microservices or the Ingress resources.

![Single-tier](../Images/singletopology.png)

### Dual-Tier topology

In Dual-Tier topology, Citrix ADC MPX or VPX devices in Tier-1 proxy the traffic (North-South) from the client to Citrix ADC CPXs in Tier-2. The Tier-2 Citrix ADC CPX then routes the traffic to the microservices in the Kubernetes cluster. The Citrix Ingress Controller deployed as a standalone pod configures the Tier-1 devices. And, the sidecar controller in one or more Citrix ADC CPX pods configures it's associated Citrix ADC CPX in the same pod.

![Dual-tier](../Images/dualtier.png)

### Cloud topology

Kubernetes clusters in public clouds such as [Amazon Web Services (AWS)](https://aws.amazon.com), [Google Cloud](https://cloud.google.com), and [Microsoft Azure](https://azure.microsoft.com/en-in/) can use their native load balancing services such as, [AWS Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/), [Google Cloud Load Balancing](https://cloud.google.com/load-balancing/), and [Microsoft Azure NLB](https://azure.microsoft.com/en-in/services/load-balancer/) as the first (relatively static) tier of load balancing to a second tier of Citrix ADC CPX. Citrix ADC CPX operates inside the Kubernetes cluster with the sidecar Ingress controller. The Kubernetes clusters can be self-hosted or managed by the cloud provider (for example, [AWS EKS](https://aws.amazon.com/eks/), [Google GKE](https://cloud.google.com/kubernetes-engine/) and [Azure AKS](https://docs.microsoft.com/en-us/azure/aks/)) while using the Citrix ADC CPX as the Ingress. If the cloud-based Kubernetes cluster is self-hosted or self-managed, the Citrix ADC VPX can be used as the first tier in a Dual-tier topology.

**Cloud deployment with Citrix ADC (VPX) in tier-1:**
![Cloud deployment with VPX in tier-1](/Images/cloud-deploy-vpx-tier-1.png)

**Cloud deployment with Cloud LB in tier-1:**
![Cloud deployment with CLB in tier-1](/Images/cloud-deploy-clb-tier-1.png)

### Using the Ingress ADC for East-West traffic

When the Citrix ADC CPX is deployed inside the cluster as an Ingress, it can be used to proxy network traffic between microservices ("East-West") within the cluster. For this to work the target microservice needs to be deployed in [headless](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) mode to bypass [kube-proxy](https://kubernetes.io/docs/concepts/overview/components/#kube-proxy) so that you can benefit from the advanced ADC functionalities provided by Citrix ADC.  

![Dual-tier-Hairpin-mode](../Images/dual-tier-topology-with-hairpin-E-W.png)
