# Kubernetes配置案例

## 1.RBAC

```bash
# Context
You have been asked to create a new ClusterRole for a deployment pipeline and bind it to a specific ServiceAccount scoped to a specific namespace.
# Task
Create a new ClusterRole named deployment-clusterrole,which only allows to create the following resource types:
- Deployment
- StatefulSet
- DaemonSet
Create a new ServiceAccount named cicd-token in the existing namespace app-team1.Bind the new ClusterRole deployment-clusterrole to the new ServiceAccount cicd-token, limited to the namespace app-team1.
```

[解题参考:RBAC鉴权](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/)

```bash
******* Prepare *******
~ kubectl config use-context k8s
```

```bash
******* Answer *******
~ kubectl create clusterrole deployment-clusterrole --verb=create --resource=deployments,statefulsets,daemonsets
~ kubectl -n app-team1 create serviceaccount cicd-token
~ kubectl -n app-team1 create rolebinding deployment-rolebinding --clusterrole=deployment-clusterrole --serviceaccount=app-team1:cicd-token
```

```bash
# 如果没有限定namespace,则需要使用clusterrolebinding
~ kubectl create clusterrolebinding deployment-clusterrolebinding --clusterrole=deployment-clusterrole --serviceaccount=app-team1:cicd-token
# clusterrole和clusterrolebinding不受namespace限制，不需要加 -n namespace
```

## 2.cordon & drain

```bash
# Task
Set the node named ek8s-node-1 as unavailable and reschedule all the pods running on it.
```

