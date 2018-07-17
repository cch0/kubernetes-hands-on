## Install ChartMuseum with Helm Chart

## Helm chart

Helm chart provided at Kubernetes github repository [ChartMuseum Helm Chart](https://github.com/kubernetes/charts/tree/master/stable/chartmuseum) can be used to install ChartMuseum.

## Configuration

* Default values defined in __values.yaml__ file have been overridden to provide persistence in GCP, access to Cloud Storage bucket, etc.

    * __env.open.STORAGE__ is set to __google__
    * __env.open.DISABLE_API__ is set to __false__
    * __env.open.ALLOW_OVERWRITE__ is set to __true__
    * __service.type__ is set to __LoadBalancer__
    * __service.externalPort__ is set to __80__
    * __persistence.enabled__ is set to __true__
    * __persistence.size__ is set to __200Gi__
    * __persistence.storageClass__ is set to __regional-sc__ 

* Additionally

    * __namespace__ variable has been added.
    * resources can be created into non-default namespace 
    * RBAC is being used so that a service account named __chartmuseum__ with permission only in a namespace 

## Install ChartMuseum

```
helm install --name {{release-name}}  --set env.open.STORAGE_GOOGLE_BUCKET={{cloud-storage-bucket-name}  .
```

## Post Install Configuration

* Install __helm push plugin__

```
helm plugin install https://github.com/chartmuseum/helm-push
```

## Use ChartMuseum

* Add created helm repo 

```
helm repo add {{helm-repo-name}} http://{{service-ip}
```

* Install your chart to ChartMuseum

```
helm push /path/to/your/chart {{helm-repo-name}} --version={{version-number}}
```

## Be Aware!!!

*  The PVC resource is expecting a StorageClass nased __regional-sc__ exists already.

   ```
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: regional-sc
   provisioner: kubernetes.io/gce-pd
   reclaimPolicy: Retain
   parameters:
     type: pd-standard
     zones: us-west1-a, us-west1-b
     replication-type: regional-pd
     reclaimPolicy: Retain
   ```

