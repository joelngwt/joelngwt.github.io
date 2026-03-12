---
layout: post
title: How to Migrate Ingress to Gateway API
tags:
  - kubernetes
  - aws
  - eks
---

## The Story
If you've been keeping up with the news in the Kubernetes world, you would have come across the news that Ingress NGINX would be deprecated at the end of March 2026. From their [official statement](https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/), about 50% of cloud native environments currently rely on this tool, so there's a high chance that you're impacted by this if you use Kubernetes.

You could migrate to another Ingress provider, but with [Gateway API](https://gateway-api.sigs.k8s.io/guides/getting-started/) being the successor, I would recommend trying to migrate to Gateway API instead. While the Kubernetes team has said that Ingress will continue to be supported and will not be removed, I have a story to tell from the software engineering world.

React also has two different types of systems - class components, and function components (hooks). The messaging was extremely similar: class components will continue to be supported, and will never be removed. Fast-forward 6 years, and class components are pretty much dead in the React ecosystem. Libraries are always designed for functional components and hooks, documentation is always using the function component syntax, and performance improvements made for the React core are applicable to hooks only (e.g. React 19's compiler).

That said, React and JavaScript moves a lot quicker than infrastructure, and infrastructure places a much higher value on reliability, stability, and uptime, so I expect that it would take much longer for Ingress to go the way of React Classes, if ever.

Still, it's something to be aware of when deciding where to go with your migration.

## The Guide
For this guide, I've chosen to use the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/). Similar to how Ingress requires a third-party Ingress Controller, the Gateway API also requires a controller.

For some reason, the official guides are fragmented and following it doesn't actually give you enough information to set up the whole thing successfully. I don't know why Kubernetes is like this.

Also, you can set this up alongside Ingress without issues. Just swap the target of your Route 53 entry to the new load balancer when you're ready.

# IAM Setup
First, you'll need to set up the IAM. There are [3 different methods](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/#configure-iam) to do this, but I've decided to use the EKS Pod Identity Agent for this. I will not go through all steps to do this, as there is a well-written guide by AWS on how to do this: [https://docs.aws.amazon.com/eks/latest/userguide/pod-id-agent-setup.html](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-agent-setup.html).

During the setup of the Pod Identity Agent, you'll need to configure the policy and role to be associated to the EKS Cluster. The policy can be obtained and created via:
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```

Then, create a trust file for the policy:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
```

Create the role:
```
aws iam create-role --role-name LBCPodIdentityRole --assume-role-policy-document file://trust-policy.json
```

Attach the policy to the role:
```
aws iam create-role --role-name LBCPodIdentityRole --assume-role-policy-document file://trust-policy.json
```

Then attach the policy to the EKS cluster:
```
aws eks create-pod-identity-association \
    --cluster-name <CLUSTER_NAME> \
    --namespace kube-system \
    --service-account aws-load-balancer-controller \
    --role-arn <ARN_OF_LBCPodIdentityRole>
```

# Install AWS Load Balancer Controller
The installation guide is here, along with some prerequisites: [https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/gateway/gateway/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/gateway/gateway/).

The main command to run would be the following. Change the version as necessary.
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```

Then, install it via a Helm Chart. This assumes that you would use an application load balancer on AWS:
```
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<CLUSTER_NAME> \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<AWS_REGION> \
  --set vpcId=<VPC_ID> \
  --set "controllerConfig.featureGates.ALBGatewayAPI=true"
```

The AWS Load Balancer Controller should now be ready to use.

# GatewayClass
You would now need to install a GatewayClass object in the cluster. This would spin up an application load balancer.
```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: alb
spec:
  controllerName: gateway.k8s.aws/alb
---
apiVersion: gateway.k8s.aws/v1beta1
kind: LoadBalancerConfiguration
metadata:
  name: alb-config
  namespace: default
spec:
  scheme: internet-facing
  loadBalancerName: eks
```

Check that it's running fine by running:
```
kubectl get gatewayclass alb -o wide
```

You should see something like this. The main thing to look out for would be `True` under the `ACCEPTED` column:
```
NAME   CONTROLLER            ACCEPTED   AGE   DESCRIPTION
alb    gateway.k8s.aws/alb   True       30d
```

# Gateway
The Gateway would be the listeners on the load balancer. Here's a basic one with TLS termination and the usual ports 80 and 433:
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eks-gateway
  namespace: default
spec:
  gatewayClassName: alb
  infrastructure:
    parametersRef:
      group: gateway.k8s.aws
      kind: LoadBalancerConfiguration
      name: alb-config
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    port: 443
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: All
    tls:
      mode: Terminate
      options:
        aws-load-balancer-ssl-cert: <ARN_OF_CERT>
```

# HTTPRoute
This would be what the Ingress object used to be, and this would create a listener rule that routes everything matching a certain host to a single Service. Note that you can use the existing Service that your Ingress is using. You do not need to create a new one for this.
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httproute
spec:
  parentRefs:
  - name: eks-gateway
    namespace: default
  hostnames:
  - <YOUR_URL>
  rules:
  - matches:
    - path: { type: PathPrefix, value: "/" }
    backendRefs:
    - name: <YOUR_K8S_SERVICE_NAME>
      port: 3000
```

If you would like to configure the target group, especially for health checks, then create a TargetGroupConfiguration:
```
apiVersion: gateway.k8s.aws/v1beta1
kind: TargetGroupConfiguration
metadata:
  name: tg-config
spec:
  targetReference:
    name: <YOUR_K8S_SERVICE_NAME>
  defaultConfiguration:
    healthCheckConfig:
      healthCheckPath: /health_check
```

## That's It
And that's it! Now you just need to swap the Route 53 target to the new load balancer, and it should work. Do check that your target group nodes are healthy first though. Migrating isn't actually that difficult - the main difficulty would be piecing together the fragmented documentation, so I hope this helps.

One last caveat. You might have noticed that each Gateway and HTTPRoute would create a new listener and rule on the load balancer. Do take note that there are [limits](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html). They are adjustable, but I have not gone through the process of doing that, so I have no idea how easy it is and how much they would be able to give.
