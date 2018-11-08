# Install Prometheus and Store Data On NFS  in Oracle OCI

###  Requirement:
To monitor and get metrics of containerized environment managed by of K8S,we are going to use Prometheus and Grafana. They can provide visualized dashboard for K8S systems with useful charts.Refer [Oracle Guide](https://cloudnative.oracle.com/template.html#observability-and-analysis/telemetry/prometheus/prometheus101.md)

### Install Helm
* Download Helm [latest version](https://github.com/helm/helm/releases/tag/v2.11.0) today from [link](https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz)
* We use linux version of Helm. More details refer [github doc](https://github.com/helm/helm#install)
* tar zxvf helm-v2.11.0-linux-amd64.tar.gz .
* Find the helm binary in the unpacked directory, and move it to its desired destination (mv linux-amd64/helm /bin/helm)
* run helm to see correct help output below. As helm is written by GO, 1 binary has all things needed. No other setup

```
$ helm
The Kubernetes package manager
To begin working with Helm, run the 'helm init' command:
        $ helm init
This will install Tiller to your running Kubernetes cluster.
It will also set up any necessary local configuration.
........
Use "helm [command] --help" for more information about a command.
```
### Set correct account and role for proper permissions
* kubectl -n kube-system create sa tiller
* kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller

### Install and Tiller Prometheus operator (set proxy if necessary)
* helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/
* helm init --service-account tiller
* helm install --namespace monitoring coreos/prometheus-operator

### Install Prometheus on different storages
* We can choose NFS or default StorageClass for Prometheus data.(only choose one)
* Use NFS to store Prometheus Data
 * We need to create Filesystem(NFS) and mount targets in OCI first, then we can let K8S to mount them and use . Please refer official [Oracle Doc](https://docs.cloud.oracle.com/iaas/Content/File/Tasks/creatingfilesystems.htm)
   * create pv in K8S,yaml is like

  ```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: livesqlsb-prometheus-pv-name
  namespace: monitoring
  labels:
    app: livesqlsb-prometheus
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteOnce # required
  nfs:
    server: 123.123.123.123
    path: "/Prometheus-Data"
  ```

 * We need to disable default StorageClass (in this case "oci" ) that is automatically created for certain Cloud Providers.refer [github doc](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/storage.md)
   * kubectl create -f <below yaml> to disable it

     ```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: oci
  annotations:
    # disable this default storage class by setting this annotation to false.
         storageclass.beta.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/oci
    ```
    * When you see error pod error "prometheus-kube-prometheus-0" is "CrashLoopBackOff"
    * Use "kubectl logs prometheus-kube-prometheus-0 prometheus" to get more details, you may see error "Opening storage failed, permission denied".
    * In this case we need to chmod 775 prometheus-db on NFS .See details in [github issue](https://github.com/coreos/prometheus-operator/issues/830)
    * After that "kubectl delete pod prometheus-kube-prometheus-0" will restart pod successfully. You can see files created in ./prometheus-db/wal/
* Get some Kubernetes manifests for Grafana dashboards, and Prometheus rules that allow us to operate Kubernetes
  * wget https://github.com/HenryXie1/Install-Prometheus-Grafana-and-Store-Data-On-NFS-or-StorageClass-in-Oracle-OCI/raw/master/values_nfs.yaml
  * Replace "app: livesqlsb-prometheus" with your labels accordingly to match the PV we created above
* Install it via below command
  * helm install coreos/kube-prometheus --name kube-prometheus --namespace monitoring --values values_nfs.yaml
* Check if all related Pods are running

```
  kubectl get po --namespace monitoring
NAME                                                        READY     STATUS    RESTARTS   AGE
alertmanager-kube-prometheus-0                              2/2       Running   0          1h
intentional-orangutan-prometheus-operator-bdbf464f6-rcklw   1/1       Running   0          1d
kube-prometheus-exporter-kube-state-85975c8577-j9hs9        2/2       Running   0          1h
kube-prometheus-exporter-node-b7dkv                         1/1       Running   0          1h
kube-prometheus-exporter-node-pbl9t                         1/1       Running   0          1h
kube-prometheus-grafana-57d5b4d79f-7mdtf                    2/2       Running   0          1h
prometheus-kube-prometheus-0                                3/3       Running   1          1h
```
 
* Expose Prometheus service to NodePort
  * kubectl edit svc kube-prometheus -n monitoring
   * At the bottom, replace "type: ClusterIP" with "type: NodePort", save it
   * kubectl get svc -n monitoring , will see which port is exposed ie http://xxx.xxx.xxx.xxx:30304/
* Expose Grafana service to NodePort
   * kubectl edit svc kube-prometheus-grafana -n monitoring
   * At the bottom, replace "type: LoadBalancer" with "type: NodePort", save it
   * kubectl get svc -n monitoring , will see which port is exposed ie http://xxx.xxx.xxx.xxx:30465/
* You may hit 403 error and no route to host issues
  * [How To Fix "server returned HTTP status 403 Forbidden" in Prometheus](http://www.henryxieblogs.com/2018/11/how-to-fix-server-returned-http-status.html)
  * [How To Fix "No Route to Host" in Prometheus node-exporter](http://www.henryxieblogs.com/2018/11/how-to-fix-no-route-to-host-in.html)

### Clean Prometheus in case we mess up
* helm ls --all kube-prometheus
* helm del --purge kube-prometheus

