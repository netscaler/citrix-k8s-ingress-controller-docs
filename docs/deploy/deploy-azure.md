# Citrix ADC CPX as an Ingress in Azure Kubernetes Engine

This section explains the deployment of Citrix ADC CPX as an ingress in [Azure Kubernetes Engine (AKS)](https://azure.microsoft.com/en-in/services/kubernetes-service/) in basic networking mode (kubenet). You can also configure kubernetes cluster in [Azure VM](https://azure.microsoft.com/en-in/services/virtual-machines/) and then deploy Citrix ADC CPX.

The procedure to deploy Citrix ADC CPX remains the same for both AKS and Azure VM. If you configure Kubernetes on Azure VM, then you need to deploy the CNI plug-in for the kubernetes cluster.

## Prerequisites

Ensure you have a Kubernetes Cluster up and running.

This section only explains deployment in Azure Kubernetes Engine in Basic Networking Mode.

Ensure that the AKS is configured only in Basic Networking mode (kubenet) and not in Advanced Networking mode (Azure CNI)

For more information, see [Guide to create an AKS cluster](https://github.com/citrix/citrix-k8s-ingress-controller/blob/master/deployment/azure/create-aks/README.md) for any help in creating a Kubernetes cluster in AKS.

**To deploy Citrix CPX as an Ingress:**

1.  Create a sample application and expose it as service. In our example, let's use an apache web-server.

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/azure/manifest/apache.yaml

1.  Create a Citrix CPX

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/azure/manifest/standalone_cpx.yaml

1.  Create an ingress object using the following command:

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/azure/manifest/cpx_ingress.yaml

1.  Expose the Citrix CPX as a service of type Load-balancer. Deploying the CPX service YAML file creates an Azure LB with an External IP for receiving traffic. This is supported in kubernetes from v1.10.0.

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/azure/manifest/cpx_service.yaml

1.  After executing the command, wait for the load-balancer to create an external IP. View the details of the created services using the following command:

        kubectl gets svc

    |NAME|TYPE|CLUSTER-IP|EXTERNAL-IP|PORT(S)| AGE|
    |----|----|-----|-----|----|----|
    |apache |ClusterIP|10.0.103.3|none|   80/TCP | 2m|
    |cpx-ingress |LoadBalancer |10.0.37.255 | pending |80:32258/TCP,443:32084/TCP |2m|
    |kubernetes |ClusterIP | 10.0.0.1 |none |  443/TCP | 22h |

1.  Once the IP address is available, you can access your resources through the external IP address provided by the load-balancer.

        kubectl get svc

    |NAME|TYPE|CLUSTER-IP|EXTERNAL-IP|PORT(S)|  AGE|
    |---|---|----|----|----|----|
    |apache|ClusterIP|10.0.103.3 |none|80/TCP|  3m|
    |cpx-ingress |LoadBalancer|10.0.37.255|  EXTERNAL-IP CREATED| 80:32258/TCP,443:32084/TCP |  3m|
    |kubernetes|    ClusterIP|10.0.0.1 |none| 443/TCP| 22h|

The health check for the cloud load-balancer is obtained from the readinessProbe configured in the [Citrix CPX deployment yaml](https://github.com/citrix/citrix-k8s-ingress-controller/blob/master/deployment/azure/manifest/cpx_service.yaml) file. So if the health check fails for some reason, you need to check the readinessProbe configured for Citrix CPX.

You can read further about [readinessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-readiness-probes) and [external Load balancer](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)

After the External-IP is created in the service, you can do a curl to the external IP address using the host header **citrix-ingress.com**

    curl http://<External-ip-of-loadbalancer>/ -H 'Host: citrix-ingress.com'

!!! note "Note"
    For ease of deployment, the following deployment models are explained with an all-in-one manifest file that combines all the explained steps. You can modify the manifest to suit your application and configuration.

## Deployment models

The following are the deployment solutions for deploying CPX as an ingress device in AKS:

-  Standalone Citrix CPX deployment

-  High availability Citrix CPX deployment

-  Citrix CPX per node deployment

### Standalone CPX deployment

To deploy Citrix CPX as an Ingress in a standalone deployment model in AKS, you need to use Service Type as LoadBalancer which would create a Load-balancer in Azure cloud. This is supported in kubernetes since v1.10.0.

![Azure_Standalone_CPX](../media/Azure_Standalone_CPX.png)

1.  Execute the following command to create a Citrix CPX ingress with Citrix Ingress Controller as a sidecar in your Kubernetes cluster:

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/azure/manifest/all-in-one.yaml

1.  To access the application:

        curl http://<External-ip-of-loadbalancer>/ -H 'Host: citrix-ingress.com'

    To delete the deployment, use the following command:

        kubectl delete -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/azure/manifest/all-in-one.yaml

### High availability Citrix CPX deployment

In the standalone deployment of Citrix CPX as ingress, if the ingress device fails for some reason there would be a traffic outage for a few seconds. To avoid the disruption, instead of deploying a single Citrix CPX ingress, you can deploy two Citrix CPX ingress devices. So that if one Citix CPX fails, the other Citrix CPX is availble to handle the traffic until the failed Citrix CPX comes up.

![Azure_HA_CPX](../media/Azure_HA_CPX.png)

1.  Execute the following command to create a CPX ingress with Citrix Ingress Controller as a sidecar in your kubernetes cluster:

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/azure/manifest/all-in-one-ha.yaml

1.  To access the application:

        curl http://<External-ip-of-loadbalancer>/ -H 'Host: citrix-ingress.com'

    To delete the deployment, use the following command:

        kubectl delete -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/azure/manifest/all-in-one-ha.yaml

### Citrix CPX per node deployment

In certain cases the cluster nodes are added and removed from the cluster. Hence, Citrix CPX can also be deployed as daemonsets so that every node has a Citrix CPX ingress in them. This is much more reliable solution than deploying two Citrix CPX as ingress devices when the traffic is high.

![Azure_CPX_per_node](../media/Azure_CPX_per_node.png)

1.  Execute the following command to create a CPX ingress with in built Citrix Ingress Controller in your kubernetes cluster:

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/azure/manifest/all-in-one-reliable.yaml

1.  To access the application:

        curl http://<External-ip-of-loadbalancer>/ -H 'Host: citrix-ingress.com'

    To delete the deployment, use the following command:

        kubectl delete -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/azure/manifest/all-in-one-reliable.yaml