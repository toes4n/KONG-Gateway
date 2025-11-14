# KONG-Gateway
Kong Gateway

# Kong Ingress Controller: Your Gateway to Kubernetes Traffic Management
Kong Ingress Controller (KIC) transforms how you manage ingress traffic in Kubernetes clusters by combining the power of Kong Gateway with native Kubernetes resources. It's a production-ready solution that supports both traditional Ingress and modern Gateway API resources.​

# Why Kong Ingress Controller?
Kong Ingress Controller acts as a bridge between Kubernetes and Kong Gateway, translating Kubernetes resources into Kong configurations using CRDs (Custom Resource Definitions). Key benefits include:​

Multi-Protocol Support: Native handling of TCP, UDP, TLS, gRPC, HTTP/HTTPS traffic​

Advanced Traffic Management: Load balancing, health checking, and automatic scaling across gateway replicas​

Rich Plugin Ecosystem: Authentication, rate limiting, request/response transformations, and 50+ plugins​

Centralized Control: Single entry point for managing external traffic with improved observability and security​

# How KONG Ingress Controller work:


<img width="902" height="303" alt="image" src="https://github.com/user-attachments/assets/4ac48824-58ff-4ca3-b8ce-1e156ce209c1" />


<img width="986" height="393" alt="image" src="https://github.com/user-attachments/assets/b3da01f9-9b36-4255-ad16-790cd283d4b3" />

# Installation Guide
Prerequisites

Kubernetes cluster (Minikube, K3s, Kind, GKE, EKS, or ACK)

Helm 3.x installed

kubectl configured with cluster access​

# Step 1: Install Gateway API CRDs

    kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
    
# Step 2: Install Kong Ingress Controller

    helm repo add kong https://charts.konghq.com
    helm repo update
    helm install kong --namespace kong --create-namespace --repo https://charts.konghq.com ingress
    
# Step 3: Verify Installation

    kubectl get pods -n kong
    kubectl get svc -n kong
    
Example: Deploying an Echo Application

Deploy the Echo Service

    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Service
    metadata:
      name: echo
      labels:
        app: echo
    spec:
      ports:
      - port: 1027
        name: tcp
        protocol: TCP
        targetPort: 1027
      selector:
        app: echo
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: echo
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: echo
      template:
        metadata:
          labels:
            app: echo
        spec:
          containers:
          - name: echo
            image: gcr.io/kubernetes-e2e-test-images/echoserver:2.2
            ports:
            - containerPort: 1027
            env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
    EOF

# Create Ingress Resource

    kubectl apply -f - <<EOF
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: echo
      annotations:
        konghq.com/strip-path: 'true'
    spec:
      ingressClassName: kong
      rules:
      - host: echo.example.com
        http:
          paths:
          - path: /echo
            pathType: ImplementationSpecific
            backend:
              service:
                name: echo
                port:
                  number: 1027
    EOF

# Test Your Deployment

# Get Kong Proxy service external IP
export PROXY_IP=$(kubectl get svc -n kong kong-gateway-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test the endpoint
curl -H "Host: echo.example.com" http://$PROXY_IP/echo
Alternative: Using HTTPRoute (Gateway API)

# For modern Gateway API approach:​

    kubectl apply -f - <<EOF
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: echo-route
      annotations:
        konghq.com/strip-path: 'true'
    spec:
      parentRefs:
      - name: kong
        namespace: kong
      rules:
      - matches:
        - path:
            type: PathPrefix
            value: /echo
        backendRefs:
        - name: echo
          port: 1027
    EOF

# Adding Plugins
Enable rate limiting on your service:​

    kubectl annotate service echo konghq.com/plugins=rate-limiting-plugin

    kubectl apply -f - <<EOF
    apiVersion: configuration.konghq.com/v1
    kind: KongPlugin
    metadata:
      name: rate-limiting-plugin
    config:
      minute: 100
      policy: local
    plugin: rate-limiting
    EOF
