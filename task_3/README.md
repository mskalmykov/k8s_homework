# Task 3
### [Read more about CSI](https://habr.com/ru/company/flant/blog/424211/)
Done.
### Create pv in kubernetes
```bash
$ kubectl apply -f pv.yaml
persistentvolume/minio-deployment-pv created
```
### Check our pv
```bash
$ kubectl get pv
NAME                  CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
minio-deployment-pv   5Gi        RWO            Retain           Available                                   42s
```
### Create pvc
```bash
$ kubectl apply -f pvc.yaml
persistentvolumeclaim/minio-deployment-claim created
```
### Check our output in pv 
```bash
$ kubectl get pv
NAME                  CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS   REASON   AGE
minio-deployment-pv   5Gi        RWO            Retain           Bound    default/minio-deployment-claim                           2m19s
```
Output has changed. PV got status bound.
### Check pvc
```bash
$ kubectl get pvc
NAME                     STATUS   VOLUME                CAPACITY   ACCESS MODES   STORAGECLASS   AGE
minio-deployment-claim   Bound    minio-deployment-pv   5Gi        RWO                           95s
```
### Apply deployment minio
```bash
$ kubectl apply -f deployment.yaml
deployment.apps/minio created
```
### Apply svc nodeport
```bash
$ kubectl apply -f minio-nodeport.yaml
service/minio-app created
```
Open minikup_ip:node_port in you browser:
![screenshot](task3_screen1.png)
### Apply statefulset
```bash
$ kubectl apply -f statefulset.yaml
statefulset.apps/minio-state created
service/minio-state created
```
### Check pod and statefulset
```bash
$ kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
minio-6db846bd9f-827s8   1/1     Running   0          13m
minio-state-0            1/1     Running   0          6s
$ kubectl get sts
NAME          READY   AGE
minio-state   1/1     2m59s
```

### Homework
* We published minio "outside" using nodePort. Do the same but using ingress.
* Publish minio via ingress so that minio by ip_minikube and nginx returning hostname (previous job) by path ip_minikube/web are available at the same time.
* Create deploy with emptyDir save data to mountPoint emptyDir, delete pods, check data.
* Optional. Raise an nfs share on a remote machine. Create a pv using this share, create a pvc for it, create a deployment. Save data to the share, delete the deployment, delete the pv/pvc, check that the data is safe.
