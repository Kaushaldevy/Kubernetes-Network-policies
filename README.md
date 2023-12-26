The following is a summary of few Network policies that we could use within Kubernetes.
Most of these polices are obtained from https://github.com/ahmetb/kubernetes-network-policy-recipes locations. But, some of them are customized and new ones are added.
#	Policy description	Manifest	Description
1	DENY all traffic to an application	
~~~
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-deny-all
spec:
  podSelector:
    matchLabels:
      app: web

      
ingress: []
~~~
ingress: [] -> This blocks all traffic to the app=web.

2	LIMIT traffic to an application	"kind: NetworkPolicy
```
apiVersion: networking.k8s.io/v1
metadata:
  name: api-allow
spec:
  podSelector:
    matchLabels:
      app: bookstore
      role: api
  ingress:
  - from:
      - podSelector:
          matchLabels:
            app: bookstore"	Allow only from labels having app: bookstore
```

3	ALLOW all traffic to an application	"kind: NetworkPolicy
~~~
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-all
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - {}	
~~~
Allow all traffic within default namespace

4	DENY all non-whitelisted traffic to a namespace	"kind: NetworkPolicy
~~~
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  ingress: []
~~~
This is a fundamental policy, blocking all cross-pod networking other than the ones whitelisted via the other Network Policies you deploy.
namespace: default deploy this policy to the default namespace.
podSelector: is empty, this means it will match all the pods. Therefore, the policy will be enforced to ALL pods in the default namespace .
There are no ingress rules specified. This causes incoming traffic to be dropped to the selected (=all) pods.
In this case, you can just omit the ingress field, or leave it empty like ingress:

5	DENY all traffic from other namespaces	"kind: NetworkPolicy
~~~
apiVersion: networking.k8s.io/v1
metadata:
  namespace: default
  name: deny-from-other-namespaces
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - podSelector: {}
~~~

6	ALLOW traffic to an application from all namespaces	"kind: NetworkPolicy
~~~
apiVersion: networking.k8s.io/v1
metadata:
  namespace: default
  name: web-allow-all-namespaces
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from:
    - namespaceSelector: {}
~~~

7	ALLOW all traffic from a namespace	"kind: NetworkPolicy
~~~
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-prod
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: production
~~~
      
8	ALLOW traffic from some pods in another namespace	"kind: NetworkPolicy
~~~
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-all-ns-monitoring
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
    - from:
      - namespaceSelector:     # chooses all pods in namespaces labelled with team=operations
          matchLabels:
            team: operations  
        podSelector:           # chooses pods with type=monitoring
          matchLabels:
            type: monitoring
~~~
The following manifest restricts traffic to only pods with label type=monitoring in namespaces labelled team=operations

9	ALLOW traffic from external clients	"kind: NetworkPolicy
~~~
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-external
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - {}
~~~

Run a web server and expose it to the internet with a Load Balancer:

```
kubectl run web --image=nginx --labels=""app=web"" --port=80

kubectl expose pod/web --type=LoadBalancer
```
Wait until an EXTERNAL-IP appears on kubectl get service output. Visit the http://[EXTERNAL-IP] on your browser and verify it is accessible.

The following manifest allows traffic from all sources (both internal from the cluster and external). 
To limit this to port 80 only, can change rule as.

```
 ingress:
  - ports:
    - port: 80"
```

10	ALLOW traffic only to a port of an application	"kind: NetworkPolicy
```apiVersion: networking.k8s.io/v1
metadata:
  name: api-allow-5000
spec:
  podSelector:
    matchLabels:
      app: apiserver
  ingress:
  - ports:
    - port: 5000
    from:
    - podSelector:
        matchLabels:
          role: monitoring
```
NOTE: Network Policies will not know the port numbers you exposed the application, such as 8001 and 5001. This is because they control inter-pod traffic and when you expose Pod as Service, ports are remapped like above. Therefore, you need to use the Pod port numbers (such as 8000 and 5000) in the NetworkPolicy specification. An alternative less error prone is to refer to the port names (such as metrics and http).

11	ALLOW traffic from apps using multiple selectors	"kind: NetworkPolicy
```
apiVersion: networking.k8s.io/v1
metadata:
  name: redis-allow-services
spec:
  podSelector:
    matchLabels:
      app: bookstore
      role: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: bookstore
          role: search
    - podSelector:
        matchLabels:
          app: bookstore
          role: api
    - podSelector:
        matchLabels:
          app: inventory
          role: web

```
service|   labels
search       app=bookstore, role=search
api              app=bookstore, role=api
catalog 	    app=inventory, role=web

The above  NetworkPolicy will allow traffic from only these microservices.

12	DENY egress traffic from an application	"apiVersion: networking.k8s.io/v1
```
kind: NetworkPolicy
metadata:
  name: foo-deny-egress
spec:
  podSelector:
    matchLabels:
      app: foo
  policyTypes:
  - Egress
  egress: []
```
You can drop this field (egress) altogether and have the same effect.

13	DENY all non-whitelisted traffic from a namespace	"kind: NetworkPolicy
```
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny-all-egress
  namespace: default
spec:
  policyTypes:
  - Egress
  podSelector: {}
  egress: []
```
Use Case: This is a fundamental policy, blocking all outgoing (egress) traffic from a namespace by default (including DNS resolution). 
After deploying this, you can deploy Network Policies that allow the specific outgoing traffic.

14	DENY external egress traffic	"apiVersion: networking.k8s.io/v1
```
kind: NetworkPolicy
metadata:
  name: foo-deny-external-egress
spec:
  podSelector:
    matchLabels:
      app: foo
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
      - port: 53
        protocol: UDP
      - port: 53
        protocol: TCP
```
"This policy applies to pods with app=foo and in Egress (outbound) direction.
Similar to DENY egress traffic from an application example, this policy allows all outbound traffic on ports 53/udp and 53/tcp to the kube-dns pods for DNS resolution.
to: specifies a namespaceSelector which matches kubernetes.io/metadata.
name: kube-system and a podSelector which matches k8s-app: kube-dns. 
This will select only the kube-dns pods in the kube-system namespace, so the outbound traffic to the kube-dns pods in the kube-system namespace will be allowed.
And since they are not listed, traffic to the IP addresses outside the cluster are denied."
![image](https://github.com/Kaushaldevy/Kubernetes-Network-policies/assets/18143604/256966f8-21cf-437a-8590-7c121cb253b8)
