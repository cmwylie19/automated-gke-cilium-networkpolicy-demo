# Automated GKE Network Policy Demo
Before running the automation, make sure you have the correct variables in `env-automation/group_vars/all.yaml`. There are explanations in the `all.yaml` file and explanations regarding the GKE cluster for some variables in the `env-automation/README.md`

## Prerequisites
- Install [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
- Install kubernetes module: `ansible-galaxy collection install kubernetes.core`   
- Install [helm](https://helm.sh/docs/intro/install/)
- Install [kubectl](https://kubernetes.io/docs/tasks/tools/)
- Have a [GKE Project](https://cloud.google.com/kubernetes-engine/docs/quickstart)
- Modify the `env-automation/group_vars/all.yaml` file.

## Spin up GKE Cluster
This will spin up a GKE cluster with Cilium installed on the nodes.
```
ansible-playbook spin-up-env.yaml
```

## Tear down GKE Clusters
This will tear down the cluster.
```
ansible-playbook tear-down-env.yaml
```

## Network Policy Demo
Create two nginx pods, `n1` and `n2`.
```
kubectl run n1 --image=nginx

kubectl run n2 --image=nginx
```

Verify that `n1` can connect to `n2`
```
kubectl exec -it n1 -- curl --connect-timeout 3 $(kubectl get pod n2 -ojsonpath="{.status.podIP}") 
```

**output**
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Create a deny all network policy
```
kubectl apply -f -<<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  policyTypes: ["Ingress"]
  podSelector: {}
EOF
```

Verify that the network policy worked and that `n1` cannot connect to `n2`
```
kubectl exec -it n1 -- curl --connect-timeout 3 $(kubectl get pod n2 -ojsonpath="{.status.podIP}") 
```

**output**
```
curl: (28) Connection timed out after 3001 milliseconds
command terminated with exit code 28
```

Create a Network Policy to allow `n1` to talk to `n2`
```
kubectl apply -f -<<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: n2-policy
spec:
  policyTypes: ["Ingress"]
  podSelector:
    matchLabels:
      run: n2
  ingress:
    - from:
        - podSelector:
            matchLabels:
              run: n1
EOF
```

Verify that the network policy `n2-policy` worked and that `n1` can connect to `n2`
```
kubectl exec -it n1 -- curl --connect-timeout 3 $(kubectl get pod n2 -ojsonpath="{.status.podIP}") 
```

**output**
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Cleanup Network Policy Demo
```
kubectl delete pod n1 --force --grace-period=0

kubectl delete pod n2 --force --grace-period=0

kubectl delete netpol --all
```