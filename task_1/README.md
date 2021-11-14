# Task 1.1
## Verify kubectl installation
```bash
$ kubectl version --client
```
Output:
```bash
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.3", GitCommit:"c92036820499fedefec0f847e2054d824aea6cd1", GitTreeState:"clean", BuildDate:"2021-10-27T18:41:28Z", GoVersion:"go1.16.9", Compiler:"gc", Platform:"linux/amd64"}
```

## Start minikube
```bash
$ minikube start
```
Output:
```bash
😄  minikube v1.24.0 on Debian 11.1
✨  Using the docker driver based on existing profile
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔄  Restarting existing docker container for "minikube" ...
🐳  Preparing Kubernetes v1.22.3 on Docker 20.10.8 ...
🔎  Verifying Kubernetes components...
    ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
    ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
    ▪ Using image k8s.gcr.io/ingress-nginx/controller:v1.0.4
🔎  Verifying ingress addon...
🌟  Enabled addons: storage-provisioner, default-storageclass, ingress
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

## Get information about cluster
```bash
$ kubectl cluster-info
```
Output:
```bash
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
## get information about available nodes
```bash
$ kubectl get nodes
```
Output:
```bash
NAME       STATUS   ROLES                  AGE     VERSION
minikube   Ready    control-plane,master   3d11h   v1.22.3
```

# Install [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```
Output:
```
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard unchanged
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard unchanged
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
Warning: spec.template.metadata.annotations[seccomp.security.alpha.kubernetes.io/pod]: deprecated since v1.19; use the "seccompProfile" field instead
deployment.apps/dashboard-metrics-scraper created
```
# Check kubernetes-dashboard ns
```bash
$ kubectl get pod -n kubernetes-dashboard
```
Output
```bash
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-856586f554-qxp8c   1/1     Running   0          85s
kubernetes-dashboard-67484c44f6-jktnv        1/1     Running   0          85s
```
# Install [Metrics Server](https://github.com/kubernetes-sigs/metrics-server#deployment)
```bash
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Output:
```
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader unchanged
clusterrole.rbac.authorization.k8s.io/system:metrics-server unchanged
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader unchanged
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator unchanged
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server unchanged
service/metrics-server unchanged
deployment.apps/metrics-server configured
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io unchanged
```

## Update deployment
```bash
$ kubectl edit -n kube-system deployment metrics-server
```
```bash
spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-insecure-tls
        - --kubelet-use-node-status-port
```
Output:
```bash
deployment.apps/metrics-server edited
```

# Connect to Dashboard
## Get token
### Manual

```bash
$ kubectl describe sa -n kube-system default

Name:                default
Namespace:           kube-system
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-m5n46
Tokens:              default-token-m5n46
Events:              <none>

