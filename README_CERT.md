# Setup EKS Cluster and LetsEncrypt with Certificate Manager

- AWS Route53 DNS and HostedZone are required.

## Setup Cluster
```bash
 eksctl create cluster -f cluster-spot-dns.yml
```

## Setup IAM account
```bash
 aws iam create-user --user-name cert-user
 aws iam create-policy --policy-name cert-user-policy --policy-document file://lets-encrypt/policy.json
 POLICYARN=$(aws iam list-policies | grep -Eo "arn:aws:iam.+cert-user-policy")
 aws iam attach-user-policy --user-name cert-user --policy-arn $POLICYARN
 aws iam create-access-key --user-name cert-user | tee /tmp/cert-user.json
```

## Setup Environments
```bash
 $ dfEmail="your@email.com.br"
 $ dfHostedZoneID="AWS-Hosted-Zone-ID"
 $ dfAccessKeyID="IAM-Access-Key-ID"
 $ dfDns="your.dns.com"
 
 for file in $(ls lets-encrypt) ; do
    for a in $(echo 'dfEmail  dfHostedZoneID  dfAccessKeyID  dfDns') ; do
      # MAC, otherwise replace -ibackp -> -i 
      sed -i.backp "s?${a}?${(P)a}?g" "lets-encrypt/${file}"    
   done
 done
```


## Setup cert-manager
```bash
 kubectl apply --validate=false -f lets-encrypt/00-crds.yaml
 kubectl create ns cert-manager
 helm repo add jetstack https://charts.jetstack.io 
 helm repo update
 
 helm install cert-manager --namespace cert-manager --version v0.12.0 jetstack/cert-manager
 kubectl get pods -n cert-manager
```

## Setup Certificate
```bash
 echo $(cat /tmp/cert-user.json | grep -E ":.{30}(/)?.+" | cut -d'"' -f4) > secretkey
 kubectl create secret generic acme-route53 --from-file=secret-access-key=./secretkey -n cert-manager
 kubectl describe secret acme-route53 -n cert-manager
 kubectl get secret acme-route53 -n cert-manager -o=yaml
 
 kubectl apply -f lets-encrypt/cluster-issue.yaml -n cert-manager
 kubectl get clusterissuer -n cert-manager
 kubectl describe clusterissuer letsencrypt-prod -n cert-manager 
 
 kubectl apply -f lets-encrypt/certificate.yaml -n cert-manager
 kubectl get certificate -n cert-manager
 kubectl describe certificate acme-route53-tls -n cert-manager  
 
 CR=$(kubectl get CertificateRequest -n cert-manager | grep -Eo "acme-route53-tls-\d+")
 kubectl describe CertificateRequest $CR -n cert-manager
```

## Setup hello-world
```bash
 kubectl apply -f lets-encrypt/hello-ingress.yaml -n default
```

## Debug
```bash
# logs from certificate process
kube logs -f pod/cert-manager-9dd764fb8-rblvx -n cert-manager

# show secret value
kubectl get secret acme-route53 -n cert-manager -o=yaml | \
grep -E "secret-access-key.+$" | cut -d":" -f2 | base64 -d

```

## CleanUp
```bash
 kubectl delete -f lets-encrypt/certificate.yaml -n cert-manager
 kubectl delete -f lets-encrypt/cluster-issue.yaml -n cert-manager
 kubectl delete secret acme-route53 -n cert-manager
 kubectl delete -f lets-encrypt/hello-ingress.yaml -n default 
 aws iam delete-access-key --user-name cert-user --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/cert-user.json)
 aws iam delete-policy --policy-name cert-user-policy
 aws iam delete-user --user-name cert-user
 
 eksctl delete cluster -f cluster-spot-dns.yml
```

## Links
 - https://medium.com/cloud-prodigy/configure-letsencrypt-and-cert-manager-with-kubernetes-3156981960d9
 - https://myhightech.org/posts/20210402-cert-manager-on-eks/