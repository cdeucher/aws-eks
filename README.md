
# Setup EKS Cluster with EC2 Spot Request and User Access

## Setup
```bash
$ eksctl create cluster -f cluster-spot.yml

#https://github.com/weaveworks/eksctl/blob/main/examples
```
## EKS User Access

```bash
$ aws iam create-user --user-name rbac-user
$ aws iam create-access-key --user-name rbac-user | tee /tmp/create_output.json
$ export ACCOUNT_ID="rbac_user_account_id"
$ kubectl get configmap -n kube-system aws-auth -o yaml | \
grep -v "creationTimestamp\|resourceVersion\|selfLink\|uid" | \
sed '/^  annotations:/,+2 d' > aws-auth.yaml
$ cat << EoF >> aws-auth.yaml
data:
  mapUsers: |
    - userarn: arn:aws:iam::${ACCOUNT_ID}:user/rbac-user
      username: rbac-user
EoF
$ kubectl apply -f aws-auth.yaml
# export new access keys
$ export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
$ export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
$ kubectl apply -f ./hello-world
$ kubectl get pods -n dev2

# Check Permissions
$ kubectl auth can-i list pods -n dev2

# https://www.eksworkshop.com/beginner/090_rbac/test_rbac_user_without_roles/
```

## 
```bash
$ arn=$(aws sts get-caller-identity | grep -oE "arn.+/\w+-\w+")
$ aws eks --region us-east-1 update-kubeconfig --name cluster-spot --role-arn ${arn}

# https://aws.amazon.com/pt/premiumsupport/knowledge-center/eks-cluster-connection/
```


## Clean-Up
```bash
$ aws iam delete-access-key --user-name rbac-user 
# OR IAM -> User -> Credentials -> exclude
$ aws iam delete-user --user-name rbac-user
$ eksctl delete cluster -f cluster-spot.yml
$ unset AWS_SECRET_ACCESS_KEY
$ unset AWS_ACCESS_KEY_ID
```

## More
- If you needed to use an existing VPC, you can use a config file like this:
  - https://eksctl.io/usage/creating-and-managing-clusters/
  - `--zones=us-east-1a,us-east-1b,us-east-1d`
- To enable CloudWatch:
  - `eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=us-east-1 --cluster=spotcluster`


## Links
- https://www.eksworkshop.com/beginner/090_rbac/test_rbac_user_without_roles/
- https://eksctl.io/introduction/
- https://github.com/weaveworks/eksctl/blob/main/examples
- https://aws.amazon.com/pt/blogs/compute/cost-optimization-and-resilience-eks-with-spot-instances/
- https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html#create-service-role
- https://docs.aws.amazon.com/pt_br/IAM/latest/UserGuide/id_roles_use_switch-role-cli.html
- https://docs.aws.amazon.com/cli/latest/reference/cloudformation/create-stack.html
- https://eksctl.io/usage/creating-and-managing-clusters/
- https://medium.com/@nageshblore/amazon-eks-access-control-for-kubectl-users-of-api-server-165c1071c525
- https://aws.amazon.com/pt/premiumsupport/knowledge-center/eks-cluster-connection/