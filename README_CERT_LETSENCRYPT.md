# Setup EKS Cluster and LetsEncrypt with nginx-ingress

- AWS Route53 DNS and HostedZone are required.

## Update Helm charts
```bash
 helm repo add jetstack https://charts.jetstack.io 
 helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
 helm repo update
```

## Setup Cluster
```bash
 eksctl create cluster -f cluster-spot.yml
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

## Setup ClusterIssue
```bash
 emailCert="example.email.com"
 sed -i.backp "s?emailCert?${emailCert}?g" "hello-world/cluster-issuer.yaml"
 kubectl apply -f hello-world/cluster-issuer.yaml
 kubectl get clusterissuer
 kubectl describe clusterissuer letsencrypt-prod
 ```

## Setup App and Ingress
```bash
 # The subdomain should be appointed to Nginx Ingress Controller
 subDomain="subdomain.example.com"
 sed -i.backp "s?subDomain?${subDomain}?g" "hello-world/hello-ingress.yaml"
 kubectl apply -f hello-world
```

## Debug
```bash
# logs from certificate process
kube logs -f pod/cert-manager-xxxxxx -n cert-manager
```

## CleanUp
```bash
 eksctl delete cluster -f cluster-spot.yml
```
