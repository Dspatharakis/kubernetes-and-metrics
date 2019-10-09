# Kubernetes

For kubernetes cluster creation [kubespray][1] is used. As is states, it allows deploying production ready kubernetes cluster. It uses ansible and thus it is highly configurable. See [documentation][2] for more information.

## Preparation

### Preparation of hosts

Before starting the deployment target hosts need to be specified. According to [kubespray requirements](https://kubespray.io/#/?id=requirements) the targets:

 * must have access to the Internet(although offline environments are also supported with additional configuration),
 * should allow IPv4 forwarding,
 * should have copies of ssh keys of the control machine (where ansible invocation takes place),
 * firewalls should be configured appropriately to not block connection of k8s services (i.e. etcd).

Finally, the targets should run one of the [supported linux distributions](https://kubespray.io/#/?id=supported-linux-distributions).

#### Applying preparation in POC

Taking these into account 3 target machines are prepared with Fedora 28 linux distribution installed. These machines are on the same network as the control machine (without this being a limitation). For ease of use static IPv4 addresses are assigned to these hosts and root account is used. Also firewall is disabled, although this can be fine-tuned so that certain k8s services can be white-listed.

For disabling firewall, Fedora 28 firewalld default behaviour should be adapted. Default zone "FedoraServer" by default every packet not being white-listed is rejected. to change this behaviour target="ACCEPT" is specified. firewalld service needs to be restarted on every host for this change to take effect.

To sum up the following steps are performed for POC:

 1. Deploy 3 hosts with Fedora 28 server, with root account enabled[^1].
 2. Assign static IPs[^2],
 3. Optionally change hostnames[^3] although ansible on next step will setup the names provided.
 4. Optionally create entries in `/etc/hosts`.
 5. Copy ssh ids from control host to target hosts[^4].
 6. Set firewalld default zone `FedoraServer` target to `ACCEPT` and restart firewalld service[^5].

### Preparation of kubespray

After preparation of hosts `kubespray` can be used to install kubernetes cluster. To use it download [tararouras/kubespray][1] (which is forked from [kubernetes-sigs/kubespray](https://github.com/kubernetes-sigs/kubespray)) and follow instructions in its [usage section](https://kubespray.io/#/?id=usage) until the point that `hosts.yml` file is created (with appropriate adjustments depending on the environment).

Before proceeding to execute the ansible command some variables need to be adjusted to prepare a k8s to our needs.

#### kubespray ansible variables

After copying `inventory/sample/` directory to `inventory/mycluster/` edit `inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml` file and set the following:

```
# Make a copy of kubeconfig on the host that runs Ansible in {{ inventory_dir }}/artifacts
kubeconfig_localhost: true
# Download kubectl onto the host that runs Ansible in {{ bin_dir }}
kubectl_localhost: true
```

Enabling these 2 options will allow controlling k8s cluster from the current machine after its creation.

#### kubespray ansible roles

**Note**: The changes on this section are ready under [extend-for-metrics-scaling branch](https://github.com/tararouras/kubespray/tree/extend-for-metrics-scaling). Use that branch to skip this step.

Since the created cluster is going to be used to control application scaling with metrics there are some options that need to be specified for later.

First, we set [Horizontal Pod Autoscaler][3] options to our needs. HPA options are passed as [command-line options](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/) to *kube-controller-manager* running on master nodes[^6].

Finally, we set *kubelet_custom_flags* in `roles/kubernetes/node/defaults/main.yml` as follows:
```
kubelet_custom_flags: ['--authentication-token-webhook=true','--authorization-mode=Webhook']
```

These options are later needed for [Prometheus][4] monitoring system to be able to authenticate to kubelet services and receive metrics.

After this configuration we are ready to deploy k8s cluster.

**Note**: Metrics server could also be enabled but later we are going to use [Prometheus][4] for metrics so there is no need to enabled k8s built-in metrics server.

#### Applying kubespray preparation in POC

**Note 1**: For this POC [kubespray][1] v2.10.4 is used.

To setup kubernetes cluster for POC 3 hosts are used:

* master1 at `192.168.0.130`,
* master2 at `192.168.0.131` and
* node1 at `192.168.0.132`.

Then:

```
# Inside kubespray directory

# Install dependencies from ``requirements.txt``
sudo pip3 install -r requirements.txt

# Copy ``inventory/sample`` as ``inventory/mycluster``
cp -rfp inventory/sample inventory/mycluster

# Update Ansible inventory file with inventory builder
declare -a IPS=(192.168.0.130 192.168.0.131 192.168.0.132)
CONFIG_FILE=inventory/mycluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

The previous commands produces the following `inventory/mycluster/hosts.yml` ansible hosts file:

```
all:
  hosts:
    node1:
      ansible_host: 192.168.0.130
      ip: 192.168.0.130
      access_ip: 192.168.0.130
    node2:
      ansible_host: 192.168.0.131
      ip: 192.168.0.131
      access_ip: 192.168.0.131
    node3:
      ansible_host: 192.168.0.132
      ip: 192.168.0.132
      access_ip: 192.168.0.132
  children:
    kube-master:
      hosts:
        node1:
        node2:
    kube-node:
      hosts:
        node1:
        node2:
        node3:
    etcd:
      hosts:
        node1:
        node2:
        node3:
    k8s-cluster:
      children:
        kube-master:
        kube-node:
    calico-rr:
      hosts: {}
```

Adaptations can be performed if needed. For POC this is the final file contents:

```
all:
  hosts:
    master1:
      ansible_host: 192.168.0.130
      ip: 192.168.0.130
      access_ip: 192.168.0.130
    master2:
      ansible_host: 192.168.0.131
      ip: 192.168.0.131
      access_ip: 192.168.0.131
    node1:
      ansible_host: 192.168.0.132
      ip: 192.168.0.132
      access_ip: 192.168.0.132
  children:
    kube-master:
      hosts:
        master1:
        master2:
    kube-node:
      hosts:
        master1:
        master2:
        node1:
    etcd:
      hosts:
        master1:
        master2:
        node1:
    k8s-cluster:
      children:
        kube-master:
        kube-node:
    calico-rr:
      hosts: {}
```

This basically suggests that k8s-cluster will be comprised of 2 master nodes (master1 and master2 hosts) and 3 nodes (master1, master2 and node1). This means that master nodes will have double role, k8s master and k8s node.

**Note 2**: The following change is ready under [extend-for-metrics-scaling branch](https://github.com/tararouras/kubespray/tree/extend-for-metrics-scaling). Use that branch to skip this step.

Edit `roles/kubernetes/master/templates/kubeadm-config.v1beta1.yaml.j2` and below `controllerManager/extraArgs` add jinja2 script that adds extra arguments specified in *controller_mgr_custom_flags* ansible variable. Then specify *controller_mgr_custom_flags* and *kubelet_custom_flags*. All adaptations should look as the following:

```
diff --git a/roles/kubernetes/master/defaults/main/main.yml b/roles/kubernetes/master/defaults/main/main.yml
index b2578e10..d63b28fb 100644
--- a/roles/kubernetes/master/defaults/main/main.yml
+++ b/roles/kubernetes/master/defaults/main/main.yml
@@ -137,7 +137,7 @@ apiserver_custom_flags: []
 # List of the preferred NodeAddressTypes to use for kubelet connections.
 kubelet_preferred_address_types: 'InternalDNS,InternalIP,Hostname,ExternalDNS,ExternalIP'

-controller_mgr_custom_flags: []
+controller_mgr_custom_flags: ['horizontal-pod-autoscaler-cpu-initialization-period: 15s', 'horizontal-pod-autoscaler-downscale-stabilization: 15s', 'horizontal-pod-autoscaler-initial-readiness-delay: 15s']

 scheduler_custom_flags: []

diff --git a/roles/kubernetes/master/templates/kubeadm-config.v1beta1.yaml.j2 b/roles/kubernetes/master/templates/kubeadm-config.v1beta1.yaml.j2
index 09b546c2..857ee0b6 100644
--- a/roles/kubernetes/master/templates/kubeadm-config.v1beta1.yaml.j2
+++ b/roles/kubernetes/master/templates/kubeadm-config.v1beta1.yaml.j2
@@ -182,6 +182,13 @@ apiServer:
   timeoutForControlPlane: 5m0s
 controllerManager:
   extraArgs:
+{% if controller_mgr_custom_flags is string %}
+    {{ controller_mgr_custom_flags }}
+{% else %}
+{% for flag in controller_mgr_custom_flags %}
+    {{ flag }}
+{% endfor %}
+{% endif %}
     node-monitor-grace-period: {{ kube_controller_node_monitor_grace_period }}
     node-monitor-period: {{ kube_controller_node_monitor_period }}
     pod-eviction-timeout: {{ kube_controller_pod_eviction_timeout }}
diff --git a/roles/kubernetes/node/defaults/main.yml b/roles/kubernetes/node/defaults/main.yml
index 6bf6c54b..f13c60d9 100644
--- a/roles/kubernetes/node/defaults/main.yml
+++ b/roles/kubernetes/node/defaults/main.yml
@@ -63,7 +63,7 @@ kubelet_load_modules: false
 kubelet_max_pods: 110

 ## Support custom flags to be passed to kubelet
-kubelet_custom_flags: []
+kubelet_custom_flags: ['--authentication-token-webhook=true', '--authorization-mode=Webhook']

 ## Support custom flags to be passed to kubelet only on nodes, not masters
 kubelet_node_custom_flags: []
```

### Deploy k8s cluster using kubespray

After making all the adaptations described above execute the following as root:

```
# Deploy Kubespray with Ansible Playbook - run the playbook as root
# The option `-b` is required, as for example writing SSL keys in /etc/,
# installing packages and interacting with various systemd daemons.
# Without -b the playbook will fail to run!
ansible-playbook -i inventory/mycluster/hosts.yml --become --become-user=root cluster.yml
```

If everything proceeds without issues `ansible-playbook` command will succeed. After this copy `inventory/mycluster/artifacts/admin.conf` to user's home directory `.kube/config` and `kubectl` binary to system's PATH. For example:

```
# set kube config received from master node to akouris default kube config
cp inventory/mycluster/artifacts/admin.conf /home/akouris/.kube/config
# set akouris as owner
chown akouris:akouris /home/akouris/.kube/config
# copy kubectl tool to /usr/local/bin/
cp inventory/mycluster/artifacts/kubectl /usr/local/bin/kubectl
```

After this as user you can access the newly created cluster using kubectl command. For example:

```
[akouris@hephaestus kubespray]$ kubectl cluster-info
Kubernetes master is running at https://192.168.0.130:6443
coredns is running at https://192.168.0.130:6443/api/v1/namespaces/kube-system/services/coredns:dns/proxy
kubernetes-dashboard is running at https://192.168.0.130:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[akouris@hephaestus kubespray]$ kubectl get nodes
NAME      STATUS   ROLES    AGE   VERSION
master1   Ready    master   14m   v1.14.3
master2   Ready    master   13m   v1.14.3
node1     Ready    <none>   12m   v1.14.3
[akouris@hephaestus kubespray]$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-cf96bc758-cz78h   1/1     Running   0          12m
kube-system   calico-node-bcnr7                         1/1     Running   0          13m
kube-system   calico-node-jflwp                         1/1     Running   0          13m
kube-system   calico-node-vbl9b                         1/1     Running   0          13m
kube-system   coredns-56bc6b976d-fxlnn                  1/1     Running   0          12m
kube-system   coredns-56bc6b976d-vw8jr                  1/1     Running   0          12m
kube-system   dns-autoscaler-5fc5fdbf6-5brmc            1/1     Running   0          12m
kube-system   kube-apiserver-master1                    1/1     Running   0          13m
kube-system   kube-apiserver-master2                    1/1     Running   0          13m
kube-system   kube-controller-manager-master1           1/1     Running   0          13m
kube-system   kube-controller-manager-master2           1/1     Running   0          13m
kube-system   kube-proxy-4vtsn                          1/1     Running   0          12m
kube-system   kube-proxy-khp6v                          1/1     Running   0          12m
kube-system   kube-proxy-rrwqx                          1/1     Running   0          13m
kube-system   kube-scheduler-master1                    1/1     Running   0          13m
kube-system   kube-scheduler-master2                    1/1     Running   0          13m
kube-system   kubernetes-dashboard-6c7466966c-45qwn     1/1     Running   0          12m
kube-system   nginx-proxy-node1                         1/1     Running   0          13m
kube-system   nodelocaldns-q98pl                        1/1     Running   0          12m
kube-system   nodelocaldns-s5v2w                        1/1     Running   0          12m
kube-system   nodelocaldns-xkkjt                        1/1     Running   0          12m
```

Kubernetes is now ready to be used.

## Add metrics support with Prometheus

To support gathering metrics from kubernetes and applications running on k8s cluster, [tararouras/kube-prometheus][5] (which is forked from [coreos/kube-prometheus](https://github.com/coreos/kube-prometheus)) is going to be used.

### Quick Start

1. Clone [tararouras/kube-prometheus](https://github.com/tararouras/kube-prometheus) and checkout to [add_custom_metrics](https://github.com/tararouras/kube-prometheus/tree/add_custom_metrics) branch.

2. Follow instructions in [Quickstart](https://github.com/tararouras/kube-prometheus/tree/add_custom_metrics#quickstart) README section.

That's it! Now a combination of deployment,service,horizontal-pod-autoscaler-ServiceMonitor such as [sample-app](https://github.com/tararouras/kube-prometheus/blob/add_custom_metrics/experimental/custom-metrics-api/sample-app.yaml) can be introduced and automatically controlled by k8s controller.

### Further information

As described in [README](https://github.com/tararouras/kube-prometheus/tree/add_custom_metrics):

> This repository collects Kubernetes manifests, Grafana dashboards, and Prometheus rules combined with documentation and scripts to provide easy to operate end-to-end Kubernetes cluster monitoring with Prometheus using the Prometheus Operator.

It provides an easy way to deploy [Prometheus](https://prometheus.io/) monitoring system in a k8s cluster. Additionally k8s metrics API is served by Prometheus using metrics it collects from all k8s resources.

Except from the existing functionality further adaptations have been performed to introduce custom metrics support in k8s. This work is based on the [experimental configuration demonstrated](https://github.com/tararouras/kube-prometheus/tree/add_custom_metrics/experimental/custom-metrics-api).

### Scaling example

After deploying all k8s resources described in `manifests/`, [sample-app](https://github.com/tararouras/kube-prometheus/blob/add_custom_metrics/experimental/custom-metrics-api/sample-app.yaml) can be deployed. Depending on the load of HTTP requests its deployment will be scaled to follow the target of http_requests metric set in HPA resource.

#### sample-app HorizontalPodAutoscaler
```
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta1
metadata:
  name: sample-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sample-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: http_requests
      targetAverageValue: 500m
```

`http_requests` is a custom metrics API metric which is introduced by defining [prometheus-adapter-extra-conf.yaml](https://github.com/tararouras/kube-prometheus/blob/add_custom_metrics/prometheus-adapter-extra-conf.yaml) in [custom-metrics.jsonnet](https://github.com/tararouras/kube-prometheus/blob/add_custom_metrics/custom-metrics.jsonnet). `prometheus-adapter-extra-conf.yaml` became part of [prometheus-adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter) configuration when manifests/ were deployed previously.

#### custom-metrics.jsonnet
```
local kp =
  (import 'kube-prometheus/kube-prometheus.libsonnet') +
  // kubespray-deployed cluster support
  (import 'kube-prometheus/kube-prometheus-kubespray.libsonnet') +
  // enable k8s node ports for monitoring services
  (import 'kube-prometheus/kube-prometheus-node-ports.libsonnet') +
  // Uncomment the following imports to enable its patches
  // (import 'kube-prometheus/kube-prometheus-anti-affinity.libsonnet') +
  // (import 'kube-prometheus/kube-prometheus-managed-cluster.libsonnet') +
  // (import 'kube-prometheus/kube-prometheus-static-etcd.libsonnet') +
  // (import 'kube-prometheus/kube-prometheus-thanos-sidecar.libsonnet') +
  {
    _config+:: {
      namespace: 'monitoring',
      // append extra configuration in prometheus adapter config (config.yaml)
      prometheusAdapter+:: {
        config+:: (importstr 'prometheus-adapter-extra-conf.yaml'),
      },
    },
  };

{ ['00namespace-' + name]: kp.kubePrometheus[name] for name in std.objectFields(kp.kubePrometheus) } +
{ ['0prometheus-operator-' + name]: kp.prometheusOperator[name] for name in std.objectFields(kp.prometheusOperator) } +
{ ['node-exporter-' + name]: kp.nodeExporter[name] for name in std.objectFields(kp.nodeExporter) } +
{ ['kube-state-metrics-' + name]: kp.kubeStateMetrics[name] for name in std.objectFields(kp.kubeStateMetrics) } +
{ ['alertmanager-' + name]: kp.alertmanager[name] for name in std.objectFields(kp.alertmanager) } +
{ ['prometheus-' + name]: kp.prometheus[name] for name in std.objectFields(kp.prometheus) } +
{ ['prometheus-adapter-' + name]: kp.prometheusAdapter[name] for name in std.objectFields(kp.prometheusAdapter) } +
{ ['grafana-' + name]: kp.grafana[name] for name in std.objectFields(kp.grafana) }

```
#### prometheus-adapter-extra-conf.yaml
```
rules:
- seriesQuery: '{__name__="http_requests_total",namespace!="",kubernetes_pod_name!=""}'
  resources:
    template: <<.Resource>>
  name:
    matches: ^(.*)_total$
    as: "${1}"
  metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)'
```

> For information about how to write prometheus-adapter rules check [prometheus-adapter walk-through](https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config-walkthrough.md)


## Add services to control through their metrics

### edge-server

#### Building appropriate images

Edge server application has been dockerized and some metrics have been added through which scaling is going to be demonstrated. Also an edge-server-client application has been dockerized so that edge-server application can be tested.

```
git clone  --branch dockerize-edge-server https://github.com/tararouras/alphabot-ppl.git
cd alphabot-ppl/

# build edge-server:latest
docker build -t edge-server:latest -f Dockerfile.edge_server .

# build edge-server-client:latest
docker build -t edge-server-client:latest -f Dockerfile.edge_server_client .
```

Now `edge-server:latest` image should be "copied" to all kubernetes nodes that can launch a pod using this image. To do that we can use `docker save` and `docker load` commands:

```
# create a file containing edge-server:latest image that can be used to copy
# it to kubernetes nodes
docker save -o edge-server.image edge-server:latest

# assuming kubernetes nodes are on the hosts: master1, master2 and node1
# on which ssh can be used perform the following

# become root (this is only needed to avoid using sudo)
sudo su

# copy edge-server.image file to kubernetes nodes
for n in node1 master2 master1; do scp edge-server.image $n:/tmp/; done

# load edge-server:latest image on these nodes and remove edge-server.image file
for n in node1 master2 master1; do ssh $n docker load -i /tmp/edge-server.image; rm /tmp/edge-server.image; done
```

#### Launching edge-server service with HPA controller

[edge-server.yaml](./edge-server.yaml) file can be used to create edge-server service whose replication factor can be controlled by kubernetes horizontal pod autoscaler.

After launching kube-prometheus manifests for custom-metrics support as explained before (and edge-server metrics rules defined) edge-server resources can be launched with:

```
docker create -f edge-server.yaml
```

This should create `Deployment` with 1 `edge-server` `Pod`, an `edge-server` `Service` and an `HorizontalPodAutoscaler` resources. Also `ServiceMonitor` resource is created that is used by prometheus node-exporter so that prometheus gets notified about this service, to poll for metrics.

The simpliest way to test this setup is to open 2 consoles and execute the following:

In one console monitor HPAs
```
# first console, monitor hpas
# something like the following should be displayed:
# NAME          REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# edge-server   Deployment/edge-server   0/200m    1         10        1          57m
watch -n 2 kubectl get horizontalpodautoscalers.autoscaling
```

In second console run edge-server-client application.
```
# assuming that edge-server-client:latest is already built as described before
docker run -it --rm \
    -e EDGE_SERVER_IP_ADDR=192.168.0.132 \ # replace this with any kubernetes node IP address
    -e EDGE_SERVER_PORT=30800 \ # by default edge-server service creates a NodePort and listens on 30800 on nodes
    -e RUNNING_PERIOD=180 \ # for how long should this command run
    -e RUNNING_INTERVAL=2.5 \ # control the rate of the requests by providing loop interval
    edge-server-client:latest
```

The above should start edge-server-client application which every `RUNNING_INTERVAL` seconds sends an image to the edge-server service. This makes edge-server produce metric `flask_http_request_total` through which this example is scaled with. Then prometheus-adapter converts this metric to a rate metric called `flask_http_requests_per_second` that HPS uses for scaling Deployment.


[^1]: On Fedora 28 the easiest way is to create a user account and set it administrator. Then on first boot perform the following: `sudo su` to switch to root user, `passwd` to set root user password. Then root account access is ready.

[^2]: Static IP addresses can be assigned by creating appropriate ifcfg script and placing it under /etc/sysconfig/network-scripts/. For example the following assigns IP `192.168.1.131` on `enp0s3` interface with MAC address `080027792FB8`:`HWADDR=08:00:27:79:2f:b8 TYPE=Ethernet BOOTPROTO=none IPADDR=192.168.0.131 NETMASK=255.255.255.0 GATEWAY=192.168.0.1 DNS1=1.1.1.1 NAME=enp0s3 DEVICE=enp0s3 ONBOOT=yes`.

[^3]: To change hostname on a host execute with administrator permissions: `hostnamectl set-hostname <hostname>`. Reboot is needed.

[^4]: On control machine switch to `root` account and perform `ssh-copy-id <host>` to copy ssh keys. If no key is created on for root user then first create one using `ssh-keygen`.

[^5]: Edit `/etc/firewalld/zones/FedoraServer.xml` and on `<zone>` tag add attribute `target="ACCEPT"`. Then restart firewalld with `systemctl restart firewalld`.

[^6]: It seems there is a special ansible variable called *controller_mgr_custom_flags* in `roles/kubernetes/master/defaults/main/main.yml` that is supposed to accept extra command-line options to be passed to *kube-controller-manager* but a quick look reveals that this variable is not currently used. So, either `roles/kubernetes/master/templates/kubeadm-config.v1beta1.yaml.j2` has to adapted to take this variable into account or extra parameters have to be passed manually in the above file (controllerManager/extraArgs).

[1]: https://github.com/tararouras/kubespray/tree/extend-for-metrics-scaling "tararouras/kubespray"
[2]: https://kubespray.io/#/ "kubespray documentation"
[3]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/ "Horizontal Pod Autoscaler"
[4]: https://prometheus.io/ "Prometheus"
[5]: https://github.com/tararouras/kube-prometheus/tree/add_custom_metrics "tararouras/kube-prometheus"
