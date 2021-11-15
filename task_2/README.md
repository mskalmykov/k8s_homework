# Task 2
### ConfigMap & Secrets
```bash
$ kubectl create secret generic connection-string --from-literal=DATABASE_URL=postgres://connect --dry-run=client -o yaml > secret.yaml
```
secret.yaml contents:
```yaml
apiVersion: v1
data:
  DATABASE_URL: cG9zdGdyZXM6Ly9jb25uZWN0
kind: Secret
metadata:
  creationTimestamp: null
  name: connection-string
```
```bash
$ kubectl create configmap user --from-literal=firstname=firstname --from-literal=lastname=lastname --dry-run=client -o yaml > cm.yaml
```
cm.yaml contents:
```yaml
apiVersion: v1
data:
  firstname: firstname
  lastname: lastname
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: user
```
```bash
$ kubectl apply -f secret.yaml
$ kubectl apply -f cm.yaml
$ kubectl apply -f pod.yaml
```
## Check env in pod
```bash
$ kubectl exec -it nginx -- sh -c printenv
```
### Output
```bash
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
DATABASE_URL=postgres://connect
HOSTNAME=nginx
HOME=/root
lastname=lastname
PKG_RELEASE=1~bullseye
TERM=xterm
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
NGINX_VERSION=1.21.4
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
NJS_VERSION=0.7.0
KUBERNETES_PORT_443_TCP_PROTO=tcp
firstname=firstname
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
```
### Create deployment with simple application
```bash
$ kubectl apply -f nginx-configmap.yaml
$ kubectl apply -f deployment.yaml
```
### Get pod ip address
```bash
$ kubectl get pods -o wide
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
web-5584c6c5c6-4wv4v   1/1     Running   0          54s   172.17.0.16   minikube   <none>           <none>
web-5584c6c5c6-8lckl   1/1     Running   0          54s   172.17.0.15   minikube   <none>           <none>
web-5584c6c5c6-pnkz7   1/1     Running   0          54s   172.17.0.14   minikube   <none>           <none>
```
### Try connect to pod with curl (curl pod_ip_address). What happens?
* From you PC
```bash
curl: (7) Failed to connect to 172.17.0.14 port 80: No route to host
```
* From minikube (minikube ssh)
```bash
docker@minikube:~$ curl 172.17.0.14
web-5584c6c5c6-pnkz7
```
* From another pod (kubectl exec -it $(kubectl get pod |awk '{print $1}'|grep web-|head -n1) bash)
```bash
$ kubectl exec -it web-5584c6c5c6-4wv4v -- curl 172.17.0.15
web-5584c6c5c6-8lckl
```
### Create service (ClusterIP)
The command that can be used to create a manifest template
```bash
$ kubectl expose deployment/web --type=ClusterIP --dry-run=client -o yaml > service_template.yaml
```
service_template.yaml contents:
```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: web
  name: web
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: web
  type: ClusterIP
status:
  loadBalancer: {}
```
Apply manifest
```bash
$ kubectl apply -f service_template.yaml
```
Get service CLUSTER-IP
```bash
$ kubectl get svc -o wide
NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
web    ClusterIP   10.99.177.156   <none>        80/TCP    44s   app=web
```
### Try to connect to service (curl service_ip_address). What happens?
* From you PC
```bash
curl: (28) Failed to connect to 10.99.177.156 port 80: Connection timed out
```
* From minikube (minikube ssh) (run the command several times)
```bash
$ for i in {1..10} ; do curl 10.99.177.156 ; done
web-5584c6c5c6-pnkz7
web-5584c6c5c6-pnkz7
web-5584c6c5c6-pnkz7
web-5584c6c5c6-4wv4v
web-5584c6c5c6-4wv4v
web-5584c6c5c6-pnkz7
web-5584c6c5c6-8lckl
web-5584c6c5c6-8lckl
web-5584c6c5c6-8lckl
web-5584c6c5c6-pnkz7
```
* From another pod (kubectl exec -it $(kubectl get pod |awk '{print $1}'|grep web-|head -n1) bash) (run the command several times)
```bash
$ kubectl exec -it web-5584c6c5c6-4wv4v -- curl 10.99.177.156
web-5584c6c5c6-8lckl
$ kubectl exec -it web-5584c6c5c6-4wv4v -- curl 10.99.177.156
web-5584c6c5c6-pnkz7
```
### NodePort
```bash
$ kubectl apply -f service-nodeport.yaml
$ kubectl get service
```
Output:
```bash
NAME     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
web      ClusterIP   10.99.177.156   <none>        80/TCP         30m
web-np   NodePort    10.108.141.36   <none>        80:30583/TCP   6s
```
Note how port is specified for a NodePort service
### Checking the availability of the NodePort service type
```bash
$ minikube ip
192.168.49.2

$ for i in {1..5}; do curl 192.168.49.2:30583 ; done
web-5584c6c5c6-8lckl
web-5584c6c5c6-4wv4v
web-5584c6c5c6-4wv4v
web-5584c6c5c6-4wv4v
web-5584c6c5c6-pnkz7
```
### Headless service
```bash
$ kubectl apply -f service-headless.yaml
service/web-headless created
$ kubectl get svc
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
web            ClusterIP   10.99.177.156   <none>        80/TCP         33m
web-headless   ClusterIP   None            <none>        80/TCP         8s
web-np         NodePort    10.108.141.36   <none>        80:30583/TCP   3m28s
```
### DNS
Connect to any pod.
Compare the IP address of the DNS server in the pod and the DNS service of the Kubernetes cluster.
```bash
$ kubectl exec -it web-5584c6c5c6-4wv4v -- cat /etc/resolv.conf | grep nameserver
nameserver 10.96.0.10
$ kubectl get svc -n kube-system | grep dns
kube-dns         ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   3d21h
```
* Compare headless and clusterip
Inside the pod run nslookup to normal clusterip and headless. Compare the results.
You will need to create pod with dnsutils.
```bash
$ nslookup web
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   web.task-2.svc.cluster.local
Address: 10.99.177.156

$ nslookup web-headless
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   web-headless.task-2.svc.cluster.local
Address: 172.17.0.16
Name:   web-headless.task-2.svc.cluster.local
Address: 172.17.0.15
Name:   web-headless.task-2.svc.cluster.local
Address: 172.17.0.14
```
### [Ingress](https://kubernetes.github.io/ingress-nginx/deploy/#minikube)
Enable Ingress controller
```bash
$ minikube addons enable ingress
    ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
    ▪ Using image k8s.gcr.io/ingress-nginx/controller:v1.0.4
    ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
🔎  Verifying ingress addon...
🌟  The 'ingress' addon is enabled
```
Let's see what the ingress controller creates for us
```bash
$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create--1-x98jc     0/1     Completed   0          6h4m
ingress-nginx-admission-patch--1-wks2q      0/1     Completed   1          6h4m
ingress-nginx-controller-5f66978484-qhpp2   1/1     Running     0          6h4m

$ kubectl get pod $(kubectl get pod -n ingress-nginx|grep ingress-nginx-controller|awk '{print $1}') -n ingress-nginx -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2021-11-14T12:06:10Z"
  generateName: ingress-nginx-controller-5f66978484-
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    gcp-auth-skip-secret: "true"
    pod-template-hash: 5f66978484
... lots of output omitted
```
Create Ingress
```bash
$ kubectl apply -f ingress.yaml
ingress.networking.k8s.io/ingress-web created

$ $ for i in {1..3} ; do curl $(minikube ip) ; done
web-5584c6c5c6-pnkz7
web-5584c6c5c6-8lckl
web-5584c6c5c6-4wv4v
```
### Homework
* In Minikube in namespace kube-system, there are many different pods running. Your task is to figure out who creates them, and who makes sure they are running (restores them after deletion).
>_My guess is that it is *kubelet* process who creates and maintains other control panel pods in kube-system namespace._
* Implement Canary deployment of an application via Ingress. Traffic to canary deployment should be redirected if you add "canary:always" in the header, otherwise it should go to regular deployment.
  - Create production environment:
