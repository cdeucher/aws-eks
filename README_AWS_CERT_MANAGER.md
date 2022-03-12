
# Setup EKS Cluster and Certificate Manager with Nginx-ingress 
- AWS IAM
- AWS EKS
- AWS Route53
- AWS Elastic LoadBalancer
- AWS Certificate Manager
- External-DNS

## Setup DNS and Certificate
```bash
 aws route53 list-hosted-zones
 aws acm list-certificates --region us-east-1
```
## Manual steps: AWS Console Management
- Create Route53 Hosted zone.
- Request a public certificatem, in Domain names.
   - domain.com
   - *.domain.com # Add another name to this certificate
 ```bash
  aws acm list-certificates --region us-east-1
  certificate=$(aws acm list-certificates --region us-east-1 | grep -Eo "arn:aws.+\w")
 ```

## Setup Cluster
```bash
 eksctl create cluster -f cluster-spot-dns.yml
 ```

## Create IAM OIDC Identity Provider for the cluster
```bash
 eksctl utils associate-iam-oidc-provider --cluster cluster-spot --approve
```

## Create Policy
```bash
 aws iam create-policy --policy-name external-records-route53 --policy-document file://nginx-ingress/policy.json
 POLICY_ARN=$(aws iam list-policies | grep -Eo "arn:aws:iam.+external-records-route53")
```

## Create a Service Account
```bash
 eksctl create iamserviceaccount \
    --name external-dns \
    --namespace default \
    --cluster cluster-spot \
    --attach-policy-arn "${POLICY_ARN}" \
    --approve \
    --override-existing-serviceaccounts

 kubectl get sa external-dns
 
 ROLE_ARN=$(kubectl describe sa external-dns | grep -Eo "eks.amazonaws.com/role-arn.+")
```

## Setup Environments
```bash
 dfDns="your.dns.com"
 for file in $(ls nginx-ingress) ; do
    sed -i.backp "s?dfDns?${dfDns}?g" "nginx-ingress/${file}"
    sed -i.backp "s?dfServiceAccountArn?${ROLE_ARN}?g" "nginx-ingress/${file}"
 done
```

## Setup nginx-ingress and ExternalDNS
 ```bash
  helm repo add bitnami https://charts.bitnami.com/bitnami
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  helm repo update
  helm show values ingress-nginx/ingress-nginx > ingress-nginx.yaml

  # The add-on ExternalDNS that can allow Kubernetes service or ingress to create DNS records.
  # In our case, we have ExternalDNS create records on Route53. 
  # Options: https://github.com/bitnami/charts/tree/master/bitnami/external-dns
  helm install extdns --values nginx-ingress/values.external-dns.yaml bitnami/external-dns
  
  kubectl get svc | grep external-dns
  # DEBUG external-dns
  # check out the service account
  kube get pod/$(kubectl get pods | grep -Eo "extdns-external-dns[0-9a-zA-Z\-]+" ) -o=yaml | grep "serviceAccount"
  kube logs -f $(kubectl get pods | grep -Eo "extdns-external-dns[0-9a-zA-Z\-]+" )


  sed -i.backp "s?certificate-arn?${certificate}?g" nginx-ingress/values.nginx-ingress.yaml
  helm install nginx-ingress --values nginx-ingress/values.nginx-ingress.yaml ingress-nginx/ingress-nginx
  
  # Check out AWS Console -> EC2 -> LoadBalancers -> (select) -> Lister -> SSL port
  # The SSL ELB must be configured.
  # Check out AWS Console -> Certificates -> (select) -> Associated resources
  # The ELB from Nginx-ingress must be configured.
 ```

## Deploy application
 ```bash
  kubectl apply -f nginx-ingress/hello-kubernetes.yaml
  kubectl get ingress
 ```
 
## Clean-Up
 ```bash
  helm delete nginx-ingress
  helm delete extdns
  kubectl delete -f nginx-ingress/hello-kubernetes.yaml
  eksctl delete cluster -f cluster-spot-dns.yml
 ```



## Links
 - https://www.nginx.com/blog/deploying-nginx-ingress-controller-on-amazon-eks-how-we-tested/
 - https://joachim8675309.medium.com/adding-ingress-with-amazon-eks-6c4379803b2

## External-DNS Links
 - https://github.com/bitnami/charts/tree/master/bitnami/external-dns
 - https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md
 - https://www.padok.fr/en/blog/external-dns-route53-eks
 - https://www.stacksimplify.com/aws-eks/aws-alb-ingress/install-externaldns-on-aws-eks/