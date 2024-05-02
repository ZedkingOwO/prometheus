# Prometheus Monitoring System

- prometheus-server：部署Promethues Metrics API Server所需要的各资源配置清单。
- prometheus-adapter：部署基于prometheus的自定义指标API服务器所需要的各资源配置清单。
- podinfo：测试使用的podinfo相关的deployment和service对象的资源配置清单。
- node_exporter：于kubernetes集群各节点部署node_exporter。
- kube-state-metrics：聚合kubernetes资源对象，提供指标数据。
- alertmanager：部署AlertManager告警系统。
- grafana：部署Grafana Dashboard。

### 部署Prometheus

部署Prometheus监控系统

```bash
kubectl apply -f namespace.yaml
kubectl apply -f prometheus-server/ -n prom
```

部署node-exporter

```bash
kubectl apply -f node-exporter/
```

### 部署Kube-State-Metrics

部署kube-state-metrics，监控Kubernetes集群的服务指标。

```bash
kubectl apply -f kube-state-metrics/
```

### 部署AlertManager

部署AlertManager，为Prometheus-Server提供可用的告警发送服务。

```bash
kubectl apply -f alertmanager/
```

### 部署Prometheus Adpater

参考相关目录中的[README](prometheus-adpater/README.md)文件中的部署说明。