```bash
$ kubectl create namespace prod
$ kubectl -n prod apply -f education/task_2/nginx-configmap.yaml
$ kubectl -n prod edit cm nginx-configmap
Change return string to:
    return 200 'Prod: $hostname\n';
$ kubectl -n prod apply -f education/task_2/deployment.yaml
$ kubectl -n prod apply -f education/task_2/service_template.yaml

$ kubectl -n prod get pods
NAME                   READY   STATUS    RESTARTS   AGE
web-5584c6c5c6-2xfvj   1/1     Running   0          28m
web-5584c6c5c6-sdncv   1/1     Running   0          28m
web-5584c6c5c6-t7dbk   1/1     Running   0          28m

$ kubectl -n prod get svc
NAME   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.104.245.187   <none>        80/TCP    31m
```
  - Create canary environment:
```bash
$ kubectl create ns canary
$ kubectl -n canary apply -f education/task_2/nginx-configmap.yaml
$ kubectl -n canary edit cm nginx-configmap
Change return string to:
    return 200 'Canary: $hostname\n';
$ kubectl -n canary apply -f education/task_2/deployment.yaml
$ kubectl -n canary apply -f education/task_2/service_template.yaml

$ kubectl -n canary get pods
NAME                   READY   STATUS    RESTARTS   AGE
web-5584c6c5c6-7926r   1/1     Running   0          32m
web-5584c6c5c6-fsq9m   1/1     Running   0          32m
web-5584c6c5c6-pkmff   1/1     Running   0          32m

$ kubectl -n canary get svc
NAME   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.100.224.204   <none>        80/TCP    31m
```
  - Create ingress for canary (with custom annotations):