# copy token name
$ kubectl get secrets -n kube-system default-token-m5n46 -o yaml
...
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklreEtNVE5mTWpaSFVFUlhXa3gzWjJWc00xTm5ObTFaZG5SbWMwcGFha2RDVm5Wa1puZDRiR2szTlZraWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUpyZFdKbExYTjVjM1JsYlNJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZqY21WMExtNWhiV1VpT2lKa1pXWmhkV3gwTFhSdmEyVnVMVzAxYmpRMklpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVibUZ0WlNJNkltUmxabUYxYkhRaUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sY25acFkyVXRZV05qYjNWdWRDNTFhV1FpT2lJNU1qVTROall3TXkwNE1ESTVMVFF5WXprdFlURTNaUzFsTkRJMlpHRmpNRFE0TVRVaUxDSnpkV0lpT2lKemVYTjBaVzA2YzJWeWRtbGpaV0ZqWTI5MWJuUTZhM1ZpWlMxemVYTjBaVzA2WkdWbVlYVnNkQ0o5Lkt3TnhzRVAtel9wVUxPRlp4UnRFeHVaTF9xQjFsQlQ0R3ZvbjZUbjBtWjBabEM3bmhGR3NOd3hEWS0yZTFuRS14eVdoRDFFWm5SOVpkVk9RWEpVUllCeHNoNGR0RU1SUXRGTkxJcW80U0x4bEI4MUNqRnBXUVk0U1gtSjRsQXNSVmJpbTRvRG9vYW1LQk9iLXdTalZDeXhKWk9mSE1ST3FYOHU3b0d2WDBfWFV0R2VJMWlUdGVLcGt3ZUFyeW1CcmlEaGZnbk1lckxHNFpmd1Z1TFhvZk84MS1XYWZPLThlaWZrMlBlZFpjcE9xSzc2QlNPOFliZHBwWVdHY2dHNWhZckJuNlZZeE1xelRCS25tRllfUzZKeEhNRk1uTXhRSi0zMlRyVzUzWVRlY2E4N2E4Ni1ORVlvNjFNeDJZVHlYS3lHckVMTnFXZnZKbTNHb2JuQmNtQQ==
...
$ echo -n "ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklreEtNVE5mTWpaSFVFUlhXa3gzWjJWc00xTm5ObTFaZG5SbWMwcGFha2RDVm5Wa1puZDRiR2szTlZraWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUpyZFdKbExYTjVjM1JsYlNJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZqY21WMExtNWhiV1VpT2lKa1pXWmhkV3gwTFhSdmEyVnVMVzAxYmpRMklpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVibUZ0WlNJNkltUmxabUYxYkhRaUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sY25acFkyVXRZV05qYjNWdWRDNTFhV1FpT2lJNU1qVTROall3TXkwNE1ESTVMVFF5WXprdFlURTNaUzFsTkRJMlpHRmpNRFE0TVRVaUxDSnpkV0lpT2lKemVYTjBaVzA2YzJWeWRtbGpaV0ZqWTI5MWJuUTZhM1ZpWlMxemVYTjBaVzA2WkdWbVlYVnNkQ0o5Lkt3TnhzRVAtel9wVUxPRlp4UnRFeHVaTF9xQjFsQlQ0R3ZvbjZUbjBtWjBabEM3bmhGR3NOd3hEWS0yZTFuRS14eVdoRDFFWm5SOVpkVk9RWEpVUllCeHNoNGR0RU1SUXRGTkxJcW80U0x4bEI4MUNqRnBXUVk0U1gtSjRsQXNSVmJpbTRvRG9vYW1LQk9iLXdTalZDeXhKWk9mSE1ST3FYOHU3b0d2WDBfWFV0R2VJMWlUdGVLcGt3ZUFyeW1CcmlEaGZnbk1lckxHNFpmd1Z1TFhvZk84MS1XYWZPLThlaWZrMlBlZFpjcE9xSzc2QlNPOFliZHBwWVdHY2dHNWhZckJuNlZZeE1xelRCS25tRllfUzZKeEhNRk1uTXhRSi0zMlRyVzUzWVRlY2E4N2E4Ni1ORVlvNjFNeDJZVHlYS3lHckVMTnFXZnZKbTNHb2JuQmNtQQ==" | base64 -d

