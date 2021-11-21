# Task 4
### Check what I can do
```bash
$ kubectl auth can-i create deployments --namespace kube-system
yes
```
### Configure user authentication using x509 certificates
### Create private key
```bash
$ openssl genrsa -out k8s_user.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
............+++++
...........+++++
e is 65537 (0x010001)
```
### Create a certificate signing request
```bash
$ openssl req -new -key k8s_user.key \
-out k8s_user.csr \
-subj "/CN=k8s_user"
```
Done.

### Sign the CSR in the Kubernetes CA. We have to use the CA certificate and the key, which are usually in /etc/kubernetes/pki. But since we use minikube, the certificates will be on the host machine in ~/.minikube
```bash
$ openssl x509 -req -in k8s_user.csr \
-CA ~/.minikube/ca.crt \
-CAkey ~/.minikube/ca.key \
-CAcreateserial \
-out k8s_user.crt -days 500

Signature ok
subject=CN = k8s_user
Getting CA Private Key
```
### Set certificate as a credential for the user
```bash
$ kubectl config set-credentials k8s_user \
--client-certificate=k8s_user.crt \
--client-key=k8s_user.key

User "k8s_user" set.
```
### Set context for user
```bash
$ kubectl config set-context k8s_user \
--cluster=minikube --user=k8s_user

Context "k8s_user" created.
```
### Edit ~/.kube/config
```bash
users:
- name: k8s_user
  user:
    client-certificate: /home/msk17/education/task_4/k8s_user.crt
    client-key: /home/msk17/education/task_4/k8s_user.key
contexts:
- context:
    cluster: minikube
    user: k8s_user
  name: k8s_user
```
File appears already changed by previous commands.

### Switch to use new context
```bash
$ kubectl config use-context k8s_user
Switched to context "k8s_user".
```
### Check privileges
```bash
$ kubectl get node
Error from server (Forbidden): nodes is forbidden: User "k8s_user" cannot list resource "nodes" in API group "" at the cluster scope

$ kubectl get pod
Error from server (Forbidden): pods is forbidden: User "k8s_user" cannot list resource "pods" in API group "" in the namespace "default"
```
### Switch to default(admin) context
```bash
$ kubectl config use-context minikube
Switched to context "minikube".
```
### Bind role and clusterrole to the user
```bash
$ kubectl apply -f binding.yaml
rolebinding.rbac.authorization.k8s.io/k8s_user created
```
### Switch context and check pods again
```bash
$ kubectl config use-context k8s_user
Switched to context "k8s_user".

$ kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5f79fbdf75-5hdjq   1/1     Running   0          44s
nginx-5f79fbdf75-lscr6   1/1     Running   0          40s
nginx-5f79fbdf75-vvgds   1/1     Running   0          42s
```
Now we can see pods

### Homework
* Create users deploy_view and deploy_edit. Give the user deploy_view rights only to view deployments, pods. Give the user deploy_edit full rights to the objects deployments, pods.

