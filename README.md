# upcloud-argocd
How to deploy Argo CD on UpCloud's Kubernete Service

This assumes you are using UpCloud's managed Kubernetes service and have it setup & running.

Argo CD can be found here: https://github.com/argoproj/argo-cd

Create the namespace:
```
kubectl create namespace argocd
```

Run the manifests. Make sure to change the version number below.:
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/vX.X.X-X/manifests/install.yaml
```

Test that you can access the argocd-server deployment:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

This will make Argo CD accessible on http://localhost:8080.

To expose the deployment using UpCloud’s Load Balancer (Paid!):
```
kubectl expose deployment -n argocd argocd-server \
--name=expose-argocd-server \
--port=443 \
--target-port=8080 \
--type=LoadBalancer
```

Optional: If you did the above you can skip this but you can also patch the Cluster IP that already exists instead of exposing a new LB service:
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Disable Argo CD TLS:

Argo runs on TLS by default, and UpCloud’s Load Balancer doesn't support TLS-enabled backends by default. This means that you will get an HTTP redirect loop; as you query the Load Balancer over HTTPS, the Load Balancer talks to the backend over HTTP, and because Argo forces TLS, it then redirects you to HTTPS with HTTP 3XX. The easy workaround is to make Argo not do that using server.insecure: true. (Optionally you can terminate TLS on the backend rather than on the load balancer)
```
kubectl patch configmap/argocd-cmd-params-cm -n argocd --type merge -p '{"data":{"server.insecure": "true"}}'
```

Redeploy the argo-server pod to propagate the patch:
```
kubectl delete pods -n argocd -l app.kubernetes.io/name=argocd-server
```
