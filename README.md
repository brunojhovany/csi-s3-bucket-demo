# CSI for S3 DEMO

This is a Container Storage Interface (CSI) for S3 (or S3 compatible) storage. This can dynamically allocate buckets and mount them via a fuse mount into any container.

## Status
This is still very experimental and should not be used in any production environment. Unexpected data loss could occur depending on what mounter and S3 storage backend is being used.

# Kubernetes installation
## Requirements
- Kubernetes 1.13+ (CSI v1.0.0 compatibility)
- Kubernetes has to allow privileged containers
- Docker daemon must allow shared mounts (systemd flag MountFlags=shared)


## If you are not allowed shared mounts
## Example using ubuntu:
### Ubuntu Configuration and Shared Mounts
- Verify that your Docker version is 1.10 or later. run: `$ docker -v`
- If the following command succeeds, shared mounts are enabled on your system and no action is needed.
    ```bash
    $ docker run -it -v /mnt:/mnt:shared busybox sh -c /bin/date
    ```
- If you are not allowed shared mounts, run: `$ sudo mount --make-shared /`
- If you are using systemd, remove the `MountFlags=slave` line in your docker.service file.
### you can refer to the following page for reference. -> [_OS Configurations for Shared Mounts_](https://legacy-docs.portworx.com/knowledgebase/shared-mount-propagation.html)


## 1. Create a secret with your S3 credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  namespace: kube-system
  name: csi-s3-secret
  # Namespace depends on the configuration in the storageclass.yaml
  namespace: kube-system
stringData:
  accessKeyID: <YOUR_ACCESS_KEY_ID>
  secretAccessKey: <YOUR_SECRET_ACCES_KEY>
  # For AWS set it to "https://s3.<region>.amazonaws.com"
  endpoint: <S3_ENDPOINT_URL>
  # region: <S3_REGION>
```
01-aws-credentials.yaml

## 2. Deploy the driver
```bash
kubectl create -f 02-provisioner.yaml
kubectl create -f 03-attacher.yaml
kubectl create -f 04-csi-s3.yaml
```

## Create the storage class
```bash
kubectl create -f 05-storageclass.yaml
```

### 4. Test the S3 driver

1. Create a pvc using the new storage class:

    ```bash
    kubectl create -f 06-pvc.yaml
    ```

1. Check if the PVC has been bound:

    ```bash
    $ kubectl get pvc csi-s3-pvc
    NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    csi-s3-pvc   Bound     pvc-c5d4634f-8507-11e8-9f33-0e243832354b   5Gi        RWO            csi-s3         9s
    ```

1. Create a test pod which mounts your volume:

    ```bash
    kubectl create -f 07-pod.yaml
    ```

    If the pod can start, everything should be working.

1. Test the mount

    ```bash
    $ kubectl exec -ti csi-s3-test-nginx bash
    $ mount | grep fuse
    s3fs on /var/lib/www/html type fuse.s3fs (rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other)
    $ touch /var/lib/www/html/hello_world
    ```


## _For more information about the csi provisioner consult the source repository_ [CSI for S3](https://github.com/ctrox/csi-s3)