# Setup EKS Cluster and LetsEncrypt with Certificate Manager

- AWS Route53 DNS and HostedZone are required.

## Update Helm charts
```bash
 helm repo add jetstack https://charts.jetstack.io 
 helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
 helm update
```

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
      sed -i.backp "s?${a}?${(P)a}?g" "lets-encrypt/${file}"    
   done
 done
```

## Setup cert-manager
```bash
 kubectl create ns cert-manager 
 helm install cert-manager jetstack/cert-manager --namespace cert-manager \
 --set installCRDs=true
 
 kubectl get pods -n cert-manager
```

## Setup nginx-ingress
```bash
 helm install ingress-controller ingress-nginx/ingress-nginx
```

## Setup cluster-issue
```bash
 echo $(cat /tmp/cert-user.json | grep -E ":.{30}(/)?.+" | cut -d'"' -f4) > secretkey
 kubectl create secret generic iam-secret --from-file=secret-access-key=./secretkey -n cert-manager
 kubectl describe secret iam-secret
 kubectl get secret iam-secret -o=yaml
 
 kubectl apply -f lets-encrypt/cluster-issue.yaml
 kubectl get clusterissuer
 kubectl describe clusterissuer letsencrypt-prod
 
 kubectl apply -f lets-encrypt/certificate.yaml
 kubectl get certificate
 kubectl describe certificate acme-route53-tls
 
 # check secret created with tls certs
 kube get secrets | grep "kubernetes.io/tls"

 # logs
 CR=$(kubectl get CertificateRequest | grep -Eo "acme-route53-tls-\d+")
 kubectl describe CertificateRequest $CR
```

## Setup certificate
```bash
 
 kubectl apply -f lets-encrypt/certificate.yaml
 kubectl get certificate
 kubectl describe certificate acme-route53-tls
 
 # check secret created with tls certs
 kube get secrets | grep "kubernetes.io/tls"

 # logs
 CR=$(kubectl get CertificateRequest | grep -Eo "acme-route53-tls-\d+")
 kubectl describe CertificateRequest $CR
```

## Setup hello-world
```bash
 kubectl apply -f lets-encrypt/hello-ingress.yaml
```

## Debug
```bash
# logs from certificate process
kube logs -f pod/cert-manager-9dd764fb8-rblvx -n cert-manager

# show secret value
kubectl get secret iam-secret -o=yaml | \
grep -E "secret-access-key.+$" | cut -d":" -f2 | base64 -d

```

## CleanUp
```bash
 kubectl delete -f lets-encrypt/certificate.yaml 
 kubectl delete -f lets-encrypt/cluster-issue.yaml 
 kubectl delete secret iam-secret -n cert-manager 
 kubectl delete -f lets-encrypt/hello-ingress.yaml -n default 
 aws iam delete-access-key --user-name cert-user --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/cert-user.json)
 aws iam delete-policy --policy-name cert-user-policy
 aws iam delete-user --user-name cert-user
 
 eksctl delete cluster -f cluster-spot-dns.yml
```

## Links
 - https://medium.com/cloud-prodigy/configure-letsencrypt-and-cert-manager-with-kubernetes-3156981960d9
 - https://myhightech.org/posts/20210402-cert-manager-on-eks/
 - https://getbetterdevops.io/k8s-ingress-with-letsencrypt/
 - https://voyagermesh.com/docs/v12.0.0/guides/cert-manager/dns01_challenge/aws-route53/