# Edge devices power consumption data collection and visualization using kepler.


## Kepler setup in edge device (using RHEL 9.2 running in KVM as edge device)


Download the Red Hat Enterprise Linux 9.2 and
install any virtualization tools (can be, KVM, Virtualbox, VMware or any tool) to setup a RHEL virtual machine.

### Installing and starting the microshift from RPM package


- Enable the Microshift RPM repository

```shell
sudo subscription-manager repos \
--enable rhocp-4.14-for-rhel-9-$(uname -m)-rpms \
--enable fast-datapath-for-rhel-9-$(uname -m)-rpms
```

- Install Red Hat build of Microshift

``` shell
dnf install -y microshift
```

- Install greenboot for Red Hat build of Microshift (Optional)

```shell
sudo dnf install -y microshift-greenboot
```

- Download pull secret from [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/install/pull-secret)


```shell
sudo cp $HOME/openshift-pull-secret /etc/crio/openshift-pull-secret

```
- Make the root user the owner of the /etc/crio/openshift-pull-secret

```shell
sudo chown root:root /etc/crio/openshift-pull-secret
```

- Make the /etc/crio/openshift-pull-secret file readable and writeable by the root user

```shell
sudo chmod 600 /etc/crio/openshift-pull-secret
```

- To access the microshift locally, copy the kuebconfig file from the /var/lib/microshift/resources/kubeadmin/kubeconfig and store it in $HOME/.kube folder.

```shell
mkdir -p ~/.kube/
```

```shell
sudo cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
```

```shell
chmod go-r ~/.kube/config
```

Follow below steps when firewall-cmd is enabled in RHEL 9.2

- 10.42.0.0/16 is the network range for pods running in microshift and 169.254.169.1 is the IP address of Microshift OVN network

```shell
sudo firewall-cmd --permanent --zone=trusted --add-source={10.42.0.0/16,169.254.169.1} && sudo firewall-cmd --reload
```

Enable the microshift using
```shell
sudo systemctl enable --now microshift.service
```
```shell
sudo systemctl start microshift.service
```

Check the status of the Microshift
```shell
sudo systemctl status microshift.service
```

For verification, that microshift is running, Use the following command

```shell
oc get all -A
```

Note: usually take 10 mins to get all the workloads up and running on the first time. If any workload doesn't get started. Restart the virtual machine.

***Optional - Access Microshift from remote machines***

- To access microshift remotely edit the microshift config file. It'll not be configured, has to be configured maunally.  



```shell
sudo mv /etc/microshift/config.yaml.default /etc/microshift/config.yaml
```

```shell
vi /etc/microshift/config.yaml
```

```yaml
dns:
  baseDomain: microshift.example.com
node:
  ...
  ...
apiServer:
  subjectAltNames:
  - edge.microshift.example.com
  ...
  ...
```


```shell
sudo systemctl restart microshift
```

- View the kubeconfig file for the newly added subjectAltNames in the folder below

```shell
cat /var/lib/microshift/resources/kubeadmin/edge-microshift.example.com/kubeconfig
```

use scp to copy this file or copy the content and do the following steps in the remote machine

```shell
mkdir -p ~/.kube/
```

```shell
MICROSHIFT_MACHINE=<name or IP address of Red Hat build of MicroShift machine>
```

```shell
ssh <user>@$MICROSHIFT_MACHINE "sudo cat /var/lib/microshift/resources/kubeadmin/$MICROSHIFT_MACHINE/kubeconfig" > ~/.kube/config
```

```shell
chmod go-r ~/.kube/config
```

Make sure you have oc client in the remote machine and verify the access to microshift using

```shell
oc get all -A
```


### Configuring Kepler DaemonSet and enabling power consumption metrics in edge microshift

Apply all the manifests in the [edge-kepler](./edge/edge-kepler) directory using [kustomization file](./edge/edge-kepler/kustomization.yaml)

```shell
oc apply -k ./edge/edge-kepler
```

Deployed the opentelemetry collector as a side-car container in kepler daemonset.

-  Config file for opentelemetry collector is added as ConfigMap resource 
and created in kepler namespace.
 
Update this file with the opentelemetry hostname and port running in openshift


```yaml  
    exporters:
       logging:
          loglevel: info

       otlp:
        endpoint: http://<external-opentelemetry>:<port>
        tls:
         insecure: true
```

```shell
vi edge/edge-otel-collector/1-kepler-microshift-otelconfig.yaml
```

```shell
oc create -n kepler -f edge/edge-otel-collector/1-kepler-microshift-otelconfig.yaml
```

- Added the opemtelemetry as sidecar container with kepler daemonset.

Update the image with any custom opentelemetry collector if necessary or use the default collector image

```shell
vi edge/edge-otel-collector/2-kepler-patch-sidecar-otel.yaml
```

```shell
oc patch daemonset kepler-exporter-ds -n kepler --patch-file edge/edge-otel-collector/2-kepler-patch-sidecar-otel.yaml
```

The opentelemetry sidecar attached to the kepler daemonset in microshift is configured to send the data to an external opentelemetry collector running in openshift.

** **

***Alternate -To deploy kepler daemonset from official source code***

***To deploy kepler exporter Daemonset from the source code (if required)***

- Create namespace in microshift to deploy kepler daemon-set.
```
oc create ns kepler
```

```
oc label ns kepler security.openshift.io/scc.podSecurityLabelSync=false
```

```
oc label ns kepler --overwrite pod-security.kubernetes.io/enforce=privileged
```

