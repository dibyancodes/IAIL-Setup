Deploying a Python Flask application with UI and API services to an EKS cluster using Helm charts involves several steps. Below, I'll provide a detailed explanation and the necessary code snippets.

### 1. Setting Up AWS EC2
First, you need to set up an EC2 instance for initial configuration and setup.

1. **Launch an EC2 Instance**: Launch an EC2 instance from the AWS Management Console. Ensure it has sufficient permissions to manage EKS and other required AWS services.

2. **Install Required Tools**:
    ```bash
    # Update package list
    sudo apt-get update

    # Install Docker
    sudo apt-get install -y docker.io

    # Start Docker service
    sudo systemctl start docker
    sudo systemctl enable docker

    # Install kubectl
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

    # Install AWS CLI
    sudo apt-get install -y awscli

    # Install eksctl
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_$(uname -m).tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin

    # Install Helm
    curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
    ```

### 2. Creating the Python Flask App

Let's create a simple Python Flask application with UI and API services.

**File Structure**:
```
/flask-app
    /ui
        app.py
        Dockerfile
    /api
        app.py
        Dockerfile
    requirements.txt
```

**`requirements.txt`**:
```plaintext
Flask==2.0.1
requests==2.25.1
```

**UI Service** (`/flask-app/ui/app.py`):
```python
# app.py for UI Service
from flask import Flask, render_template
import requests

app = Flask(__name__)

@app.route('/')
def home():
    return "Welcome to the Flask UI Service!"

@app.route('/api-data')
def api_data():
    # Call the API service
    response = requests.get('http://api-service:5001/data')
    return f"API Data: {response.text}"

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000)
```

**Dockerfile for UI Service** (`/flask-app/ui/Dockerfile`):
```dockerfile
# Use the official Python image from the Docker Hub
FROM python:3.9-slim

# Set the working directory
WORKDIR /app

# Copy the requirements file into the container
COPY ../requirements.txt .

# Install the required packages
RUN pip install -r requirements.txt

# Copy the current directory contents into the container at /app
COPY . .

# Make port 5000 available to the world outside this container
EXPOSE 5000

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

**API Service** (`/flask-app/api/app.py`):
```python
# app.py for API Service
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/data')
def data():
    return jsonify({"message": "Hello from API Service!"})

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5001)
```

**Dockerfile for API Service** (`/flask-app/api/Dockerfile`):
```dockerfile
# Use the official Python image from the Docker Hub
FROM python:3.9-slim

# Set the working directory
WORKDIR /app

# Copy the requirements file into the container
COPY ../requirements.txt .

# Install the required packages
RUN pip install -r requirements.txt

# Copy the current directory contents into the container at /app
COPY . .

# Make port 5001 available to the world outside this container
EXPOSE 5001

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

### 3. Building and Pushing Docker Images to GitLab Registry

Assuming you have set up GitLab's Container Registry:

```bash
# Navigate to the UI directory and build the Docker image
cd /flask-app/ui
docker build -t registry.gitlab.com/yourusername/yourproject/ui-service:latest .

# Navigate to the API directory and build the Docker image
cd /flask-app/api
docker build -t registry.gitlab.com/yourusername/yourproject/api-service:latest .

# Login to GitLab Registry
docker login registry.gitlab.com

# Push the Docker images
docker push registry.gitlab.com/yourusername/yourproject/ui-service:latest
docker push registry.gitlab.com/yourusername/yourproject/api-service:latest
```

### 4. Setting Up EKS Cluster

Create an EKS cluster using `eksctl`:

```bash
# Create EKS Cluster
eksctl create cluster --name flask-app-cluster --region us-west-2 --nodegroup-name flask-app-nodes --node-type t2.micro --nodes 2 --nodes-min 1 --nodes-max 3
```

### 5. Kubernetes YAML Files