```bash
$ kubectl -n canary apply -f education/task_2/ingress.yaml
$ kubectl -n canary edit ingress ingress-web
Add to the annotations dictionary:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-by-header: canary
```
  - Create ingress for prod (without annotations):
```bash
$ kubectl -n prod apply -f education/task_2/ingress.yaml
```
  - Check prod:
```bash
$ minikube ip
192.168.49.2

$ for i in {1..10} ; do curl 192.168.49.2 ; done
Prod: web-5584c6c5c6-2xfvj
Prod: web-5584c6c5c6-sdncv
Prod: web-5584c6c5c6-t7dbk
Prod: web-5584c6c5c6-2xfvj
Prod: web-5584c6c5c6-sdncv
Prod: web-5584c6c5c6-t7dbk
Prod: web-5584c6c5c6-2xfvj
Prod: web-5584c6c5c6-sdncv
Prod: web-5584c6c5c6-t7dbk
Prod: web-5584c6c5c6-2xfvj
```
  - Check canary:
```bash
$ for i in {1..10} ; do curl -H "canary:always" 192.168.49.2 ; done
Canary: web-5584c6c5c6-7926r
Canary: web-5584c6c5c6-fsq9m
Canary: web-5584c6c5c6-pkmff
Canary: web-5584c6c5c6-7926r
Canary: web-5584c6c5c6-7926r
Canary: web-5584c6c5c6-fsq9m
Canary: web-5584c6c5c6-fsq9m
Canary: web-5584c6c5c6-pkmff
Canary: web-5584c6c5c6-7926r
Canary: web-5584c6c5c6-pkmff
```
* Set to redirect a percentage of traffic to canary deployment.
```bash
$ kubectl -n canary edit ingress ingress-web
Replace annotation key:
    nginx.ingress.kubernetes.io/canary-by-header: canary
with this one:
    nginx.ingress.kubernetes.io/canary-weight: "20"
```
Check if it works:
```bash
$ for i in {1..20} ; do curl 192.168.49.2 ; done
Prod: web-5584c6c5c6-t7dbk
Prod: web-5584c6c5c6-2xfvj
Prod: web-5584c6c5c6-sdncv
Prod: web-5584c6c5c6-t7dbk
Prod: web-5584c6c5c6-2xfvj
Canary: web-5584c6c5c6-7926r
Prod: web-5584c6c5c6-t7dbk
Prod: web-5584c6c5c6-sdncv
Prod: web-5584c6c5c6-2xfvj
Prod: web-5584c6c5c6-t7dbk
Prod: web-5584c6c5c6-2xfvj
Canary: web-5584c6c5c6-fsq9m
Prod: web-5584c6c5c6-sdncv
Canary: web-5584c6c5c6-7926r
Prod: web-5584c6c5c6-t7dbk
Prod: web-5584c6c5c6-2xfvj
Canary: web-5584c6c5c6-fsq9m
Prod: web-5584c6c5c6-sdncv
Prod: web-5584c6c5c6-t7dbk
Prod: web-5584c6c5c6-2xfvj
```
