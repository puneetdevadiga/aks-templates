# Dynamic NFS Azure File Share on AKS (Premium Azure Files via CSI)

This guide shows how to dynamically provision an NFS (protocol=nfs) Azure File share using the Azure File CSI driver and mount it to a StatefulSet (nginx container) on an AKS cluster.

## Prerequisites
- AKS cluster with Azure File CSI driver (default on recent AKS versions).
- Premium Azure Storage Account with NFS protocol enabled (minimum file share size 100 GiB).
- `kubectl` configured (kubeconfig authenticated; if using devicecode ensure `kubelogin` installed).

## 1. Create the StorageClass (NFS)
Create a file `nfs-sc.yaml` with the following manifest:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
	name: azurefile-csi-nfs
provisioner: file.csi.azure.com
allowVolumeExpansion: true
parameters:
	protocol: nfs
mountOptions:
	- nconnect=4
	- noresvport
	- actimeo=30
# NOTE: vers, minorversion, sec are managed internally by Azure File CSI driver; do not set them here.
```

Apply it:
```bash
kubectl apply -f nfs-sc.yaml
```
Expected output:
```
storageclass.storage.k8s.io/azurefile-csi-nfs created
```

## 2. StatefulSet Using the NFS-backed PVC
Create a file `statefulset-nfs.yaml`:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
	name: statefulset-azurefile
	labels:
		app: nginx
spec:
	podManagementPolicy: Parallel
	serviceName: statefulset-azurefile
	replicas: 1
	selector:
		matchLabels:
			app: nginx
	template:
		metadata:
			labels:
				app: nginx
		spec:
			nodeSelector:
				kubernetes.io/os: linux
			containers:
				- name: statefulset-azurefile
					image: mcr.microsoft.com/oss/nginx/nginx:1.19.5
					command:
						- /bin/bash
						- -c
						- set -euo pipefail; while true; do echo "$(date)" >> /mnt/azurefile/outfile; sleep 1; done
					volumeMounts:
						- name: persistent-storage
							mountPath: /mnt/azurefile
	updateStrategy:
		type: RollingUpdate
	volumeClaimTemplates:
		- metadata:
				name: persistent-storage
			spec:
				storageClassName: azurefile-csi-nfs
				accessModes: ["ReadWriteMany"]
				resources:
					requests:
						storage: 100Gi
```

Apply it:
```bash
kubectl apply -f statefulset-nfs.yaml
```
Expected output:
```
statefulset.apps/statefulset-azurefile created
```

## 3. Validate Mount & Capacity
Check the pod and filesystem:
```bash
kubectl get pods -l app=nginx
kubectl exec -it statefulset-azurefile-0 -- df -h
```

### Sample Output (Proof)
```
Filesystem                                                                                                        Size  Used Avail Use% Mounted on
overlay                                                                                                           291G   13G  279G   5% /
tmpfs                                                                                                              64M     0   64M   0% /dev
fc36479de7c5445ffa097dc.file.core.windows.net:/fc36479de7c5445ffa097dc/pvcn-75ad135c-8a7a-44f8-823c-5221f18adfed  100G     0  100G   0% /mnt/azurefile
/dev/root                                                                                                         291G   13G  279G   5% /etc/hosts
shm                                                                                                                64M     0   64M   0% /dev/shm
tmpfs                                                                                                              30G   12K   30G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                                                                                              16G     0   16G   0% /proc/acpi
tmpfs                                                                                                              16G     0   16G   0% /proc/scsi
tmpfs                                                                                                              16G     0   16G   0% /sys/firmware
```

Observe the NFS path format: `<storageAccount>.file.core.windows.net:/<storageAccount>/<generatedShareOrPVC>` mounted at `/mnt/azurefile`.

## 4. Inspect File Writes
```bash
kubectl exec -it statefulset-azurefile-0 -- tail -f /mnt/azurefile/outfile
```
You should see timestamps appended every second.


## 7. Cleanup
```bash
kubectl delete statefulset statefulset-azurefile
kubectl delete storageclass azurefile-csi-nfs
```
Note: Deleting the StatefulSet removes the PVCs (depending on reclaim policy). Ensure data retention strategy if needed.

## 8. Notes
- Do **not** specify `vers`, `minorversion`, or `sec` in `mountOptions`; they are managed by CSI.
- Premium NFS minimum share size is 100 GiB, so requesting smaller sizes will auto-expand or fail.
- For production, consider tagging resources and monitoring latency (Azure Monitor + metrics for storage account).

---
End of Dynamic NFS README.
