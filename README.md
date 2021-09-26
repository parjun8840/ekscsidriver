
[![|](https://img.shields.io/badge/Docker-2CA5E0?style=for-the-badge&logo=docker&logoColor=white)](https://hub.docker.com/r/parjun8840/django-app02)
![|](https://img.shields.io/badge/kubernetes-326ce5.svg?&style=for-the-badge&logo=kubernetes&logoColor=white)

# Installing CSI driver and creating an admin user to access the cluster

### 1. Configuring EKS access
#### Setting up kube config
> aws eks --region YOUR_REGION_NAME update-kubeconfig --name YOUR_CLUSTER_NAME

##### Create any random IAM user. Configure aws cli Access and Secret key. 

**Note:** To be an admin of EKS cluster we don't need to attach any IAM policy to the user but for other AWS spefic things we  need to attach policies accordingly. 
**System:masters** is a group which is hardcoded into the Kubernetes API server source code as having unrestricted rights to the Kubernetes API server.
To grant additional AWS users or roles the ability to interact with your cluster, you must edit the aws-auth ConfigMap within Kubernetes. Verify the user's details 
> aws sts get-caller-identity

Edit aws-auth ConfigMap.
```
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::XXXXXX:role/ops-worker-dev-eks
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - userarn: arn:aws:iam::XXXXXX:user/randomuser
      username: randomuser
      groups:
      - system:masters
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
```
Test the user by executing below commands
> aws sts get-caller-identity
> kubectl get nodes

### 2. Get OIDC provider url and create an identity provider
To get OIDC provider you can use cli:
>  aws eks describe-cluster --name YOUR_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text
```https://oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXX```

OR

You can check from console. EKS -> Clusters -> YOUR_CLUSTER_NAME -> configuration -> OpenID Connect provider URL

Add an Identity provider from the IAM option:
- IAM -> Identity providers -> Add provider (choose- OpenID Connect)
- Provider URL- https://oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXX ( Click- Get thumbprint)
- Add 	Audience- sts.amazonaws.com

 ### 3. Create an IAM policy "AmazonEKS_EBS_CSI_Driver_Policy"
Download IAM policy and create a policy with the name "AmazonEKS_EBS_CSI_Driver_Policy" :
> curl -o example-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v1.0.0/docs/example-iam-policy.json
> aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy --policy-document file://example-iam-policy.json

```
example-iam-policy.json 
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:ModifyVolume",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumesModifications"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateTags"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:snapshot/*"
      ],
      "Condition": {
        "StringEquals": {
          "ec2:CreateAction": [
            "CreateVolume",
            "CreateSnapshot"
          ]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteTags"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:snapshot/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/CSIVolumeName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/CSIVolumeName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/CSIVolumeSnapshotName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    }
  ]
}
```

 ### 4. Create an IAM role "AmazonEKS_EBS_CSI_DriverRole"
 Create IAM role with the name "AmazonEKS_EBS_CSI_DriverRole" from the console and attach trust policy to it.
- Attach the previously created policy to it- "AmazonEKS_EBS_CSI_Driver_Policy".
- Attach the trust policy to this role. Note: Edit if it already exists.

Trust policy: Get the OIDC url from step 2.
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::XXXXXXXXXX:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXX"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXX:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```
 ### 5. Create SA "ebs-csi-controller-sa". Annotate the SA with the rolename.
 > kubectl create sa ebs-csi-controller-sa -nkube-system
 > kubectl describe sa ebs-csi-controller-sa -nkube-system | grep Annotations Annotations
 
 ```eks.amazonaws.com/role-arn: arn:aws:iam::XXXXXXXXX:role/AmazonEKS_EBS_CSI_DriverRole```
 
  ### 6.  Install the aws csi driver. 
  Check existing storage class.
  > kubectl get sc
  ```
  NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  113m
  ```
  Cloning CSI driver repo and edit the region in "kustomization.yaml" file.
  
 > git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
 cd aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr
cat kustomization.yaml 
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../../base
images:
  - name: k8s.gcr.io/provider-aws/aws-ebs-csi-driver
    newName: 602401143452.dkr.ecr.us-east-1.amazonaws.com/eks/aws-ebs-csi-driver
    newTag: v1.2.1
  - name: k8s.gcr.io/sig-storage/csi-provisioner
    newName: public.ecr.aws/eks-distro/kubernetes-csi/external-provisioner
    newTag: v2.1.1-eks-1-18-3
  - name: k8s.gcr.io/sig-storage/csi-attacher
    newName: public.ecr.aws/eks-distro/kubernetes-csi/external-attacher
    newTag: v3.1.0-eks-1-18-3
  - name: k8s.gcr.io/sig-storage/livenessprobe
    newName: public.ecr.aws/eks-distro/kubernetes-csi/livenessprobe
    newTag: v2.2.0-eks-1-18-3
  - name: k8s.gcr.io/sig-storage/csi-node-driver-registrar
    newName: public.ecr.aws/eks-distro/kubernetes-csi/node-driver-registrar
    newTag: v2.1.0-eks-1-18-3
````
```
kubectl apply -k ../ecr
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
serviceaccount/ebs-csi-controller-sa configured
serviceaccount/ebs-csi-node-sa created
clusterrole.rbac.authorization.k8s.io/ebs-csi-node-role created
clusterrole.rbac.authorization.k8s.io/ebs-external-attacher-role created
clusterrole.rbac.authorization.k8s.io/ebs-external-provisioner-role created
clusterrole.rbac.authorization.k8s.io/ebs-external-resizer-role created
clusterrole.rbac.authorization.k8s.io/ebs-external-snapshotter-role created
clusterrolebinding.rbac.authorization.k8s.io/ebs-csi-attacher-binding created
clusterrolebinding.rbac.authorization.k8s.io/ebs-csi-node-getter-binding created
clusterrolebinding.rbac.authorization.k8s.io/ebs-csi-provisioner-binding created
clusterrolebinding.rbac.authorization.k8s.io/ebs-csi-resizer-binding created
clusterrolebinding.rbac.authorization.k8s.io/ebs-csi-snapshotter-binding created
deployment.apps/ebs-csi-controller created
poddisruptionbudget.policy/ebs-csi-controller created
daemonset.apps/ebs-csi-node created
csidriver.storage.k8s.io/ebs.csi.aws.com created
```
```
 kubectl get pod -nkube-system
NAME                                  READY   STATUS    RESTARTS   AGE
aws-node-bfsq9                        1/1     Running   0          114m
aws-node-mhttd                        1/1     Running   0          113m
coredns-66c454664-75jjz              1/1     Running   0          117m
coredns-66cb5534656-zq7nc              1/1     Running   0          117m
ebs-csi-controller-5db87XXXXX-oorx2   6/6     Running   0          45s
ebs-csi-controller-5db87XXXXX-nnpg2   6/6     Running   0          45s
ebs-csi-node-2mkv9                    3/3     Running   0          44s
ebs-csi-node-q4ghf                    3/3     Running   0          44s
kube-proxy-8rnxz                      1/1     Running   0          113m
kube-proxy-jtcvh                      1/1     Running   0          114m
```

Make sure no errors in any of the controller Pods.
```
# kubectl logs -f ebs-csi-controller-5db87XXXXX-oorx2 -c ebs-plugin -nkube-system
I0925 17:20:19.337223       1 metadata.go:101] retrieving instance data from ec2 metadata
I0925 17:20:19.339489       1 metadata.go:108] ec2 metadata is available

# kubectl logs -f ebs-csi-controller-5db87XXXXX-oorx2 -c csi-provisioner -nkube-system
W0925 17:20:27.763570       1 feature_gate.go:235] Setting GA feature gate Topology=true. It will be removed in a future release.
I0925 17:20:27.763669       1 feature_gate.go:243] feature gates: &{map[Topology:true]}
I0925 17:20:27.763684       1 csi-provisioner.go:132] Version: v2.1.1-0-g353098c90
I0925 17:20:27.763706       1 csi-provisioner.go:155] Building kube configs for running in cluster...
I0925 17:20:27.783381       1 connection.go:153] Connecting to unix:///var/lib/csi/sockets/pluginproxy/csi.sock
I0925 17:20:27.783862       1 common.go:111] Probing CSI driver for readiness
I0925 17:20:27.786288       1 csi-provisioner.go:202] Detected CSI driver ebs.csi.aws.com
I0925 17:20:27.787292       1 csi-provisioner.go:241] CSI driver supports PUBLISH_UNPUBLISH_VOLUME, watching VolumeAttachments
I0925 17:20:27.787361       1 csi-provisioner.go:337] Supports migration from in-tree plugin: kubernetes.io/aws-ebs
I0925 17:20:27.787730       1 controller.go:756] Using saving PVs to API server in background
I0925 17:20:27.789938       1 leaderelection.go:243] attempting to acquire leader lease kube-system/ebs-csi-aws-com...
I0925 17:20:27.810132       1 leaderelection.go:253] successfully acquired lease kube-system/ebs-csi-aws-com
I0925 17:20:27.810287       1 leader_election.go:205] became leader, starting
I0925 17:20:27.810765       1 reflector.go:219] Starting reflector *v1.Node (1h0m0s) from k8s.io/client-go/informers/factory.go:134
I0925 17:20:27.811092       1 reflector.go:219] Starting reflector *v1.StorageClass (1h0m0s) from k8s.io/client-go/informers/factory.go:134
I0925 17:20:27.811388       1 reflector.go:219] Starting reflector *v1.PersistentVolumeClaim (15m0s) from k8s.io/client-go/informers/factory.go:134
I0925 17:20:27.811631       1 reflector.go:219] Starting reflector *v1.VolumeAttachment (1h0m0s) from k8s.io/client-go/informers/factory.go:134
I0925 17:20:27.811854       1 reflector.go:219] Starting reflector *v1.CSINode (1h0m0s) from k8s.io/client-go/informers/factory.go:134
I0925 17:20:27.910428       1 controller.go:835] Starting provisioner controller ebs.csi.aws.com_ebs-csi-controller-5db8789c48-jmrx2_efaba3e3-2509-444e-a32d-42c02df66a63!
I0925 17:20:27.910485       1 volume_store.go:97] Starting save volume queue
I0925 17:20:27.910684       1 reflector.go:219] Starting reflector *v1.PersistentVolume (15m0s) from sigs.k8s.io/sig-storage-lib-external-provisioner/v6/controller/controller.go:869
I0925 17:20:27.910695       1 reflector.go:219] Starting reflector *v1.StorageClass (15m0s) from sigs.k8s.io/sig-storage-lib-external-provisioner/v6/controller/controller.go:872
I0925 17:20:28.011170       1 controller.go:884] Started provisioner controller ebs.csi.aws.com_ebs-csi-controller-5db8789c48-jmrx2_efaba3e3-2509-444e-a32d-42c02df66a63!

```

### 7. Creating Storage Class and Testing it.
Create Storage Class:
```
# cat storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer

# kubectl apply -f storageclass.yaml
storageclass.storage.k8s.io/ebs-sc created
# kubectl get sc
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc          ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  7s
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  122m
# 

# kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
storageclass.storage.k8s.io/gp2 patched

# kubectl get sc
NAME     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc   ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  2m35s
gp2      kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  125m
# kubectl patch storageclass ebs-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/ebs-sc patched
[root@test-mongo-03 specs]# kubectl get sc
NAME               PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc (default)   ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  3m35s
gp2                kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  126m

```
Testing the Storage class:
```
# Source: djangoapp/charts/postgresql/templates/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: djangpostgresql
  namespace: default
  labels:
    app: djangpostgres
    chart: postgresql-0.1.0
    heritage: Helm
    release: webapp
    user: postgres
  annotations:
    docker.repo.kubernetes.io/name: "parjun8840"
    kubernetes.io/ingress.class: nginx
    generator: helm
    date: "20210925112940"
spec:
  serviceName: djangpostgresql
  replicas: 1
  selector:
    matchLabels:
      app: djangpostgres
  template:
    metadata:
      labels:
        app: djangpostgres
        release: webapp
    spec:
      initContainers:
      - name: pgsql-data-path-change
        image: parjun8840/handyimage:v1
        imagePullPolicy: IfNotPresent
        command: ["sh", "-c", "if test -d /var/lib/postgresql/9.3/main; then rsync -av /var/lib/postgresql /data; sed -i 's|/var/lib/postgresql/9.3/main|/data|g' /etc/postgresql/9.3/main/postgresql.conf;mv /var/lib/postgresql/9.3/main /opt/pg_backup;fi"]
        volumeMounts:
        - name: data-volume
          mountPath: /data01
      containers:
        - name: djangpostgres
          image: "parjun8840/postgresql:9.3"
          imagePullPolicy: IfNotPresent
          command: ["/usr/lib/postgresql/9.3/bin/postgres", "-D", "/data/postgresql/9.3/main", "-c", "config_file=/etc/postgresql/9.3/main/postgresql.conf"]
          ports:
          - name: http
            protocol: TCP
            containerPort: 5432
          volumeMounts:
          - name: data-volume
            mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data-volume
      namespace: default
    spec:
      storageClassName: ebs-sc
      accessModes: 
        - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi

# kubectl apply -f djangoapp/charts/postgresql/templates/statefulset.yaml
# kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ebs-claim   Bound    pvc-49abefbb-8971-4e64-8d22-XXXXXXXXXXXXX   20Gi        RWO            ebs-sc         12s
# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
pvc-49abefbb-8971-4e64-8d22-XXXXXXXX   20Gi        RWO            Delete           Bound    default/ebs-claim   ebs-sc                  19s
#
```


## License

Apache

