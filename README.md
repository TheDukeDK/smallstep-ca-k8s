# Demo of Internal ACME CA with Cert Manager

This repo contains a simple **demo** setup for using smallsteps 
[certificate authority(CA)](https://github.com/smallstep/certificates)
and [cert-manager](https://cert-manager.io/docs/configuration/acme/) deployed 
on a K8s cluster to act like LetsEncrypt. This allows cluster workloads exposed 
with Ingress to get automated certificate generation and renewal.

You, **OBVIOUSLY**, would not use this demo repo as is. The certs and passwords 
are plain text and are the ones generated when I created this.

Also there is a some hardcoded things like the host in the ingress, CA url, etc.

Again it is a **personal** demo for understanding.

# The Pieces

* Step Certificates(`step-certificates`): Smallsteps Certificate Authority(CA) running in 
the cluster. It has an ACME provisioner configured in the values.yaml file.

* Cert Manager(`cert-manager`): A certificate manager running in the cluster to manage 
certificate requests which will be stored as K8s secrets. We configure a cluster 
wide ACME issuer(`acme-issuer`) which points to the step-ca's ACME provisioner.

* Nginx Ingress Controller(`ingress-nginx-controller`): A controller for ingress's 
within the cluster. E.g. it will route requests to services based on the host/path. 

* Echo Server(`echo1`): Simple echo server which will return some info about the request.

## Setup Process
You obviosuly need a K8s cluster. This demo was done using version 1.21 on **EKS**. 
You will also need Helm installed.

The basic process looks like the following.

1. Install step-ca with its [Helm](https://github.com/smallstep/helm-charts/tree/master/step-certificates) 
chart.

```console
helm install -f step-ca/values.yaml \
     --set inject.secrets.ca_password=$(cat step-ca/password.txt) \
     --set inject.secrets.provisioner_password=$(cat step-ca/password.txt) \
     --set service.targetPort=9000 \
     step-certificates smallstep/step-certificates
```

2. Install cert-manager

```console
kubectl apply -f cert-manager/cert-manager.yaml
```

3. Configure cert-manager

Smallsteps CA `step-ca` is available over TLS so you need to **patch** `cert-manager` 
with the CA's root cert. Basically apply the ConfigMap for the cert and a patch for 
cert-manager. Finally apply an ACME [Issuer](https://cert-manager.io/docs/configuration/acme/#configuration) 
which will use the `step-ca` as the authority.

```console
kubectl apply -f cert-manager/internal-ca.yaml
kubectl patch deployment cert-manager -n cert-manager --patch "$(cat cert-manager/cm-ca-patch.yaml)"
kubectl apply -f cert-manager/acme-issuer.yaml
```

Wait for the cert-manager to be restarted then apply the ACME issuer.

```console
kubectl apply -f cert-manager/acme-issuer.yaml
```

4. Deploy the ingress controller

```console
kubectl apply -f ingress-controller/nginx-ingress-controller.yaml
```

5. Deploy the `echo1` service

```console
kubectl apply -f echo/echo1.yaml
```

You should have a secret containing the cert called `echo1-tls` which contains 
The base64 encoded cert signed by the step-certificates.

6. Configure DNS

At this point if you want to browse to the echo server and look at the cert 
you will need to create a DNS record for the host defined in the Ingress and 
have it point to the AWS Load Balancer created when you deployed the Ingress.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo1-ingress
  annotations:
    cert-manager.io/cluster-issuer: "acme-issuer"
spec:
  tls:
  - hosts:
    - echo1.<YOUR DOMAIN>.com   #<-- Need a DNS entry
    secretName: echo1-tls
  rules:
  - host: echo1.<YOUR DOMAIN>.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: echo1
            port:
              number: 8080
  ingressClassName: nginx
```

## Notes

* You could replace the *generated* root and intermediate certificates with 
your own. 

* You can easily generate a values file for `step-ca` by using the `step-cli`.
> [Install step CLI](https://smallstep.com/docs/step-cli/installation) and run 
> `step ca init --helm > values.yaml`.