- Downloaded the Kepler source code from [kepler - git repo](https://github.com/sustainable-computing-io/kepler) and renamed as edge/edge-kepler. 

```
git clone https://github.com/sustainable-computing-io/kepler.git ./edge/kepler
```

Kepler itself provides the easy way of deploying kepler expoerter as a daemonset in microshift using the [kustomization.yaml](./edge/edge-kepler/manifests/config/exporter/kustomization.yaml)

Since, microshift is a lightweight version of openshift, it requires scc permissions to be added.

Edit the following lines in [kustomization.yaml](./edge/edge-kepler/manifests/config/exporter/kustomization.yaml)

```
vi edge-kepler/manifests/config/exporter/kustomization.yaml
```

     1. Uncomment line 3
       - openshift_scc.yaml

     2. Remove [] in line 8
        patchesStrategicMerge:

     3. uncomment line 18
       - ./patch/patch-openshift.yaml



- Deployed kepler on microshift under the kepler namespace.
```
oc apply --kustomize edge/edge-kepler/manifests/config/base -n kepler
```


** **

### Data visualization setup in External openshift cluster (Openshift cluster running in remote)


- Namespace used for this demo in Openshift Container Platform

```shell
oc new-project kepler-demo
```

- Before Setting up install the necessary operators in the Openshift Console

  1. Red Hat OpenShift distributed tracing data collection operator

      Install from [Operator Hub](https://docs.openshift.com/container-platform/4.13/distr_tracing/distr_tracing_otel/distr-tracing-otel-installing.html)
 
  2. grafana-operator

     Install from [Operator Hub](https://www.ibm.com/docs/ko/erqa?topic=monitoring-installing-grafana-operator) \
   (or) \
   run the following command
   ```shell
   oc apply -f 0-grafana-operator.yaml
   ```

  3. observability-operator

     Install from [Operator Hub](https://docs.openshift.com/container-platform/4.14/monitoring/cluster_observability_operator/installing-the-cluster-observability-operator.html)
   
#### OpenTelemetry Setup
- Configured the opentelemetry collector using Red Hat OpenShift distributed tracing data collection operator (From openshift Operator Hub).

```shell
oc apply -f ocp/ocp-otlp_collector/1-kepler-otel_collector.yaml
```



***Note:*** \
Opentelemetry collector can be exposed in both secure and insecure method.

**Secure Route**: Ingress configuration in OpenTelemetry Collector CRD creates a secure route for exposing the Otel collector outside of the cluster (Not tested).***

```yaml
  ingress:
    route:
      termination: edge
    type: route
```

**Insecure Route**: For exposing OpenTelemetry collector outside the OCP cluster (Insecure), MetalLB LoadBalancer is used for this demo, since openshift cluster is running in Bare Metal. This configuration may vary for Openshift running in cloud environment.

Apply the patch command to change the type in openshift service
```shell
oc patch service kepler-otel-service -n kepler-demo --type='json' -p '[{"op": "replace", "path": "/spec/type", "value": "LoadBalancer"}]'
```

#### Prometheus Stack Setup
- Deployed Monitoring Stack to collect prometheus data from the opentelemetry (running in external openshift) using Observability Operator (From openshift Operator Hub).

```shell
oc apply -f ocp/ocp-prometheus/1-kepler-prometheus-monitoringstack.yaml
```

#### Grafana Setup
- Deployed grafana Dashboard to visualize the data collected in the Prometheus Monitoring Stack.

Apply the grafana CRD

```shell
oc apply -f ocp/ocp-grafana/1-grafana-kepler.yaml
``` 


Apply the grafanaDatasource CRD

```shell
oc apply -f ocp/ocp-grafana/2-grafana-datasource-kepler.yaml
```

Note: 
- Grafana Dashboard model is provided as a JSON file in the [ocp/ocp-grafana/4-grafana-dashboard-kepler.json](./ocp/ocp-grafana/4-grafana-dashboard-kepler.json). (
This dashboard json is also available in official [kepler repo](https://raw.githubusercontent.com/sustainable-computing-io/kepler/main/grafana-dashboards/Kepler-Exporter.json)  )

- Configured in the [grafana-dashboard](./ocp/ocp-grafana/3-grafana-dashboard-kepler) CRD as


```yaml
spec:
  instanceSelector:
    matchLabels:
      dashboards: grafana-kepler
  url: https://raw.githubusercontent.com/ZagaUS/kepler-demo/main/ocp/ocp-grafana/4-grafana-dashboard-kepler.json
```

Apply the grafanaDashboard CRD

```shell
oc apply -f ocp/ocp-grafana/3-grafana-dashboard-kepler.yaml
```

**Note**: Route for Grafana CRD need to be added manually, either through web console (Administrator -> Networking -> Routes)  or use the [5-grafana-dashboard-route](./ocp/ocp-grafana/5-grafana-dashboard-route.yaml) yaml file.


Edit the yaml file, to use custom hostname if necessary or leave this line commented \
And verify the name of the service
```yaml
spec:
  # host: >-
  #   grafana-kepler-dashboard-kepler-demo.apps.zagaopenshift.zagaopensource.com
  to:
    kind: Service
    name: grafana-kepler-service
```
Apply the grafana route

```shell
oc apply -f ocp/ocp-grafana/5-grafana-dashboard-route.yaml
```

**  **

### References

https://github.com/sustainable-computing-io/kepler


https://github.com/redhat-et/edge-ocp-observability


https://github.com/openshift/microshift/blob/main/docs/contributor/network/ovn_kubernetes_traffic_flows.md


https://access.redhat.com/documentation/en-us/red_hat_build_of_microshift

https://docs.openshift.com/container-platform/4.13/distr_tracing/distr_tracing_otel/distr-tracing-otel-installing.html

https://www.ibm.com/docs/ko/erqa?topic=monitoring-installing-grafana-operator

https://docs.openshift.com/container-platform/4.14/monitoring/cluster_observability_operator/installing-the-cluster-observability-operator.html
