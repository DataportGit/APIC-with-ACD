# Troubleshooting

Get Pods with WorkerNode and PersistentVolumes

```
kubectl get pod -o custom-columns=\
NAME:metadata.name,\
HOST:status.hostIP,\
PV:spec.volumes[*].persistenVolumeClaim.claimName
```
