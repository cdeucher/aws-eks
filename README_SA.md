
## Setup Service Accounts to EKS deploys

## Setup EKS Cluster
```bash
$ eksctl create cluster -f simple-cluster/cluster-spot.yml
```

## Setup SA
```
# Exports
 export ACCOUNT_ID="user_account_id"
 export REGION="us-east-1"
 
# Create your IAM OIDC Identity Provider for your cluster
eksctl utils associate-iam-oidc-provider --cluster cluster-spot --approve

# create bucket S3
aws s3 mb s3://eksworkshop-$ACCOUNT_ID-$REGION --region $REGION

# search for the policy
aws iam list-policies --query 'Policies[?PolicyName==`AmazonS3ReadOnlyAccess`].Arn'

# create a IAM role bound to a service account with read-only access to S3
eksctl create iamserviceaccount \
    --name iam-test \
    --namespace default \
    --cluster cluster-spot \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
    --approve \
    --override-existing-serviceaccounts

# verify your service account iam-test exists 
kubectl get sa iam-test

# deploy
kubectl apply -f simple-cluster/job-s3.yaml

# list
kubectl get job -l app=eks-iam-test-s3

# logs
kubectl logs -l app=eks-iam-test-s3

```
