
# Setup EKS Cluster with EC2 Spot Request and Group Access

## Setup
```bash
$ eksctl create cluster -f cluster-spot.yml

#https://github.com/weaveworks/eksctl/blob/main/examples
```
## EKS Group Access

+ a kube-admin role which will have admin rights in our EKS cluster
+ a kube-dev role which will give access to the development namespace in our EKS cluster

```bash
 export ACCOUNT_ID="user_account_id"
 POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; 
echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')

 echo ACCOUNT_ID=$ACCOUNT_ID
 echo POLICY=$POLICY

# groups
 aws iam create-group --group-name kube-admin
 aws iam create-group --group-name kube-dev

# roles admin and develop
 aws iam create-role \
  --role-name kube-admin \
  --description "Kubernetes administrator role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'

 aws iam create-role \
  --role-name kube-dev \
  --description "Kubernetes developer role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'

# admin policy
 ADMIN_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/kube-admin"
    }
  ]
}')
 echo ADMIN_GROUP_POLICY=$ADMIN_GROUP_POLICY

 aws iam put-group-policy \
--group-name kube-admin \
--policy-name kube-admin-policy \
--policy-document "$ADMIN_GROUP_POLICY"

# dev policy
 DEV_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/kube-dev"
    }
  ]
}')
 echo DEV_GROUP_POLICY=$DEV_GROUP_POLICY

 aws iam put-group-policy \
--group-name kube-dev \
--policy-name kube-dev-policy \
--policy-document "$DEV_GROUP_POLICY"

# IAM groups
 aws iam list-groups

# create users
 aws iam create-user --user-name xxadmin
 aws iam create-user --user-name xxdev

# add users to groups
 aws iam add-user-to-group --group-name kube-admin --user-name xxadmin
 aws iam add-user-to-group --group-name kube-dev --user-name xxdev

# check
 aws iam get-group --group-name kube-admin
 aws iam get-group --group-name kube-dev

# create access keys
 aws iam create-access-key --user-name xxadmin | tee /tmp/admin.json
 aws iam create-access-key --user-name xxdev | tee /tmp/dev.json

#
 eksctl create iamidentitymapping \
  --cluster cluster-spot \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/kube-admin \
  --username xxadmin \
  --group system:masters

 eksctl create iamidentitymapping \
  --cluster cluster-spot \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/kube-dev \
  --username xxdev

# check
 kubectl get configmap -n kube-system aws-auth -o yaml
 eksctl get iamidentitymapping --cluster cluster-spot

# RBAC dev
 unset AWS_SECRET_ACCESS_KEY
 unset AWS_ACCESS_KEY_ID
 unset AWS_PROFILE
 kubectl apply -f ./rbac-develop -n development 
 
# export KUBECONFIG dev
 kubectl config view --flatten --minify > kubeconfig-dev
 export KUBECONFIG=tmp

 aws eks update-kubeconfig \
  --name cluster-spot \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/kube-dev \
  --alias cluster-spot
 kubectl config view --flatten --minify > kubeconfig-dev
 export KUBECONFIG=kubeconfig-dev

# test 
 kubectl get pods -n development
 kubectl get pods

# test xxuser
 export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/dev.json)
 export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/dev.json)
 kubectl get pods -n development
 
# test admin
 export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/admin.json)
 export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/admin.json)
 kubectl get pods -n development

```
## Clean-Up
```bash
 unset AWS_SECRET_ACCESS_KEY
 unset AWS_ACCESS_KEY_ID
 unset KUBECONFIG
 eksctl delete iamidentitymapping --cluster cluster-spot --arn arn:aws:iam::${ACCOUNT_ID}:role/kube-admin
 eksctl delete iamidentitymapping --cluster cluster-spot --arn arn:aws:iam::${ACCOUNT_ID}:role/kube-dev

 aws iam remove-user-from-group --group-name kube-admin --user-name xxadmin
 aws iam remove-user-from-group --group-name kube-dev --user-name xxdev

 aws iam delete-group-policy --group-name kube-admin --policy-name kube-admin-policy 
 aws iam delete-group-policy --group-name kube-dev --policy-name kube-dev-policy 

 aws iam delete-group --group-name kube-admin
 aws iam delete-group --group-name kube-dev

 aws iam delete-access-key --user-name xxadmin --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/admin.json)
 aws iam delete-access-key --user-name xxdev --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/dev.json)

 aws iam delete-user --user-name xxadmin
 aws iam delete-user --user-name xxdev

 aws iam delete-role --role-name kube-admin
 aws iam delete-role --role-name kube-dev

 eksctl delete cluster -f cluster-spot.yml
```