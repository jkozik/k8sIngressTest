# k8sIngressTest
I created a Kubernetes cluster, I tested it by creating a basic nginx deployment, and then deployed a Wordpress instance using NFS mounted storage (using PV / PVC) and secrets.  All of this was exposed to my home LAN using a NodePort-type service.
See these links
- https://github.com/jkozik/SetupKubeadmCentos7
- https://github.com/jkozik/SetupKubeadmCentos7/blob/main/ClusterBasics.md
- https://github.com/jkozik/k8sWordpressTest

Here are my deployments and services:
```
[jkozik@dell2 ingresstest]$  kubectl get deployments,services -o wide
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS            IMAGES                                         SELECTOR
deployment.apps/nginx-deployment      1/1     1            1           4d21h   nginx                 nginx:1.13.12                                  app=nginx-app
deployment.apps/wordpress             1/1     1            1           3d19h   wordpress             wordpress:4.8-apache                           app=wordpress,tier=frontend
deployment.apps/wordpress-mysql       1/1     1            1           3d19h   mysql                 mysql:5.6                                      app=wordpress,tier=mysql

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE     SELECTOR
service/kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP        25d     <none>
service/nginx-svc         ClusterIP   10.99.145.80     <none>        80/TCP         4d22h   app=nginx-app
service/wordpress         NodePort    10.105.26.145    <none>        80:30622/TCP   3d19h   app=wordpress,tier=frontend
service/wordpress-mysql   ClusterIP   None             <none>        3306/TCP       3d19h   app=wordpress,tier=mysql
[jkozik@dell2 ingresstest]$
```
The exposed service, wordpress, is accessible from my home LAN on port 30622. If I want I can using port forwarding on my home router to expose this service to the Internet.  If I want, I could also configure the nginx-svc with NodePort and expose it to the internet. Etc.
Each exposed service would need it's own port number.  Perhaps wordpress could use port 80, then nginx-svc could use port 8080 -- I don't want to expose a service to the Internet requiring a port number!  I want a unique URL for each and have that URL matched to port number hidden from the user.  Yes, I need a reverse proxy function much like one can find in Apache, nginx, HAproxy.

Thus we need another exposure layer.  Kubernetes defines an Ingress resource to do this. This notes shows how I setup an Ingress Controller and create a couple of ingress instances for my two example services.

## Install Ingress Controller

From my kubectl host that is running outside of my cluster (non root login):
```
[jkozik@dell2 ~]$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/baremetal/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
deployment.apps/ingress-nginx-controller created 

[jkozik@dell2 ~]$ kubectl get pods -n ingress-nginx   -l app.kubernetes.io/name=ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-cglhn        0/1     Completed   0          11m
ingress-nginx-admission-patch-6dzqj         0/1     Completed   1          11m
ingress-nginx-controller-55bc4f5576-wjsqt   1/1     Running     0          11m

[jkozik@dell2 ingresstest]$ kubectl get service -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.97.70.29     <none>        80:30140/TCP,443:30023/TCP   2d21h
ingress-nginx-controller-admission   ClusterIP   10.111.250.10   <none>        443/TCP                      2d21h
```
