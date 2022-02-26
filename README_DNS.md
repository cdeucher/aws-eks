
# Setup EKS Cluster and Certificate Manager with Nginx-ingress 

## Setup Cluster
```bash
 eksctl create cluster -f cluster-spot-dns.yml
```

## Setup DNS and Certificate
```bash
 aws route53 list-hosted-zones
 aws acm list-certificates --region us-east-1
```
## Setup Environments
```bash
 $ dfDns="your.dns.com"
 
 for file in $(ls nginx-ingress) ; do
    sed -i.backp "s?dfDns?${dfDns}?g" "nginx-ingress/${file}"    
 done
```

## Manual steps: AWS Console Management
 - Create Route53 DNS.
 - Create Certificate Manager solicitation to the DNS.
 ```bash
  aws acm list-certificates --region us-east-1
  certificate=$(aws acm list-certificates --region us-east-1 | grep -Eo "arn:aws.+\w")
 ```

## Setup nginx-ingress
 ```bash
  helm repo add "bitnami" "https://charts.bitnami.com/bitnami"
  helm repo add "ingress-nginx" "https://kubernetes.github.io/ingress-nginx"

  helm install extdns --values nginx-ingress/values.external-dns.yaml bitnami/external-dns
  kubectl get svc | grep external-dns

  sed -i.backp "s?certificate-arn?${certificate}?g" nginx-ingress/values.nginx-ingress.yaml

  helm install nginx-ingress --values nginx-ingress/values.nginx-ingress.yaml ingress-nginx/ingress-nginx
 ```

## Deploy application
 ```bash
  # replace 'host' inside the file `hello-kubernetes.yaml`
  kubectl apply -f nginx-ingress/hello-kubernetes.yaml
  kubectl get ingress
 ```
 
## Clean-Up
 ```bash
  eksctl delete cluster -f cluster-spot-dns.yml
 ```



## Links
 - https://www.nginx.com/blog/deploying-nginx-ingress-controller-on-amazon-eks-how-we-tested/
 - https://joachim8675309.medium.com/adding-ingress-with-amazon-eks-6c4379803b2