**Deployment for UI Service** (`/k8s/ui-deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ui
  template:
    metadata:
      labels:
        app: ui
    spec:
      containers:
      - name: ui-container
        image: registry.gitlab.com/yourusername/yourproject/ui-service:latest
        ports:
        - containerPort: 5000
```

**Deployment for API Service** (`/k8s/api-deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api-container
        image: registry.gitlab.com/yourusername/yourproject/api-service:latest
        ports:
        - containerPort: 5001
```

**Service for UI** (`/k8s/ui-service.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui-service
spec:
  selector:
    app: ui
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: LoadBalancer
```

**Service for API** (`/k8s/api-service.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
    - protocol: TCP
      port: 5001
      targetPort: 5001
```

**Ingress** (`/k8s/ingress.yaml`):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flask-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ui-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 5001
```

**ConfigMaps** (`/k8s/configmap.yaml`):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flask-app-config
data:
  UI_SERVICE_PORT: "5000"
  API_SERVICE_PORT: "5001"
```

**Secrets** (`/k8s/secrets.yaml`):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: flask-app-secrets
type: Opaque
data:
  DATABASE_PASSWORD: <base64-encoded-password>
```

### 6. Helm Chart

**File Structure**:
```
/helm
    /flask-app
        Chart.yaml
        values.yaml
        templates
            deployment.yaml
            service.yaml
            ingress.yaml
            configmap.yaml
            secrets.yaml
```

**Chart.yaml**:
```yaml
apiVersion: v2
name: flask-app
description: A Helm chart for Flask App with UI and API services
version: 0.1.0
appVersion: "1.0"
```

**values.yaml**:
```yaml
ui:
  image: registry.gitlab.com/yourusername/yourproject/ui-service:latest
  replicas: 2
  service:
    type: LoadBalancer
    port: 80

api:
  image: registry.gitlab.com/yourusername/yourproject/api-service:latest
  replicas: 2
  service:
    port: 5001

ingress:
  enabled: true
  annotations:
    nginx.ingress.kubernetes.io/re

write-target: /
  hosts:
    - host: yourdomain.com
      paths:
        - path: /
          pathType: Prefix
        - path: /api
          pathType: Prefix

config:
  UI_SERVICE_PORT: "5000"
  API_SERVICE_PORT: "5001"

secrets:
  DATABASE_PASSWORD: <base64-encoded-password>
```

**Deployment Template** (`/helm/flask-app/templates/deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-deployment
spec:
  replicas: {{ .Values.ui.replicas }}
  selector:
    matchLabels:
      app: ui
  template:
    metadata:
      labels:
        app: ui
    spec:
      containers:
      - name: ui-container
        image: {{ .Values.ui.image }}
        ports:
        - containerPort: {{ .Values.config.UI_SERVICE_PORT | int }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: {{ .Values.api.replicas }}
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api-container
        image: {{ .Values.api.image }}
        ports:
        - containerPort: {{ .Values.config.API_SERVICE_PORT | int }}
```

**Service Template** (`/helm/flask-app/templates/service.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui-service
spec:
  selector:
    app: ui
  ports:
    - protocol: TCP
      port: {{ .Values.ui.service.port }}
      targetPort: {{ .Values.config.UI_SERVICE_PORT | int }}
  type: {{ .Values.ui.service.type }}
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
    - protocol: TCP
      port: {{ .Values.api.service.port }}
      targetPort: {{ .Values.config.API_SERVICE_PORT | int }}
```

**Ingress Template** (`/helm/flask-app/templates/ingress.yaml`):
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flask-app-ingress
  annotations:
{{ toYaml .Values.ingress.annotations | indent 4 }}
spec:
  rules:
  {{- range .Values.ingress.hosts }}
    - host: {{ .host }}
      http:
        paths:
        {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ ternary "ui-service" "api-service" (eq .path "/api") }}
                port:
                  number: {{ ternary "80" "5001" (eq .path "/api") }}
        {{- end }}
  {{- end }}
{{- end }}
```

**ConfigMap Template** (`/helm/flask-app/templates/configmap.yaml`):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flask-app-config
data:
  UI_SERVICE_PORT: {{ .Values.config.UI_SERVICE_PORT }}
  API_SERVICE_PORT: {{ .Values.config.API_SERVICE_PORT }}
```

**Secrets Template** (`/helm/flask-app/templates/secrets.yaml`):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: flask-app-secrets
type: Opaque
data:
  DATABASE_PASSWORD: {{ .Values.secrets.DATABASE_PASSWORD }}
```

### 7. Deploying with Helm

Navigate to the `helm/flask-app` directory and run:

```bash
helm install flask-app . --values values.yaml
```

### Explanation of Environment Variables

1. **UI_SERVICE_PORT & API_SERVICE_PORT**: These environment variables are used to configure the ports on which the UI and API services will run.
2. **DATABASE_PASSWORD**: This secret is used for storing sensitive information securely and can be accessed by the application as needed.
