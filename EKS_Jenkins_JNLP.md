# Enabling JNLP Connectivity for Jenkins Agents through AWS ALB Ingress Controller

To enable JNLP connectivity for remote Jenkins agents through an AWS ALB Ingress Controller on your EKS cluster, you'll need to configure a few specific elements:

## Solution Overview

1. Create a dedicated JNLP service port in Jenkins
2. Configure the Ingress resource to route JNLP traffic
3. Adjust network policies if necessary
4. Update agent configuration

## Implementation Steps

### 1. Update Your Jenkins Service

First, ensure your Jenkins Kubernetes Service exposes the JNLP port (typically 50000):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  selector:
    app: jenkins
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: jnlp
    port: 50000
    targetPort: 50000
  type: ClusterIP
```

### 2. Create a Separate Service for JNLP

It's often better to create a dedicated service for JNLP traffic:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-jnlp
  namespace: jenkins
spec:
  selector:
    app: jenkins
  ports:
  - name: jnlp
    port: 50000
    targetPort: 50000
  type: NodePort  # This allows the ALB to route traffic to this port
```

### 3. Configure ALB Ingress for Both HTTP(S) and TCP (JNLP)

The AWS ALB Ingress Controller needs special annotations to handle TCP traffic for JNLP:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins-ingress
  namespace: jenkins
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}, {"TCP": 50000}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": {"Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
    alb.ingress.kubernetes.io/backend-protocol: HTTP
    # Add SSL certificate ARN
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account-id:certificate/certificate-id
    # JNLP traffic handling
    alb.ingress.kubernetes.io/target-group-attributes: stickiness.enabled=true,stickiness.lb_cookie.duration_seconds=60
spec:
  rules:
  - host: jenkins.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              name: http
```

### 4. Create a Target Group Binding for JNLP

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: jenkins-jnlp
  namespace: jenkins
spec:
  targetGroupARN: arn:aws:elasticloadbalancing:region:account-id:targetgroup/jenkins-jnlp/abcdef123456
  targetType: ip
  serviceRef:
    name: jenkins-jnlp
    port: 50000
```

### 5. Configure Jenkins for JNLP Connectivity

In Jenkins UI:

1. Go to "Manage Jenkins" > "Configure Global Security"
2. Under "Agents", set "TCP port for inbound agents" to "Fixed: 50000"
3. Save the configuration

### 6. Update Agent Configuration

When configuring your remote agents:

1. Use the ALB DNS or your custom domain as the Jenkins URL
2. Set the JNLP port to 50000
3. Make sure the agents have the proper credentials to connect

## Alternative Approach Using NLB

If TCP/JNLP traffic handling through ALB becomes problematic, consider using a Network Load Balancer (NLB) specifically for JNLP traffic:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-jnlp
  namespace: jenkins
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
spec:
  selector:
    app: jenkins
  ports:
  - port: 50000
    targetPort: 50000
  type: LoadBalancer
```

## Security Considerations

1. Consider implementing security groups to restrict JNLP access
2. Use agent certificates for authentication when possible
3. Consider using a VPN connection for highly sensitive environments

Let me know if you need more specific configuration details for your particular setup!
