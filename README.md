# K8s-Infra

東西不多，先用 Monorepo 架構

## 配置步驟

```bash
aws sso login --profile AdministratorAccess-714902217101
eval "$(aws configure export-credentials --profile AdministratorAccess-714902217101 --format env)"
```

```bash
aws iam create-group --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEventBridgeFullAccess --group-name kops
aws iam create-user --user-name kops
aws iam add-user-to-group --user-name kops --group-name kops
aws iam create-access-key --user-name kops

ACCOUNT_ID=$(aws sts get-caller-identity --output text --query 'Account')
# define a role trust policy that opens the role to users in your account (limited by IAM policy)
POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')
# create a role named KubernetesAdmin (will print the new role's ARN)
aws iam create-role \
  --role-name KubernetesAdmin \
  --description "Kubernetes administrator role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'
```

```bash
S3BUCKET_NAME_PREFIX=0-k8s-rem-club
REGION=us-east-1
# Cluster State store
aws s3api create-bucket \
    --bucket ${S3BUCKET_NAME_PREFIX}-state-store \
    --region ${REGION}
aws s3api put-bucket-versioning \
    --bucket ${S3BUCKET_NAME_PREFIX}-state-store \
    --versioning-configuration Status=Enabled
# Cluster OIDC store
aws s3api create-bucket \
    --bucket ${S3BUCKET_NAME_PREFIX}-oidc-store \
    --region ${REGION} \
    --object-ownership BucketOwnerPreferred
aws s3api put-public-access-block \
    --bucket ${S3BUCKET_NAME_PREFIX}-oidc-store \
    --public-access-block-configuration BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
aws s3api put-bucket-acl \
    --bucket ${S3BUCKET_NAME_PREFIX}-oidc-store \
    --acl public-read
```

```bash
export NAME=0k8sremclub.k8s.local
export KOPS_STATE_STORE=s3://${S3BUCKET_NAME_PREFIX}-state-store
kops create cluster \
    --name=${NAME} \
    --cloud=aws \
    --zones=us-east-1a \
    --topology=private \
    --discovery-store=s3://${S3BUCKET_NAME_PREFIX}-oidc-store/${NAME}/discovery \
    --ssh-public-key=$HOME/.ssh/REM-K8S.pub
```

```bash
# cluster
kops edit cluster --name ${NAME}

# apiVersion: kops.k8s.io/v1alpha2
# kind: Cluster
# metadata:
#   creationTimestamp: "2023-09-10T19:49:18Z"
#   name: 0k8sremclub.k8s.local
# spec:
#   nodeProblemDetector:
#     enabled: true
#     memoryRequest: 32Mi
#     cpuRequest: 10m
#   certManager:
#     enabled: true
#   metricsServer:
#     enabled: true
#     insecure: false
#   api:
#     loadBalancer:
#       class: Network
#       type: Public
#       crossZoneLoadBalancing: true
#   authentication:
#     aws:
#       backendMode: CRD
#       clusterID: 0k8sremclub.k8s.local
#       identityMappings:
#       - arn: arn:aws:iam::714902217101:role/KubernetesAdmin
#         username: admin:{{ SessionName }}
#         groups:
#         - system:masters
#       - arn: arn:aws:iam::714902217101:user/Alice
#         username: alice
#         groups:
#         - system:masters
#   networking:
#     cilium:
#       enableNodePort: true
#       hubble:
#         enabled: true
#   awsLoadBalancerController:
#     enabled: false
```

```bash
# Worker node instance group
kops edit ig --name=${NAME} nodes-us-east-1a

# apiVersion: kops.k8s.io/v1alpha2
# kind: InstanceGroup
# metadata:
#   creationTimestamp: "2023-09-10T19:49:19Z"
#   generation: 1
#   labels:
#     kops.k8s.io/cluster: 0k8sremclub.k8s.local
#   name: nodes-us-east-1a
# spec:
#   image: 099720109477/ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-20230728
#   machineType: t3.small # t2.micro, t3.medium
#   maxSize: 5
#   minSize: 2
#   role: Node
#   subnets:
#   - us-east-1a
```

```bash
# control-plane-instance group
kops edit ig --name=${NAME} control-plane-us-east-1a

# apiVersion: kops.k8s.io/v1alpha2
# kind: InstanceGroup
# metadata:
#   creationTimestamp: "2023-09-10T19:49:19Z"
#   generation: 1
#   labels:
#     kops.k8s.io/cluster: 0k8sremclub.k8s.local
#   name: control-plane-us-east-1a
# spec:
#   image: 099720109477/ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-20230728
#   machineType: t3.medium # t3.small
#   maxSize: 1
#   minSize: 1
#   role: Master
#   subnets:
#   - us-east-1a
```

```bash
kops update cluster --name ${NAME} --yes --admin
kops validate cluster --wait 300m
kubectl -n kube-system get po
```

```bash
helm upgrade --install --namespace kube-system --repo https://helm.cilium.io cilium cilium --version 1.12.1 --values - <<EOF
agent: false

operator:
  enabled: false

cni:
  install: false

hubble:
  enabled: false

  relay:
    enabled: false

  ui:
    # enable Hubble UI
    enabled: true

    standalone:
      # enable Hubble UI standalone deployment
      enabled: true

    # -- hubble-ui ingress configuration.
    ingress:
      enabled: true
      annotations: 
        cert-manager.io/cluster-issuer: letsencrypt-prod
      hosts:
        - hubble.see2.club
      labels: {}
      tls:
        - secretName: hubble-ui-cert
          hosts:
            - hubble.see2.club
EOF

```
