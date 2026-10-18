helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo update
helm install vmks vm/victoria-metrics-k8s-stack -n monitoring -f my-stack-values.yaml
