# k8s-hpa-demo

### How to scale k8s deployments depending on custom prometheus metric using HPA

---
## setup kube-prometheus-stack
https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# deploy kube-prometheus-stack
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 45.28.0 -f kube-prometheus-stack/values.yaml

# access grafana
kubectl port-forward service/kube-prometheus-stack-grafana 8080:80

```

---
## setup app with metrics (bitnami nginx as example)
https://github.com/bitnami/charts/blob/main/bitnami/nginx/values.yaml

```sh
# deploy app
helm upgrade --install test-app oci://registry-1.docker.io/bitnamicharts/nginx --version 14.2.1 -f app/values.yaml
```
App will create public loadbalancer
```sh
# access app by IP
kubectl get svc --namespace default test-app-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
Generate some load and check metric `sum(rate(nginx_http_requests_total[1m]))` at grafana

---
## setup prometheus-adapter to extend k8s metrics API with prometheus metrics
https://github.com/kubernetes-sigs/prometheus-adapter

https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-adapter/values.yaml

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# deploy prometheus-adapter
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter  --version 4.2.0 -f prometheus-adapter/values.yaml

# check k8s API for my_custom_metric_nginx_http_requests_total
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq .

# check for current metric value
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/my_custom_metric_nginx_http_requests_total" | jq .
```

---
## modify app HPA to be based on custom metric

```sh
# backup current hpa config
kubectl get hpa test-app-nginx -o yaml > app/hpa-helm.yaml
# delete current hpa
kubectl delete hpa test-app-nginx
# apply modified hpa
kubectl apply -f app/hpa.yaml
```

---
## Generate some load and check pod scaling

```sh
# generate load
plow http://LB_IP_ADDRESS/ --rate 50/s -c 10


# watch scaling status
kubectl get hpa test-app-nginx  --watch
```

---
## Clean up

```sh
helm uninstall test-app
helm uninstall prometheus-adapter
helm uninstall kube-prometheus-stack
```
