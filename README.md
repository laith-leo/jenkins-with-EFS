# Deploying Jenkins on Amazon EKS with Amazon EFS

This is about creating an Amazon EFS file system with mount targets in the DEV AWS AZs:

1. Capture the VPC ID: 
`aws ec2 describe-vpcs`

2. Create a security group for your Amazon EFS mount target: 
```
aws ec2 create-security-group \
--region us-east-1 \
--group-name efs-mount-sg \
--description "Amazon EFS for EKS, SG for mount target" \
--vpc-id identifier for our VPC (i.e. vpc-009bece7cdbdad52e)  
```

3. Add rules to the security group to authorize inbound/outbound access:
```
aws ec2 authorize-security-group-ingress  --group-id sg-05ce2021cb2a3713d --region us-west-2 --protocol tcp \                 
--port 2049 --cidr 172.16.162.0/23
```
4. Create an Amazon EFS file system:
```
aws efs create-access-point --file-system-id fs-2776ec22 --posix-user Uid=1000,Gid=1000 --root-directory "Path=/jenkins,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=777}"
```

5. Capture your VPC subnet IDs
```
aws ec2 describe-instances --region us-west-2 --filters Name=vpc-id,Values=vpc-009bece7cdbdad52e --query 'Reservations[*].Instances[].SubnetId'
```
6. Create Amazon EFS mount targets (*for each AZ*):
```
aws efs create-mount-target --file-system-id fs-2776ec22 --security-group sg-05ce2021cb2a3713d --region us-west-2 --subnet-id subnet-02484ecbacd53cfa6
```

7. Create an Amazon EFS access point:
```
aws efs create-access-point --file-system-id fs-2776ec22 --posix-user Uid=1000,Gid=1000 --root-directory "Path=/jenkins,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=777}"
```
8. Deploy the Amazon EFS CSI driver to your Amazon EKS cluster:
```
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```
9. Create efs-sc storage class YAML file:
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
```

10. Create the efs-pv persistent volume YAML file:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle:  fs-2776ec22::fsap-015ef0e99664dbe25
```

11. Create the efs-claim persistent volume claim YAML file to be used with Jenkins:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```
12. Deploy the efs-sc storage class, efs-pv persistent volume, and efs-claim persistent volume claim:

```
kubectl apply -f storageclass.yaml,persistentvolume.yaml,persistentvolumeclaim.yaml
```

13. Now we can install Jenkins with Helm referencing the existing PVC from :
```
helm install jenkins-uses-efs --set persistence.existingClaim=jenkins-efs-claim stable/jenkins
```
