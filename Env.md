### 1. General Environment Variables

**DOCKER_DRIVER**
- **Used in**: `.gitlab-ci.yml`
- **Value**: `overlay2`
- **Purpose**: Specifies the storage driver for Docker. The `overlay2` driver is recommended for modern Linux distributions.
- **Where Used**: Throughout the Docker stages in the CI pipeline.

**DOCKER_HOST**
- **Used in**: `.gitlab-ci.yml`
- **Value**: `tcp://localhost:2375/`
- **Purpose**: Defines the Docker daemon socket to connect to. It's necessary for Docker-in-Docker (`dind`) to work.
- **Where Used**: Throughout the Docker stages in the CI pipeline.

**EKS_CLUSTER_NAME**
- **Used in**: `.gitlab-ci.yml`
- **Value**: `flask-app-cluster`
- **Purpose**: The name of the EKS cluster. Used to configure `kubectl` to interact with the correct Kubernetes cluster.
- **Where Used**: In the deploy stage for `aws eks update-kubeconfig`.

**AWS_DEFAULT_REGION**
- **Used in**: `.gitlab-ci.yml`
- **Value**: `us-west-2`
- **Purpose**: Specifies the AWS region where the EKS cluster is hosted.
- **Where Used**: In the deploy stage for `aws eks update-kubeconfig`.

**HELM_RELEASE_NAME**
- **Used in**: `.gitlab-ci.yml`
- **Value**: `flask-app`
- **Purpose**: Specifies the release name for Helm. It is used to identify the Helm release within the EKS cluster.
- **Where Used**: In the deploy stage for `helm upgrade --install`.

### 2. GitLab CI/CD Environment Variables

**CI_REGISTRY**
- **Used in**: `.gitlab-ci.yml`
- **Value**: Automatically set by GitLab
- **Purpose**: The URL of the GitLab Container Registry.
- **Where Used**: In the push stage for `docker login` and `docker push`.

**CI_REGISTRY_IMAGE**
- **Used in**: `.gitlab-ci.yml`
- **Value**: Automatically set by GitLab
- **Purpose**: The full path of the Docker image in the GitLab Container Registry.
- **Where Used**: Throughout the CI pipeline for tagging and pushing Docker images.

**CI_REGISTRY_USER**
- **Used in**: `.gitlab-ci.yml`
- **Value**: Automatically set by GitLab
- **Purpose**: The GitLab CI/CD username for Docker registry authentication.
- **Where Used**: In the push stage for `docker login`.

**CI_REGISTRY_PASSWORD**
- **Used in**: `.gitlab-ci.yml`
- **Value**: Automatically set by GitLab
- **Purpose**: The GitLab CI/CD password/token for Docker registry authentication.
- **Where Used**: In the push stage for `docker login`.

### 3. Kubernetes ConfigMaps and Secrets

**UI_SERVICE_PORT**
- **Used in**: Kubernetes ConfigMap (`/k8s/configmap.yaml` and `/helm/flask-app/templates/configmap.yaml`)
- **Value**: `"5000"`
- **Purpose**: Configures the port for the UI service to run on.
- **Where Used**: In the Flask UI service's deployment and Helm values.

**API_SERVICE_PORT**
- **Used in**: Kubernetes ConfigMap (`/k8s/configmap.yaml` and `/helm/flask-app/templates/configmap.yaml`)
- **Value**: `"5001"`
- **Purpose**: Configures the port for the API service to run on.
- **Where Used**: In the Flask API service's deployment and Helm values.

**DATABASE_PASSWORD**
- **Used in**: Kubernetes Secret (`/k8s/secrets.yaml` and `/helm/flask-app/templates/secrets.yaml`)
- **Value**: `<base64-encoded-password>`
- **Purpose**: Stores sensitive information securely, such as the database password.
- **Where Used**: In the Flask application if it requires a database connection (not explicitly shown in the Flask code but typically used in real applications).

### Summary

- **General Environment Variables**: `DOCKER_DRIVER`, `DOCKER_HOST`, `EKS_CLUSTER_NAME`, `AWS_DEFAULT_REGION`, `HELM_RELEASE_NAME` are used to configure Docker, AWS, and Helm settings throughout the CI/CD pipeline.
- **GitLab CI/CD Environment Variables**: `CI_REGISTRY`, `CI_REGISTRY_IMAGE`, `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD` are automatically set by GitLab to facilitate Docker image pushing and registry authentication.
- **Kubernetes ConfigMaps and Secrets**: `UI_SERVICE_PORT`, `API_SERVICE_PORT`, `DATABASE_PASSWORD` are used to configure service ports and securely manage sensitive information within the Kubernetes cluster.
