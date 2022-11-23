---
author: "Viki Pranata"
title: "Kubernetes Monitoring dengan Prometheus Stack"
description : "Memonitoring Sumberdaya Kubernetes dengan kube-prometheus-stack"
date: "22-11-2022"
tags: ["linux", "Kubernetes", "helm", "Prometheus"]
showToc: true
---

### Membuat secret akses certificate etcd client
> run as root user
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl -n monitoring create secret generic etcd-client-cert --from-file=/etc/kubernetes/pki/etcd/ca.crt --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.crt --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

### Membuat helm values kube-prometheus-stack
buat file dan isi data sebagai berikut :
`sudo nano kube-prometheus-stack-helm-values.yaml`
```yaml
alertmanager:
  enabled: false

grafana:
  defaultDashboardsTimezone: Asia/Jakarta
  adminPassword: P@ssw0rd
  image:
    repository: grafana/grafana
    tag: "8.2.7"
  persistence:
    enabled: true
    storageClassName: "local-path"
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi

prometheus:
  prometheusSpec:
    secrets: ['etcd-client-cert']
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: "local-path"
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi
kubeEtcd:
  service:
    targetPort: 2379
  serviceMonitor:
    scheme: https
    insecureSkipVerify: false
    serverName: localhost
    caFile: /etc/prometheus/secrets/etcd-client-cert/ca.crt
    certFile: /etc/prometheus/secrets/etcd-client-cert/healthcheck-client.crt
    keyFile: /etc/prometheus/secrets/etcd-client-cert/healthcheck-client.key

kubeScheduler:
  service:
    targetPort: 10259
  serviceMonitor:
    https: "true"
    insecureSkipVerify: "true"
```
`storageClassName` didapat pada pemasangan local-path pada postingan [menambahkan local path provisioner](/posts/kubernetes-getting-started/#menambahkan-local-path-provisioner)

### Memasang kube-prometheus-stack dengan helm
```bash
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring -f kube-prometheus-stack-helm-values.yaml
```

### Membuat ingress kube-prometheus-stack
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: grafana.syslog.my.id
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: monitor-grafana
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: prometheus.syslog.my.id
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: monitor-kube-prometheus-st-prometheus
            port:
              number: 9090
```

### Memperbaik masalah yang terjadi
![image](/assets/images/kube-prometheus-stack-issue.jpg)
Update ip 127.0.0.1 ke ip 0.0.0.0.
```bash
sudo nano /etc/kubernetes/manifests/etcd.yaml
    - --listen-metrics-urls=http://127.0.0.1:2381
sudo nano /etc/kubernetes/manifests/kube-controller-manager.yaml
    - --bind-address=127.0.0.1
sudo nano /etc/kubernetes/manifests/kube-scheduler.yaml
    - --bind-address=127.0.0.1
```

### Sumber Referensi
Persistent volumes issue:    
- https://github.com/prometheus-community/helm-charts/issues/186#issuecomment-899669790

etcd monitoring issue:    
- https://github.com/prometheus-community/helm-charts/issues/1005#issuecomment-1014873446
- https://github.com/prometheus-community/helm-charts/issues/204#issuecomment-765155883

kube-prometheus-stack scraping metrics issue:    
- https://stackoverflow.com/questions/65901186/kube-prometheus-stack-issue-scraping-metrics
- https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy