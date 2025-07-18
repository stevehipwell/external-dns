# AWS Cloud Map API

This tutorial describes how to set up ExternalDNS for usage within a Kubernetes cluster with [AWS Cloud Map API](https://docs.aws.amazon.com/cloud-map/).

**AWS Cloud Map** API is an alternative approach to managing DNS records directly using the Route53 API. It is more suitable for a dynamic environment where service endpoints change frequently.
It abstracts away technical details of the DNS protocol and offers a simplified model. AWS Cloud Map consists of three main API calls:

* CreatePublicDnsNamespace – automatically creates a DNS hosted zone
* CreateService – creates a new named service inside the specified namespace
* RegisterInstance/DeregisterInstance – can be called multiple times to create a DNS record for the specified *Service*

Learn more about the API in the [AWS Cloud Map API Reference](https://docs.aws.amazon.com/cloud-map/latest/api/API_Operations.html).

## IAM Permissions

To use the AWS Cloud Map API, a user must have permissions to create the DNS namespace. You need to make sure that your nodes (on which External DNS runs) have an IAM instance profile with the `AWSCloudMapFullAccess` managed policy attached, that provides following permissions:

> Please be aware that this IAM role grants broad permissions across Route 53, and Service Discovery. For enhanced security, it's strongly recommended to review and restrict the actions and resources to the absolute minimum required for its intended purpose, following the principle of least privilege

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:GetHostedZone",
        "route53:ListHostedZonesByName",
        "route53:CreateHostedZone",
        "route53:DeleteHostedZone",
        "route53:ChangeResourceRecordSets",
        "route53:CreateHealthCheck",
        "route53:GetHealthCheck",
        "route53:DeleteHealthCheck",
        "route53:UpdateHealthCheck",
        "ec2:DescribeVpcs",
        "ec2:DescribeRegions",
        "servicediscovery:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

### IAM Permissions with ABAC

You can use Attribute-based access control(ABAC) for advanced deployments.

You can define AWS tags that are applied to services created by the controller. By doing so, you can have precise control over your IAM policy to limit the scope of the permissions to services managed by the controller, rather than having to grant full permissions on your entire AWS account.
To pass tags to service creation, use either CLI flags or environment variables:

*cli:* `--aws-sd-create-tag=key1=value1 --aws-sd-create-tag=key2=value2`

*environment:* `EXTERNAL_DNS_AWS_SD_CREATE_TAG=key1=value1\nkey2=value2`

Using tags, your `servicediscovery` policy can become:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ],
      "Condition": {
        "ForAllValues:StringLike": {
          "route53:ChangeResourceRecordSetsNormalizedRecordNames": ["*example.com", "marketing.example.com", "*-beta.example.com"],
          "route53:ChangeResourceRecordSetsActions": ["CREATE", "UPSERT", "DELETE"],
          "route53:ChangeResourceRecordSetsRecordTypes": ["A", "AAAA", "MX"]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "servicediscovery:ListNamespaces",
        "servicediscovery:ListServices"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "servicediscovery:CreateService",
        "servicediscovery:TagResource"
      ],
      "Resource": [
        "*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestTag/YOUR_TAG_KEY": "YOUR_TAG_VALUE"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "servicediscovery:DiscoverInstances"
      ],
      "Resource": [
        "*"
      ],
      "Condition": {
        "StringEquals": {
          "servicediscovery:NamespaceName": "YOUR_NAMESPACE_NAME"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "servicediscovery:RegisterInstance",
        "servicediscovery:DeregisterInstance",
        "servicediscovery:DeleteService",
        "servicediscovery:UpdateService"
      ],
      "Resource": [
        "*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/YOUR_TAG_KEY": "YOUR_TAG_VALUE"
        }
      }
    }
  ]
}
```

Additional resources:

* AWS IAM actions [documentation](https://www.awsiamactions.io/?o=servicediscovery%3A)

## Set up a namespace

Create a DNS namespace using the AWS Cloud Map API:

```console
aws servicediscovery create-public-dns-namespace --name "external-dns-test.my-org.com"
```

Verify that the namespace was truly created

```console
aws servicediscovery list-namespaces
```

## Deploy ExternalDNS

Connect your `kubectl` client to the cluster that you want to test ExternalDNS with.
Then apply the following manifest file to deploy ExternalDNS.

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
        env:
          - name: AWS_REGION
            value: us-east-1 # put your CloudMap NameSpace region
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=external-dns-test.my-org.com # Makes ExternalDNS see only the namespaces that match the specified domain. Omit the filter if you want to process all available namespaces.
        - --provider=aws-sd
        - --aws-zone-type=public # Only look at public namespaces. Valid values are public, private, or no value for both)
        - --txt-owner-id=my-identifier
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
  verbs: ["list","watch"]
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
        env:
          - name: AWS_REGION
            value: us-east-1 # put your CloudMap NameSpace region
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=external-dns-test.my-org.com # Makes ExternalDNS see only the namespaces that match the specified domain. Omit the filter if you want to process all available namespaces.
        - --provider=aws-sd
        - --aws-zone-type=public # Only look at public namespaces. Valid values are public, private, or no value for both)
        - --txt-owner-id=my-identifier
```

## Verify that ExternalDNS works (Service example)

Create the following sample application to test that ExternalDNS works.

> For services ExternalDNS will look for the annotation `external-dns.alpha.kubernetes.io/hostname` on the service and use the corresponding value.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.external-dns-test.my-org.com
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx

---

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
          name: http
```

After one minute check that a corresponding DNS record for your service was created in your hosted zone. We recommended that you use the [Amazon Route53 console](https://console.aws.amazon.com/route53) for that purpose.

## Custom TTL

The default DNS record TTL (time to live) is 300 seconds. You can customize this value by setting the annotation `external-dns.alpha.kubernetes.io/ttl`.
For example, modify the service manifest YAML file above:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.external-dns-test.my-org.com
    external-dns.alpha.kubernetes.io/ttl: "60"
spec:
    ...
```

This will set the TTL for the DNS record to 60 seconds.

## IPv6 Support

If your Kubernetes cluster is configured with IPv6 support, such as an [EKS cluster with IPv6 support](https://docs.aws.amazon.com/eks/latest/userguide/deploy-ipv6-cluster.html), ExternalDNS can
also create AAAA DNS records.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.external-dns-test.my-org.com
    external-dns.alpha.kubernetes.io/ttl: "60"
spec:
  ipFamilies:
    - "IPv6"
  type: NodePort
  ports:
    - port: 80
      name: http
      targetPort: 80
  selector:
    app: nginx
```

:information_source: The AWS-SD provider does not currently support dualstack load balancers and will only create A records for these at this time. See the AWS provider and the [AWS Load Balancer Controller Tutorial](./aws-load-balancer-controller.md) for dualstack load balancer support.

## Clean up

Delete all service objects before terminating the cluster so all load balancers get cleaned up correctly.

```console
kubectl delete service nginx
```

Give ExternalDNS some time to clean up the DNS records for you. Then delete the remaining service and namespace.

```console
$ aws servicediscovery list-services

{
    "Services": [
        {
            "Id": "srv-6dygt5ywvyzvi3an",
            "Arn": "arn:aws:servicediscovery:us-west-2:861574988794:service/srv-6dygt5ywvyzvi3an",
            "Name": "nginx"
        }
    ]
}
```

```console
aws servicediscovery delete-service --id srv-6dygt5ywvyzvi3an
```

```console
$ aws servicediscovery list-namespaces
{
    "Namespaces": [
        {
            "Type": "DNS_PUBLIC",
            "Id": "ns-durf2oxu4gxcgo6z",
            "Arn": "arn:aws:servicediscovery:us-west-2:861574988794:namespace/ns-durf2oxu4gxcgo6z",
            "Name": "external-dns-test.my-org.com"
        }
    ]
}
```

```console
aws servicediscovery delete-namespace --id ns-durf2oxu4gxcgo6z
```