Output:
eyJhbGciOiJSUzI1NiIsImtpZCI6IkxKMTNfMjZHUERXWkx3Z2VsM1NnNm1ZdnRmc0paakdCVnVkZnd4bGk3NVkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZWZhdWx0LXRva2VuLW01bjQ2Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlZmF1bHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI5MjU4NjYwMy04MDI5LTQyYzktYTE3ZS1lNDI2ZGFjMDQ4MTUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06ZGVmYXVsdCJ9.KwNxsEP-z_pULOFZxRtExuZL_qB1lBT4Gvon6Tn0mZ0ZlC7nhFGsNwxDY-2e1nE-xyWhD1EZnR9ZdVOQXJURYBxsh4dtEMRQtFNLIqo4SLxlB81CjFpWQY4SX-J4lAsRVbim4oDooamKBOb-wSjVCyxJZOfHMROqX8u7oGvX0_XUtGeI1iTteKpkweArymBriDhfgnMerLG4ZfwVuLXofO81-WafO-8eifk2PedZcpOqK76BSO8YbdppYWGcgG5hYrBn6VYxMqzTBKnmFY_S6JxHMFMnMxQJ-32TrW53YTeca87a86-NEYo61Mx2YTyXKyGrELNqWfvJm3GobnBcmA
```
# Same thing in one command
```bash
$ kubectl get secrets -n kube-system $(kubectl describe sa -n kube-system default|grep Tokens|awk '{print $2}') -o yaml|grep -E "^[[:space:]]*token:"|awk '{print $2}'|base64 -d
```
Output is the same.

### Auto
```bash
export SECRET_NAME=$(kubectl get sa -n kube-system default -o jsonpath='{.secrets[0].name}')
export TOKEN=$(kubectl get secrets -n kube-system $SECRET_NAME -o jsonpath='{.data.token}' | base64 -d)
echo $TOKEN
```
Output is the same.

## Connect to Dashboard
```bash
$ kubectl proxy
```
In browser connect to http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

Output:
![screenshot](task1_screen1.png)

# Task 1.2
# Kubernetes resources introduction
```bash
$ kubectl run web --image=nginx:latest
```
Output:
```bash
pod/web created
```

- take a look at created resource in cmd "kubectl get pods"
```bash
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          77s
```
- take a look at created resource in Dashboard
![screenshot](task1_screen2.png)
- take a look at created resource in cmd
```bash
$ minikube ssh
$ docker container ls | grep web
```
Output:
```bash
c023694515b9   nginx                    "/docker-entrypoint.…"   8 minutes ago       Up 8 minutes                 k8s_web_web_default_58015b6b-d521-446b-bac0-628bea41a308_0
34b998e96a5c   k8s.gcr.io/pause:3.5     "/pause"                 8 minutes ago       Up 8 minutes                 k8s_POD_web_default_58015b6b-d521-446b-bac0-628bea41a308_0
```

## [Specification](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/)
```bash
$ kubectl explain pods.spec
```
Output:
```bash
KIND:     Pod
VERSION:  v1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior of the pod. More info:
... 
more output omitted
```

Apply manifests (download from repository)
```bash
$ kubectl apply -f pod.yaml
pod/nginx created
$ kubectl apply -f rs.yaml
replicaset.apps/webreplica created
```
Look at pod
```bash
$ kubectl get pod
NAME               READY   STATUS    RESTARTS   AGE
nginx              1/1     Running   0          55s
web                1/1     Running   0          17m
webreplica-6pzk9   1/1     Running   0          46s
```
# You can create simple manifest from cmd
```bash
$ kubectl run web --image=nginx:latest --dry-run=client -o yaml
```
Output:
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: web
  name: web
spec:
  containers:
  - image: nginx:latest
    name: web
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

### Homework
* Create a deployment nginx. Set up two replicas. Remove one of the pods, see what happens.
```bash
$ kubectl create deployment nginx --image=nginx --replicas=2
deployment.apps/nginx created

$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6799fc88d8-pssj4   1/1     Running   0          46s
nginx-6799fc88d8-xvw9r   1/1     Running   0          46s

$ kubectl delete pod nginx-6799fc88d8-pssj4 && kubectl get pods
pod "nginx-6799fc88d8-pssj4" deleted
NAME                     READY   STATUS              RESTARTS   AGE
nginx-6799fc88d8-sdm5f   0/1     ContainerCreating   0          0s
nginx-6799fc88d8-xvw9r   1/1     Running             0          99s

$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6799fc88d8-sdm5f   1/1     Running   0          3s
nginx-6799fc88d8-xvw9r   1/1     Running   0          102s
```
The new pod is created immediately.
