# k0s local env helm chart

## Overview

k0s setup for development purpose. 

Contains:
- openebs
- metallb
- cert-manager
- trust-manager
- ingress-nginx
- docker-registry

## Configuration

See [values.yaml](values.yaml)

## Install

### k0s

ref: https://docs.k0sproject.io/stable/install/

Install k0s

`curl --proto '=https' --tlsv1.2 -sSf https://get.k0s.sh | sudo sh`

Install controller with custom k0s.yaml

`sudo k0s install controller --single -c k0s.yaml`

Start controller

`sudo k0s start`

(Optional) Use k8s config from k0s

`sudo cat /var/lib/k0s/pki/admin.conf > ~/.kube/config`

Monitor setup with

`heml list -a -A`

Wait untill status=deployed on all charts except ingress-nginx 

### Current chart

`helm upgrade --create-namespace -n k0s-local  --install k0s-local .`

Install custom CA

```
kubectl get secrets -n cert-manager k0s-local-cert-root-ca-secret -o yaml \
      -o jsonpath="{.data.ca\.crt}" | \
      base64 -d > k0s-local-cert-root-ca.crt
sudo cp k0s-local-cert-root-ca.crt /usr/local/share/ca-certificates
sudo update-ca-certificates
```

Restart k0s controller to apply new CA

```
sudo k0s stop
sudo k0s start
```

### Update dns

Point your dns to 192.168.123.53. That is default value from values.yaml.

Sample for resolved. Here virbr0 interface is where lb anonced new ips.

```
sudo resolvectl llmnr virbr0 no
sudo resolvectl mdns virbr0 no
sudo resolvectl dns virbr0 192.168.123.53
sudo resolvectl domain virbr0 "~k0s.internal"
sudo resolvectl default-route virbr0 no
sudo resolvectl dnsovertls virbr0 no
```

```
resolvectl status
Link 4 (virbr0)
    Current Scopes: DNS
         Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 192.168.123.53
       DNS Servers: 192.168.123.53
        DNS Domain: ~k0s.internal
     Default Route: no
```

`curl https://image-regisry.lb-ingress.k0s.internal/v2/_catalog`

Must return 200 OK

## Testing

### Push image

Pull new image to registry

`podman pull debian:12 && podman push debian:12 image-regisry.lb-ingress.k0s.internal/debian:12`

Run new container

`kubectl run -ti --image image-regisry.lb-ingress.k0s.internal/debian:12 --rm test -- bash -c "read;echo OK"`

### Podinfo

Install Podinfo with ingress pointed to `podinfo.lb-ingress.k0s.internal`

`helm upgrade -i my-release oci://ghcr.io/stefanprodan/charts/podinfo -f podinfo-values.yaml`

Get info from podinfo

`curl https://podinfo.lb-ingress.k0s.internal`

Result

```
{
  "hostname": "my-release-podinfo-5776bdd57-8rdfv",
  "version": "6.9.2",
  "revision": "e86405a8674ecab990d0a389824c7ebbd82973b5",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.9.2",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.25.1",
  "num_goroutine": "9",
  "num_cpu": "8"
}%
```
