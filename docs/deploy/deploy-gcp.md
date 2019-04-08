# Citrix ADC CPX as an Ingress in Google Cloud Platform

This section explains the deployment of Citrix ADC CPX as an ingress in [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/) and [Google Compute Engine (GCE)](https://cloud.google.com/compute/). The procedure to deploy the Citrix ADC CPX remains the same for both Google Kubernetes Engine (GKE) and Google Compute Engine (GCE). If you configure Kubernetes on Google Compute Engine (GCE), then you need to deploy the CNI plug-in for the kubernetes cluster.

## Prerequisites

Ensure you have a Kubernetes Cluster up and running.

If you are running your cluster in GKE, then ensure you have configured a cluster-admin cluster role binding. You can do that using the following command:

    kubectl create clusterrolebinding citrix-cluster-admin --clusterrole=cluster-admin --user=<email-id of your google account>

Get your Google account details using the following command:

    gcloud info | grep Account

**To deploy Citrix CPX as an Ingress:**

1.  Create a sample application and expose it as service. In our example, let's use an apache web-server.

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/gcp/manifest/apache.yaml

1.  Create a Citrix CPX

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/gcp/manifest/standalone_cpx.yaml

1.  Create an ingress object

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/gcp/manifest/cpx_ingress.yaml

1.  Expose the Citrix CPX as a service of type Load-balancer. It creates an Azure LB with an External IP for receiving traffic. This is supported in kubernetes since v1.10.0.

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/gcp/manifest/cpx_service.yaml

1.  After executing the command, wait for the load-balancer to create an external IP. View the deployed services using the following command:

        kubectl get svc

    |NAME | TYPE | CLUSTER-IP | EXTERNAL-IP | PORT(S) | AGE |
    | --- | ---| ----| ----| ----| ----|
    |apache | ClusterIP |10.7.248.216 |none |  80/TCP | 2m |
    |cpx-ingress |LoadBalancer | 10.7.241.6 |  pending | 80:32258/TCP,443:32084/TCP | 2m|
    |kubernetes |ClusterIP |10.7.240.1 |none | 443/TCP | 22h|

1.  Once the IP is available, you can access your resources through the External IP provided by the load-balancer.

        kubectl get svc

    |Name | Type | Cluster-IP | External IP| Port(s) | Age |
    |-----| -----| -------| -----| -----| ----|
    |apache| ClusterIP|10.7.248.216|none|80/TCP |3m|
    |cpx-ingress|LoadBalancer|10.7.241.6|EXTERNAL-IP CREATED|80:32258/TCP,443:32084/TCP|3m|
    |kubernetes| ClusterIP| 10.7.240.1|none|443/TCP|22h|`

    The health check for the cloud load-balancer is obtained from the readinessProbe configured in the [Citrix CPX deployment yaml](https://github.com/citrix/citrix-k8s-ingress-controller/blob/master/deployment/gcp/manifest/cpx_service.yaml) file. So if the health check fails for some reason, you need to check the readinessProbe configured for Citrix CPX.

    You can read further about [readinessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-readiness-probes) and [external Load balancer](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)

    After the External-IP is created in the service, you can do a curl to the external IP using the host header **citrix-ingress.com**

        curl http://<External-ip-of-loadbalancer>/ -H 'Host: citrix-ingress.com'

    !!! note "Note"
        For ease of deployment, the following deployment models are explained with an all-in-one manifest file that combines all the explained steps. You can modify the manifest to suit your application and configuration.

## Deployment models

The following are the deployment solutions for deploying CPX as an ingress device in Google Cloud

-  Standalone Citrix CPX deployment

-  High availability Citrix CPX deployment

-  Citrix CPX per node deployment

### Standalone CPX deployment

To deploy Citrix CPX as an Ingress in a standalone deployment model in GCP, you must use the Service Type as ***LoadBalancer*** that would create a Load-balancer in Google cloud. This is supported in kubernetes since v1.10.0.

![CPX-GCP-Topology-Standalone](../media/CPX-GCP-Topology-Standalone.png)

1.  Execute the following command to create a Citrix CPX ingress with Citrix Ingress Controller as a sidecar in your Kubernetes cluster:

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/gcp/manifest/all-in-one.yaml

1.  To access the application:

        curl http://<External-ip-of-loadbalancer>/ -H 'Host: citrix-ingress.com'

To delete the deployment, use the following command:

    kubectl delete -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/gcp/manifest/all-in-one.yaml

### High availability Citrix CPX deployment

In the standalone deployment of Citrix ADC CPX as ingress, if the ingress device fails for some reason there would be a traffic outage for a few seconds. To avoid this disruption, instead of deploying a single CPX ingress, you can deploy two CPX ingress devices. So that if one CPX fails, the other CPX is available to handle traffic until the failed CPX comes up.

![CPX-GCP-HA-Solution-Topology](../media/CPX-GCP-HA-Solution-Topology.png)

1.  Execute the following command to create a CPX ingress with Citrix Ingress Controller as a sidecar in your kubernetes cluster:

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/gcp/manifest/all-in-one-ha.yaml

1.  To access the application:

        curl http://<External-ip-of-loadbalancer>/ -H 'Host: citrix-ingress.com'

To delete the deployment, use the following command:

    kubectl delete -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/gcp/manifest/all-in-one-ha.yaml

### Citrix CPX per node deployment

Sometimes where cluster nodes are added and removed from the cluster, CPX can also be deployed as daemonsets so that every node has a CPX ingress in them. This is much more reliable solution than deploying two CPX as ingress devices when the traffic is high.

![CPX-GCP-Daemonset-Topology](../media/CPX-GCP-Daemonset-Topology.png)

1.  Execute the following command to create a CPX ingress with in built Citrix Ingress Controller in your kubernetes cluster:

        kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/gcp/manifest/all-in-one-reliable.yaml

1.  To access the application:

        curl http://<External-ip-of-loadbalancer>/ -H 'Host: citrix-ingress.com'

To delete the deployment, use the following command:

    kubectl delete -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/deployment/gcp/manifest/all-in-one-reliable.yaml