Create certification requests:
```bash
$ for arg in deploy_view deploy_edit ; do openssl req -new -newkey rsa:2048 -keyout $arg.key -out $arg.csr -subj "/CN=$arg" -passout pass:"q23N8pdg" ; done
Generating a RSA private key
....+++++
...............................................+++++
writing new private key to 'deploy_view.key'
-----
Generating a RSA private key
.....................+++++
..............+++++
writing new private key to 'deploy_edit.key'
-----
```
Issue the certificates signed with CA signature:
```bash
$ for arg in deploy_view deploy_edit ; do \
      openssl x509 -req -in $arg.csr -CA ~/.kube/ca.crt \
         -CAkey ~/.kube/ca.key -CAcreateserial -out $arg.crt -days 500 ; done
Signature ok
subject=CN = deploy_view
Getting CA Private Key
Signature ok
subject=CN = deploy_edit
Getting CA Private Key
```
Create roles for users:
```bash
$ kubectl create role r_dep_view --verb=get,list,watch \
      --resource=pods,deployments
role.rbac.authorization.k8s.io/r_dep_view created

$ kubectl create role r_dep_edit \
      --verb=* --resource=pods,deployments
role.rbac.authorization.k8s.io/r_dep_edit created
```
Bind roles to users:
```bash
$ kubectl create rolebinding rb_dep_view --role=r_dep_view --user=deploy_view
rolebinding.rbac.authorization.k8s.io/rb_dep_view created

$ kubectl create rolebinding rb_dep_edit --role=r_dep_edit --user=deploy_edit
rolebinding.rbac.authorization.k8s.io/rb_dep_edit created
```
Add user config to ~/.kube/config
```bash
$ for arg in deploy_view deploy_edit ; do \
      kubectl config set-credentials $arg \
      --client-certificate=$arg.crt --client-key=$arg.key ; done
User "deploy_view" set.
User "deploy_edit" set.
```
Create contexts for users:
```bash
$ for arg in deploy_view deploy_edit ; do kubectl config set-context $arg --cluster=minikube --user=$arg  ; done
Context "deploy_view" created.
Context "deploy_edit" created.
```
Switch to deploy_view context and check access:
```bash
$ kubectl config use-context deploy_view
Switched to context "deploy_view".

$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5f79fbdf75-5hdjq   1/1     Running   0          2d4h
nginx-5f79fbdf75-lscr6   1/1     Running   0          2d4h
nginx-5f79fbdf75-vvgds   1/1     Running   0          2d4h

$ kubectl delete pod nginx-5f79fbdf75-vvgds
Error from server (Forbidden): pods "nginx-5f79fbdf75-vvgds" is forbidden: User "deploy_view" cannot delete resource "pods" in API group "" in the namespace "default"

$ kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           2d4h

$ kubectl delete deployment nginx
Error from server (Forbidden): deployments.apps "nginx" is forbidden: User "deploy_view" cannot delete resource "deployments" in API group "apps" in the namespace "default"

$ kubectl get services
Error from server (Forbidden): services is forbidden: User "deploy_view" cannot list resource "services" in API group "" in the namespace "default"
m
```
As you can see, user deploy_view can only view pods and deployments, but cannot change them.

Switch to deploy_edit context and check privileges:
```bash
$ kubectl config use-context deploy_edit
Switched to context "deploy_edit".

$ kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           2d4h

$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5f79fbdf75-5hdjq   1/1     Running   0          2d4h
nginx-5f79fbdf75-lscr6   1/1     Running   0          2d4h
nginx-5f79fbdf75-vvgds   1/1     Running   0          2d4h

$ kubectl delete pod nginx-5f79fbdf75-vvgds
pod "nginx-5f79fbdf75-vvgds" deleted

$ kubectl get configmaps
Error from server (Forbidden): configmaps is forbidden: User "deploy_edit" cannot list resource "configmaps" in API group "" in the namespace "default"
```
User can view and edit pods and deployments, but no more.

* Create namespace prod. Create users prod_admin, prod_view. Give the user prod_admin admin rights on ns prod, give the user prod_view only view rights on namespace prod.