[解题参考:Cordon&Drain](https://kubernetes.io/zh/docs/reference/kubectl/cheatsheet/#%E4%B8%8E%E8%8A%82%E7%82%B9%E5%92%8C%E9%9B%86%E7%BE%A4%E8%BF%9B%E8%A1%8C%E4%BA%A4%E4%BA%92)

```bash
******* Prepare *******
~ kubectl config use-context ek8s
******* Answer *******
~ kubectl cordon ek8s-node-1
~ kubectl drain ek8s-node-1 --ignore-daemonsets --delete-emptydir-data --force
```

## 3.Cluster upgrade

```bash
# Task
Given an existing Kubernetes cluster running version 1.22.3, upgrade all the Kubernetes control plane and node components on the master node only to version 1.22.4
You can also expected to upgrade kubelet and kubectl on the master node.
Be sure drain the master node before upgrading it and uncordon it after upgrading. Do not upgrade the worker nodes, etcd,the container manager, the CNI plugin, the DNS server and any other addons.
```

[解题参考:升级集群](https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

```bash
******* Prepare *******
~ kubectl config use-context mk8s
******* Answer *******
# 查看
~ kubectl get nodes
~ kubectl drain mk8s-master-1 --ignore-daemonsets --delete-emptydir-data --force
# 切换到要升级的对应版本的master节点上
~ ssh mk8s-master-1
~ sudo -i
~ apt update
~ apt-cache madison kubeadm | grep 1.22.4
~ apt install kubeadm=1.22.4-00 -y
~ kubeadm version
~ kubeadm upgrade plan
# 重点是要不要忽略etcd，注意审题，忽略时参数--etcd-upgrade=false
~ kubeadm upgrade apply 1.22.4 --etcd-upgrade=false
~ apt-get install kubelet=1.22.4-00 -y
~ apt-get install kubectl=1.22.4-00 -y
~ kubelet --version
~ kubectl version
~ exit
# 退出节点后执行
~ kubectl uncordon mk8s-master-1
```

## 4.ETCD backup and restore

```bash
# Task
First, create a snapshot of the existing etcd instance running at https://127.0.0.1:2379, saving the snapshot to /srv/data/etcd-snapshot.db.Creating a snapshot of the given instance is expected to complete in seconds.If the operation seems to hang, something is likely wrong with your command. Use ctrl+c to cancel the operation and try again.

Next, restore an existing, previous snapshot located at /var/lib/backup/etcd-snapshot-previous.db. The following TLS certificates/key are supplied for connecting to the server with etcdctl:
- CA certificate: /opt/KUIN00601/ca.crt
- Client certificate: /opt/KUIN00601/etcd-client.crt
- Client key: /opt/KUIN00601/etcd-client.key
```

[解题参考:ETCD备份与恢复](https://kubernetes.io/zh/docs/tasks/administer-cluster/configure-upgrade-etcd/#%E4%BD%BF%E7%94%A8-etcdctl-%E9%80%89%E9%A1%B9%E7%9A%84%E5%BF%AB%E7%85%A7)

```bash
******* Prepare *******
# 如果没有etcdctl命令，安装一下etcd-client
~ apt-get install etcd-client
******* Answer *******
~ ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/opt/KUIN00601/ca.crt --cert=/opt/KUIN00601/etcd-client.crt --key=/opt/KUIN00601/etcd-client.key snapshot save /srv/data/etcd-snapshot.db
~ ETCDCTL_API=3 etcdctl snapshot restore /var/lib/backup/etcd-snapshot-previous.db
```

## 5.NetworkPolicy

```bash
******* Type1 *******
# Task
Create a new NetworkPolicy name allow-port-from-namespace that allows Pods in the existing namespace internal to connect to port 9000 of other Pods in the same namespace.

Ensure that the new NetworkPolicy:
- does not allow access to Pods not listening on port 9000
- does not allow access from Pods not in namespace internal
```

[解题参考:网络策略](https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/#networkpolicy-resource>)

```bash
******* Prepare *******
~ kubectl config use-context hk8s
******* Answer1 *******
~ vim networkpolicy.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: internal
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 9000
~ kubectl apply -f networkpolicy.yaml
```

```bash
******* Type2 *******
# Task
Create a new NetworkPolicy named allow-port-from-namespace in the existing namesapce internal that allows Pods in namespace big-corp to connect to port 9000 of the Pods in namespace internal.

Ensure that the new NetworkPolicy:
- does not allow access to Pods not listening on port 9000
- does not allow access from Pods not in namespace big-corp
```

```bash
******* Prepare *******
~ kubectl config use-context hk8s
******* Answer2 *******
~ vim networkpolicy.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: internal
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          name: big-corp
    ports:
    - protocol: TCP
      port: 9000
~ kubectl apply -f networkpolicy.yaml
# 查看一下big-corp这个namespace是否有name=big-corp这个标签，如果没有则创建一下
~ kubectl label ns big-corp name=big-corp
```

```bash
******* Type3 *******
# Task
Create a new NetworkPolicy named allow-port-from-namespace in the existing namesapce internal that allows Pods in namespace internal to connect to port 9000 of the Pods in namespace big-corp.

Ensure that the new NetworkPolicy:
- does not allow access to Pods not listening on port 9000
- does not allow access from Pods not in namespace internal
```

```bash
******* Prepare *******
~ kubectl config use-context hk8s
******* Answer3 *******
~ vim networkpolicy.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: internal
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 9000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: big-corp
    ports:
    - protocol: TCP
      port: 9000
~ kubectl apply -f networkpolicy.yaml
# 查看一下big-corp这个namespace是否有name=big-corp这个标签，如果没有则创建一下
~ kubectl label ns big-corp name=big-corp
```

## 6.Service

```bash
# Task
Reconfigure the existing deployment front-end and add a port specification name http exposing port 80/tcp of the existing container nginx.

Creat a new service named front-end-svc exposing the container port http.

Configure the new service to also expose the individual Pods via a NodePort on the node on which they are scheduled.
```

[解题参考:创建Service](https://kubernetes.io/zh/docs/concepts/services-networking/connect-applications-service/)

```bash
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl edit deploy front-end
...
      containers:
      - name: nginx
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
...
~ kubectl expose deploy front-end --name=front-end-svc  --port=80 --target-port=http --type=NodePort
```

## 7.Ingress

```bash
# Task
Create a new nginx Ingress resource as follows:
- Name: pong
- Namespace: ing-internal
- Exposing service hi on path /hi using service port 5678

The availability of service hi can be checked using the following command, which shoud return hi:
[student@node-1] $ curl -kL <INTERNAL_IP>/hi
```

[解题参考:Ingress](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#the-ingress-resource)

```bash
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ vim ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pong
  namespace: ing-internal
spec:
  rules:
  - http:
      paths:
      - path: /hi
        pathType: Prefix
        backend:
          service:
            name: hi
            port:
              number: 5678
~ kubectl apply -f ingress.yaml
```

## 8.Replicas Deployment

```bash
# Task
Scale the deployment presentation to 3 pods.
```

[解题参考:资源伸缩](https://kubernetes.io/zh/docs/reference/kubectl/cheatsheet/#%E5%AF%B9%E8%B5%84%E6%BA%90%E8%BF%9B%E8%A1%8C%E4%BC%B8%E7%BC%A9)

```bash
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl scale deployment presentation --replicas=3
```

## 9.Schedule Pod

```bash
# Task
Schedule a pod as follows:
- Name: nginx-kusc00401
- Image: nginx
- Node selector: disk=spinning
```

[解题参考:调度Pod](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/)

```bash
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ vim nginx-kusc00401.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-kusc00401
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disk: spinning
~ kubectl apply -f nginx-kusc00401.yaml
```

## 10.Check node

```bash
# Task
Check to see how many nodes are ready(not including nodes tainted NoSchedule) and write the number to /opt/KUSC00402/kusc00402.txt
```

```bash
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl get nodes | grep -i ready
~ kubectl describe nodes | grep -i taint
~ echo number > /opt/KUSC00402/kusc00402.txt
```

## 11.Images Pod

```bash
Create a pod named kucc8 with a single app container for each of the following images rinning inside(there may be between 1 and 4 images specified):
nginx + redis + memcached + consul
```

[解题参考:创建Pod](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/#%E6%AD%A5%E9%AA%A4%E4%BA%8C-%E6%B7%BB%E5%8A%A0-nodeselector-%E5%AD%97%E6%AE%B5%E5%88%B0-pod-%E9%85%8D%E7%BD%AE%E4%B8%AD)

```bash
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ vim kucc8.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  - name: redis
    image: redis
  - name: memcached
    image: memcached
  - name: consul
    image: consul
~ kubectl apply -f kucc8.yaml
```

## 12.PV

```bash
# Task
Create a persistent volume with name app-config, of capacity 1Gi and access mode ReadOnlyMany. The type of volume is hostPath and its location is /srv/app-config
```

[解题参考:创建PV](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#%E5%88%9B%E5%BB%BA-persistentvolume)

```bash
******* Prepare *******
~ kubectl config use-context hk8s
******* Answer *******
~ vim pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-config
  labels:
    type: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadOnlyMany
  hostPath:
    path: "/srv/app-config"
~ kubectl apply -f pv.yaml
```

## 13.PVC

```bash
# Task
Create a new PersistentVloumeClaim:
- Name: pv-volume
- Class: csi-hostpath-sc
- Capacity: 10Mi

Create a new Pod which mounts the PersistentVolumeClaim as a volume:
- Name: web-server
- Image: nginx
- Mount path: /usr/share/nginx/html

Configure the new pod to have ReadWriteOnce access to the volume

Finally, using kubectl edit or kubectl patch expand the PersistentVolumeClaim to a capacity of 70Mi and record the change
```

[解题参考:创建PVC](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#%E5%88%9B%E5%BB%BA-persistentvolumeclaim)

```bash
******* Prepare *******
~ kubectl config use-context ok8s
******* Answer *******
~ vim pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-volume
spec:
  storageClassName: csi-hostpath-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
~ kubectl apply -f pvc.yaml
~ vim pvc-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  volumes:
    - name: pv-volume
      persistentVolumeClaim:
        claimName: pv-volume
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-volume
~ kubectl apply -f pvc-pod.yaml
# 编辑大小为70Mi
~ kubectl edit pvc pv-volume --record
```

## 14.Save Pod Error Log

```bash
# Task
Monitor the logs of pod bar and:
- Extract log lines corresponding to error unable-to-access-website
- Write them to /opt/KUTR00101/bar
```

```bash
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl logs bar | grep "unable-to-access-website" >> /opt/KUTR00101/bar
```

## 15.SideCar

```bash
# Context
Without changing its existing containers, anexisting Pod needs to be integrated into Kubernetes built-in logging architecure(e.g. kubectl logs). Adding a streaming sidecar conatainer is a good and common way to accomplish this requirement.
# Task
Add a busybox sidecar container to the existing Pod big-corp-app. The new sidecar container has to run the following command: /bin/sh -c tail -n+1 /var/log/big-corp-app.log
Use a volume mount named logs to make the file /var/log/big-corp-app.log available to sidecar container.

Do not modify the existing container. Do not modify the path of the log file, both containers must access it at /var/log/big-corp-app.log
```

[解题参考:日志分离Sidecar](https://kubernetes.io/zh/docs/concepts/cluster-administration/logging/#sidecar-container-with-logging-agent)

```bash
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl get pod big-corp-app -oyaml > sidecar.yaml
~ vim sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: big-corp-app
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      mkdir /var/log;
      while true;
      do
        echo "$(date) INFO $i" >> /var/log/big-corp-app.log;
        i=$((i+1));
        sleep 1;
      done
    # 添加下面内容
    volumeMounts:
    - name: logs
      mountPath: /var/log
  - name: busybox
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 /var/log/big-corp-app.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
# 删除旧的，创建新的sidecar
~ kubectl delete -f sidecar.yaml
~ kubectl apply -f sidecar.yaml
```

## 16.Top

```bash
# Task
From the pod label name=cpu-loader, find pods running high CPU wordloads and write the name of the pod consuming most CPU to the file /opt/KUTR00401/KUTR00401.txt(which already exists)
```

```bash
******* Prepare *******
~ kubectl config use-context k8s
******* Answer *******
~ kubectl top pod -A -l name=cpu-loader
# 将查到的占用CPU最大的pod的名字写到文件中，假设coredns-54d67798b7-hl8xc这个pod占用最高
~ echo "coredns-54d67798b7-hl8xc" >> /opt/KUTR00401/KUTR00401.txt
```

## 17.Node NotReady Check

```bash
# Task
A Kubernetes work node named wk8s-node-0 is in state NotReady. Investigate why this is the case, and perform any appropriate step to bring the node to a Ready state., ensuring that any changes are made permanent.

You can ssh to the failed node using: ssh wk8s-node-0
You can assume elevated privileges on the node with the following command: sudo -i
```

```bash
******* Prepare *******
~ kubectl config use-context wk8s
******* Answer *******
~ ssh wk8s-node-0
~ sudo -i
~ systemctl status kubelet
~ systemctl start kubelet
~ systemctl enable kubelet
```

