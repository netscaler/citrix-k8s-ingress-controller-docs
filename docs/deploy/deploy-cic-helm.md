# Deploying Citrix Ingress Controller using Helm charts

You can deploy Citrix Ingress Controller (CIC) in the following modes:

-  As a standalone pod in the Kubernetes cluster. Use this mode if you are controlling Citrix ADCs (Citrix ADC MPX or Citrix ADC VPX) outside the cluster. For example, with [dual-tier](../deployment-topologies.md#dual-tier-topology) topologies, or [single-tier](../deployment-topologies.md#single-tier-topology) topology where the single tier is a Citrix ADC MPX or VPX. 

-  As a sidecar (in the same pod) with Citrix ADC CPX in the Kubernetes cluster. The sidecar controller is only responsible for the associated Citrix ADC CPX within the same pod. This mode is used in [dual-tier](../deployment-topologies.md#dual-tier-topology) or [cloud](../deployment-topologies.md#cloud-topology)) topologies.

The CIC [repo](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/charts) contains the [Helm](https://helm.sh/) charts of CIC that you can use to configure Citrix ADCs such as, VPX, MPX, or CPX in Kubernetes environment. The repo contains a directory called [stable](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/charts/stable) that includes stable version of the charts that are created and tested by Citrix.

## Deploying CIC as a pod in the Kubernetes cluster

Use the [citrix-k8s-ingress-controller](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/charts/stable/citrix-k8s-ingress-controller) chart to run Citrix Ingress Controller (CIC) as a pod in your Kubernetes cluster. The chart deploys CIC as a pod in your Kubernetes cluster and configures the Citrix ADC VPX or MPX ingress device.

### Prerequisites

-  Determine the NS_IP IP address needed by the controller to communicate with the appliance. The IP address might be anyone of the following depending on the type of Citrix ADC deployment:

    -  (Standalone appliances) NSIP - The management IP address of a standalone Citrix ADC appliance. For more information, see [IP Addressing in Citrix ADC](https://docs.citrix.com/en-us/citrix-adc/12-1/networking/ip-addressing.html).

    -  (Appliances in High Availability mode) SNIP - The subnet IP address. For more information, see [IP Addressing in Citrix ADC](https://docs.citrix.com/en-us/citrix-adc/12-1/networking/ip-addressing.html).

    -  (Appliances in Clustered mode) CLIP - The cluster management IP (CLIP) address for a clustered Citrix ADC deployment. For more information, see [IP addressing for a cluster](https://docs.citrix.com/en-us/citrix-adc/12-1/clustering/cluster-overview/ip-addressing.html).

-  The username and password of the Citrix ADC VPX or MPX appliance used as the Ingress device. The Citrix ADC appliance needs to have system user account (non-default) with certain privileges so that CIC can configure the Citrix ADC VPX or MPX appliance. For instructions to create the system user account on Citrix ADC, see [Create System User Account for CIC in Citrix ADC](#create-system-user-account-for-cic-in-citrix-adc).

    You can directly pass the username and password or use Kubernetes secrets. If you want to use Kubernetes secrets, create a secrete for the username and password using the following command:

        kubectl create secret  generic nslogin --from-literal=username='cic' --from-literal=password='mypassword'

#### Create System User Account for CIC in Citrix ADC

Citrix Ingress Controller (CIC) configures the Citrix ADC using a system user account of the Citrix ADC. The system user account should have certain privileges so that the CIC has permission to configure the following on the Citrix ADC:

-  Add, Delete, or View Content Switching (CS) virtual server
-  Configure CS policies and actions
-  Configure Load Balancing (LB) virtual server
-  Configure Service groups
-  Cofigure SSl certkeys
-  Configure routes
-  Configure user monitors
-  Add system file (for uploading SSL certkeys from Kubernetes)
-  Configure Virtual IP address (VIP)
-  Check the status of the Citrix ADC appliance

To create the system user account, do the following:

1.  Log on to the Citrix ADC appliance. Perform the following:
    1.  Use an SSH client, such as PuTTy, to open an SSH connection to the Citrix ADC appliance.

    1.  Log on to the appliance by using the administrator credentials.

1.  Create the system user account using the following command:

        add system user <username> <password>

    For example:

        add system user cic mypassword

1.  Create a policy to provide required permissions to the system user account. Use the following command:

        add cmdpolicy cic-policy ALLOW "(^\S+\s+cs\s+\S+)|(^\S+\s+lb\s+\S+)|(^\S+\s+service\s+\S+)|(^\S+\s+servicegroup\s+\S+)|(^stat\s+system)|(^show\s+ha)|(^\S+\s+ssl\s+certKey)|(^\S+\s+ssl)|(^\S+\s+route)|(^\S+\s+monitor)|(^show\s+ns\s+ip)|(^\S+\s+system\s+file)"

    !!! note "Note"
        The system user account would have privileges based on the command policy that you define.

1.  Bind the policy to the system user account using the following command:

        bind system user cic cic-policy 0

**To deploy the [citrix-k8s-ingress-controller](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/charts/stable/citrix-k8s-ingress-controller) chart, perform the following:**

Use the `helm install` command to install the [citrix-k8s-ingress-controller](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/charts/stable/citrix-k8s-ingress-controller) chart.

For example:

    helm install citrix-k8s-ingress-controller -set nsIP= <NSIP>,license.accept=yes,ingressClass=<ingressClassName>

!!! note "Note"
    By default the chart installs the recommended [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) roles and role bindings.

To configure the CIC when installing the chart, you need to pass CIC specific parameters in `helm install`. The following table lists the mandatory and optional parameters that you can use:

!!! note "Note"
    You can also use the [values.yaml](https://github.com/citrix/citrix-k8s-ingress-controller/blob/master/charts/stable/citrix-k8s-ingress-controller/values.yaml) to pass the parameters.

| Parameters | Mandatory or Optional | Default value | Description |
| --------- | --------------------- | ------------- | ----------- |
| nsIP | Mandatory | N/A | The IP address of the Citrix ADC device. For details, see [Prerequisites](#prerequistes).
| license.accept | Mandatory | no | Set `yes` to accept the CIC end user license agreement. |
| image.repository | Mandatory | `quay.io/citrix/citrix-k8s-ingress-controller` | The repository of the CIC image |
| image.tag | Mandatory | 1.1.1 | The CIC image tag. |
| image.pullPolicy | Mandatory | Always | The CIC image pull policy. |
| nsPort | Optional | 443 | The port used by CIC to communicate with Citrix ADC. You can port 80 for HTTP. |
| nsProtocol | Optional | HTTPS | The protocol used by CIC to communicate with Citrix ADC. You can also use HTTP on port 80. |
| logLevel | Optional | DEPUG | The loglevel to control the logs generated by CIC. The supported loglevels are: CRITICAL, ERROR, WARNING, INFO, and DEBUG. For more information, see [Log Levels](../configure/log-levels.md).|
| kubernetesURL | Optional | N/A | The kube-apiserver url that CIC uses to register the events. If the value is not specified, CIC uses the [internal kube-apiserver IP address](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#accessing-the-api-from-a-pod). |
| ingressClass | Optional | N/A | If multiple ingress load balancers are used to load balance different ingress resources. You can use this parameter to specify CIC to configure Citrix ADC associated with specific ingress class. |
| name | Optional | N/A | Use the argument to specify the release name using which you want to install the chart. |
| | | |For example: `helm install citrix-k8s-ingress-controller --name my-release --set nsIP= <NSIP>,license.accept=yes,ingressClass=<ingressClassName>`|
| exporter.require=1.0 | Optional | N/A | Use the argument If you want to run the [Exporter for Citrix ADC Stats](https://github.com/citrix/netscaler-metrics-exporter) along with CIC to pull metrics for the Citrix ADC VPX or MPX|
| | | | For example: `helm install citrix-k8s-ingress-controller --name my-release --set license.accept=yes,ingressClass=<ingressClassName>,exporter.require=1.0` |

## Deploying CIC as a sidecar with Citrix ADC CPX

Use the [citrix-k8s-cpx-ingress-controller](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/charts/stable/citrix-k8s-cpx-ingress-controller) chart to deploy a Citrix ADC CPX with CIC as a sidecar. The chart deploys a Citrix ADC CPX instance that is used for load balancing the North-South traffic to the microservices in your Kubernetes cluster and the sidecar CIC configures the Citrix ADC CPX.

Use the `helm install` command to install the [citrix-k8s-cpx-ingress-controller](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/charts/stable/citrix-k8s-cpx-ingress-controller) chart.

!!! note
    By default the chart installs the recommended [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) roles and role bindings.

To configure the CIC when installing the chart, you need to pass CIC specific parameters in `helm install`. The following table lists the mandatory and optional parameters that you can use:

!!! tip
    You can also use the [values.yaml](https://github.com/citrix/citrix-k8s-ingress-controller/blob/master/charts/stable/citrix-k8s-cpx-ingress-controller/values.yaml).

| Parameters | Mandatory or Optional | Default value | Description |
| ---------- | --------------------- | ------------- | ----------- |
| license.accept | Mandatory | no | Set `yes` to accept the CIC end user license agreement. |
| cpximage.repository | Mandatory | `quay.io/citrix/citrix-k8s-cpx-ingress` | The Citrix ADC CPX image repository. |
| cpximage.tag | Mandatory | 12.1-51.16 | The Citrix ADC CPX image tag. |
| cpximage.pullPolicy | Mandatory | Always | The Citrix ADC CPX image pull policy. |
| cicimage.repository | Mandatory | `quay.io/citrix/citrix-k8s-ingress-controller` | The CIC image repository. |
| cicimage.tag | Mandatory | 1.1.1 | The CIC image tag. |
| cicimage.pullPolicy | Mandatory | Always | The CIC image pull policy. |
| exporter.require=1.0 | Optional | N/A | Use the argument if you want to run the [Exporter for Citrix ADC Stats](https://github.com/citrix/netscaler-metrics-exporter) along with CIC to pull metrics for the Citrix ADC VPX or MPX|
| exporter.image.repository | Optional | `quay.io/citrix/netscaler-metrics-exporter` | The Exporter for Citrix ADC Stats image repository. |
| exporter.image.tag | Optional | v1.0.0 | The Exporter for Citrix ADC Stats image tag. |
| exporter.image.pullPolicy | Optional | Always | The Exporter for Citrix ADC Stats image pull policy. |
| exporter.ports.containerPort | Optional | 8888 | The Exporter for Citrix ADC Stats container port. |
| ingressClass | Optional | N/A | If multiple ingress load balancers are used to load balance different ingress resources. You can use this parameter to specify CIC to configure Citrix ADC associated with specific ingress class. |
