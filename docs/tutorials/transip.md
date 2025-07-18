# TransIP

This tutorial describes how to setup ExternalDNS for usage within a Kubernetes cluster using TransIP.

Make sure to use **>=0.5.14** version of ExternalDNS for this tutorial, have at least 1 domain registered at TransIP and enabled the API.

## Enable TransIP API and prepare your API key

To use the TransIP API you need an account at TransIP and enable API usage as described in the [knowledge base](https://www.transip.eu/knowledgebase/entry/77-want-use-the-transip-api/). With the private key generated by the API, we create a kubernetes secret:

```console
kubectl create secret generic transip-api-key --from-file=transip-api-key=/path/to/private.key
```

## Deploy ExternalDNS

Below are example manifests, for both cluster without or with RBAC enabled. Don't forget to replace `YOUR_TRANSIP_ACCOUNT_NAME` with your TransIP account name.
In these examples, an example domain-filter is defined. Such a filter can be used to prevent ExternalDNS from touching any domain not listed in the filter. Refer to the docs for any other command-line parameters you might want to use.

### Manifest (for clusters without RBAC enabled)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.18.0
        args:
        - --source=service # ingress is also possible
        - --domain-filter=example.com # (optional) limit to only example.com domains
        - --provider=transip
        - --transip-account=YOUR_TRANSIP_ACCOUNT_NAME
        - --transip-keyfile=/transip/transip-api-key
        volumeMounts:
        - mountPath: /transip
          name: transip-api-key
          readOnly: true
      volumes:
      - name: transip-api-key
        secret:
          secretName: transip-api-key
```

### Manifest (for clusters with RBAC enabled)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["discovery.k8s.io"]
  resources: ["endpointslices"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.18.0
        args:
        - --source=service # ingress is also possible
        - --domain-filter=example.com # (optional) limit to only example.com domains
        - --provider=transip
        - --transip-account=YOUR_TRANSIP_ACCOUNT_NAME
        - --transip-keyfile=/transip/transip-api-key
        volumeMounts:
        - mountPath: /transip
          name: transip-api-key
          readOnly: true
      volumes:
      - name: transip-api-key
        secret:
          secretName: transip-api-key
```

## Deploying an Nginx Service

Create a service file called 'nginx.yaml' with the following contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: my-app.example.com
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Note the annotation on the service; this is the name ExternalDNS will create and manage DNS records for.

ExternalDNS uses this annotation to determine what services should be registered with DNS. Removing the annotation will cause ExternalDNS to remove the corresponding DNS records.

Create the deployment and service:

```console
kubectl create -f nginx.yaml
```

Depending where you run your service it can take a little while for your cloud provider to create an external IP for the service.

Once the service has an external IP assigned, ExternalDNS will notice the new service IP address and synchronize the TransIP DNS records.

## Verifying TransIP DNS records

Check your [TransIP Control Panel](https://transip.eu/cp) to view the records for your TransIP DNS zone.

Click on the zone for the one created above if a different domain was used.

This should show the external IP address of the service as the A record for your domain.
