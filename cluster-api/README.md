# Cluster-API

## Bootstrap

- Docs
  - [Getting Started](https://cluster-api-aws.sigs.k8s.io/getting-started.html)

- Command

```sh
# 一般登入
export AWS_REGION=us-east-1 # This is used to help encode your environment variables
export AWS_ACCESS_KEY_ID=<your-access-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
export AWS_SESSION_TOKEN=<session-token> # If you are using Multi-Factor Auth.

# AWS SSO 方法
# eval "$(aws configure export-credentials --profile <your-profile-name> --format env)"
eval "$(aws configure export-credentials --profile PowerUserAccess-714902217101 --format env)"

# The clusterawsadm utility takes the credentials that you set as environment
# variables and uses them to create a CloudFormation stack in your AWS account
# with the correct IAM resources.
clusterawsadm bootstrap iam create-cloudformation-stack

# Create the base64 encoded credentials using clusterawsadm.
# This command uses your environment variables and encodes
# them in a value to be stored in a Kubernetes Secret.
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)

# 使用以下指令檢查還有哪些環境變數要設定 (可選)
clusterctl generate provider --infrastructure aws --describe

# Finally, initialize the management cluster (他會用你 kubectl 選的 cluster 安裝必要組件)
clusterctl init --infrastructure aws
```

## 