Create keys and certificate requests for users:
```bash
$ for user in prod_admin prod_view ; do openssl genrsa -out $user.key 2048 ; done
Generating RSA private key, 2048 bit long modulus (2 primes)
.........................................+++++
......................+++++
e is 65537 (0x010001)
Generating RSA private key, 2048 bit long modulus (2 primes)
..............................................................................................................................+++++
....................+++++
e is 65537 (0x010001)

$ for user in prod_admin prod_view ; do openssl req -new -key $user.key -out $user.csr -subj "/CN=$user" ; done
```
Issue the certificates signed with CA signature:
```bash
$ for user in prod_admin prod_view ; do \
>     openssl x509 -req -in $user.csr \
>       -CA ~/.minikube/ca.crt -CAkey ~/.minikube/ca.key \
>       -CAcreateserial -days 500 -out $user.crt ;
> done
Signature ok
subject=CN = prod_admin
Getting CA Private Key
Signature ok
subject=CN = prod_view
Getting CA Private Key
```
Create namespace and role bindings:
```bash
$ kubectl create ns prod
namespace/prod created

$ kubectl -n prod create rolebinding rb_prod_view \
    --clusterrole=view --user=prod_view
rolebinding.rbac.authorization.k8s.io/rb_prod_view created

$ kubectl -n prod create rolebinding rb_prod_admin \
    --clusterrole=admin --user=prod_admin
rolebinding.rbac.authorization.k8s.io/rb_prod_admin created
```
Establish credentials and contexts for users:
```bash
$ for user in prod_admin prod_view ; do \
>     kubectl config set-credentials $user \
>       --client-certificate=$user.crt \
>       --client-key=$user.key ; \
> done
User "prod_admin" set.
User "prod_view" set.

$ for user in prod_admin prod_view ; do \
>     kubectl config set-context $user \
>       --cluster=minikube --user=$user ; \
> done
Context "prod_admin" created.
Context "prod_view" created.
```
Switch to prod_admin and check access:
```bash
$ kubectl config use-context prod_admin
Switched to context "prod_admin".

$ kubectl get pods
Error from server (Forbidden): pods is forbidden: User "prod_admin" cannot list resource "pods" in API group "" in the namespace "default": RBAC: role.rbac.authorization.k8s.io "r_prod_admin" not found

$ kubectl -n prod get pods
No resources found in prod namespace.

$ kubectl -n prod create deployment test --image=nginx --replicas=3
deployment.apps/test created

$ kubectl -n prod get pods
NAME                    READY   STATUS        RESTARTS   AGE
test-588bbdd875-hdrrc   1/1     Running       0          5s
test-588bbdd875-j7dfw   1/1     Running       0          9s
test-588bbdd875-stzdh   1/1     Running       0          7s
```
User can create, change and delete resources in the namespace prod.

Switch to prod_view and check access:
```bash
$ kubectl config use-context prod_view
Switched to context "prod_view".

$ kubectl -n prod get pods
NAME                    READY   STATUS    RESTARTS   AGE
test-588bbdd875-hdrrc   1/1     Running   0          2m58s
test-588bbdd875-j7dfw   1/1     Running   0          3m2s
test-588bbdd875-stzdh   1/1     Running   0          3m

$ kubectl -n prod delete pod test-588bbdd875-j7dfw
Error from server (Forbidden): pods "test-588bbdd875-j7dfw" is forbidden: User "prod_view" cannot delete resource "pods" in API group "" in the namespace "prod"
```
User have read-only access to the namespace prod resources.

* Create a serviceAccount sa-namespace-admin. Grant full rights to namespace default. Create context, authorize using the created sa, check accesses.
```bash
$ kubectl create sa sa-namespace-admin
serviceaccount/sa-namespace-admin created

$ kubectl create rolebinding rb_ns_admin \
    --clusterrole=admin \
    --serviceaccount=default:sa-namespace-admin
rolebinding.rbac.authorization.k8s.io/rb_ns_admin created

$ kubectl get sa sa-namespace-admin -o jsonpath={.secrets[0].name}
sa-namespace-admin-token-m766t

$ export TOKEN=`kubectl get secrets sa-namespace-admin-token-m766t -o jsonpath={.data.token} | base64 -d`

$ kubectl config set-credentials sa-namespace-admin --token=$TOKEN
User "sa-namespace-admin" set.

$ kubectl config set-context sa-namespace-admin --user=sa-namespace-admin --cluster=minikube
Context "sa-namespace-admin" created.

$ kubectl config use-context sa-namespace-admin
Switched to context "sa-namespace-admin".

$ kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5f79fbdf75-5cdqx   1/1     Running   0          110m
nginx-5f79fbdf75-5hdjq   1/1     Running   0          2d6h
nginx-5f79fbdf75-lscr6   1/1     Running   0          2d6h

$ kubectl delete pod nginx-5f79fbdf75-5hdjq
pod "nginx-5f79fbdf75-5hdjq" deleted
```
