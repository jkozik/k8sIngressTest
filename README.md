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

[jkozik@dell2 ingresstest]$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-cglhn        0/1     Completed   0          2d23h
ingress-nginx-admission-patch-6dzqj         0/1     Completed   1          2d23h
ingress-nginx-controller-55bc4f5576-wjsqt   1/1     Running     0          2d23h

```
It is useful to note that for any ingress traffic, the following command views the log files.  This was useful when debugging my setup
```
[jkozik@dell2 ingresstest]$ kubectl logs -n ingress-nginx ingress-nginx-controller-55bc4f5576-wjsqt
-------------------------------------------------------------------------------
NGINX Ingress controller
  Release:       v0.46.0
  Build:         6348dde672588d5495f70ec77257c230dc8da134
  Repository:    https://github.com/kubernetes/ingress-nginx
  nginx version: nginx/1.19.6

-------------------------------------------------------------------------------
192.168.100.174 - - [27/Jun/2021:16:57:04 +0000] "GET /wordpress HTTP/1.1" 200 4736 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.77 Safari/537.36" 909 0.730 [default-wordpress-80] [] 10.68.41.141:80 4736 0.734 200 97bc24777ef031eb3033381bed59350a
192.168.100.174 - - [27/Jun/2021:16:59:39 +0000] "GET /nginx HTTP/1.1" 200 25 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.77 Safari/537.36" 905 0.009 [default-nginx-svc-80] [] 10.68.41.140:80 25 0.008 200 ebb6f64ffa561468467177ddb98ba2d4
```
Next, apply the following to create an ingress resource for wordpress and nginx-svc.  I created a temporary subdomain called k8s.kozk.net.  The ingress resource below, looks at the path in the URL and redirects to the appropriate service.  
- k8s.kozik.net/wordpress redirects to service wordpress
- k8s.kozik.net/nginx redirects to service nginx-svc
```
cat <<EOF >>nginx-wordpress-ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-wordpress-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: k8s.kozik.net
    http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
      - path: /wordpress
        pathType: Prefix
        backend:
          service:
            name: wordpress
            port:
              number: 80
EOF

[jkozik@dell2 ingresstest]$ kubectl apply -f nginx-wordpress-ingress.yml
ingress.networking.k8s.io/nginx-wordpress-ingress created

[jkozik@dell2 ingresstest]$ kubectl get ingress -o wide
NAME                      CLASS    HOSTS           ADDRESS           PORTS   AGE
nginx-wordpress-ingress   <none>   k8s.kozik.net   192.168.100.174   80      96s

[jkozik@dell2 ingresstest]$ kubectl get services -o wide
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE     SELECTOR
kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP        25d     <none>
nginx-svc         ClusterIP   10.99.145.80     <none>        80/TCP         5d1h    app=nginx-app
wordpress         NodePort    10.105.26.145    <none>        80:30622/TCP   3d22h   app=wordpress,tier=frontend
wordpress-mysql   ClusterIP   None             <none>        3306/TCP       3d23h   app=wordpress,tier=mysql

[jkozik@dell2 ingresstest]$ kubectl get service -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.97.70.29     <none>        80:30140/TCP,443:30023/TCP   3d
ingress-nginx-controller-admission   ClusterIP   10.111.250.10   <none>        443/TCP                      3d

```
The ingress is IP address is any IP address of the cluster.  But the ingress controller service's NodePort address is 30140.  A quick test on the kubectl host can verify the setup.
```
[jkozik@dell2 ingresstest]$ curl -H "Host: k8s.kozik.net" 192.168.100.173:30140/nginx
Hello, NFS Storage NGINX

[jkozik@dell2 ingresstest]$ curl -H "Host: k8s.kozik.net" 192.168.100.173:30140/wordpress
<!DOCTYPE html>
<html lang="en-US" class="no-js">
<head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <link rel="profile" href="http://gmpg.org/xfn/11">
                <script>(function(html){html.className = html.className.replace(/\bno-js\b/,'js')})(document.documentElement);</script>
<title>WP Test &#8211; Just another WordPress site</title>
...
```
Note:  the curl commands, above, need the Host header filled-in, the ingress controller will return a 404 error if there is not a match to the host name. At this point, I updated my kozik.net DNS entry to include the k8s subdomain.  I assigned it the public IP address of my home network; I setup port forwarding in my home router and the outside world can (temporarily) access http://k8s.kozik.net/wordpress and http://k8s.kozik.net/nginx. 
The ingress controller is basically an instance of nginx configured to run as a reverse proxy.  Kubernetes has a library of ingress controllers based on familiar projects, eg HAProxy, Apache, F5 BIG-IP, Envoy/Istio, etc.  The nginx one that I use is the default one. 
