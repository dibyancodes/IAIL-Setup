Sure, here's a comprehensive `.gitlab-ci.yml` file to automate the deployment of the Python Flask application to an EKS cluster using Helm. This CI/CD pipeline includes stages for building Docker images, pushing them to the GitLab Container Registry, and deploying to the EKS cluster.

### `.gitlab-ci.yml`

```yaml
stages:
  - build
  - push
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_HOST: tcp://localhost:2375/
  EKS_CLUSTER_NAME: flask-app-cluster
  AWS_DEFAULT_REGION: us-west-2
  HELM_RELEASE_NAME: flask-app

# Step 1: Build Docker Images
build_ui:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE/ui-service:latest ./ui
  artifacts:
    paths:
      - ./ui/Dockerfile

build_api:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE/api-service:latest ./api
  artifacts:
    paths:
      - ./api/Dockerfile

# Step 2: Push Docker Images to GitLab Registry
push_ui:
  stage: push
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE/ui-service:latest

push_api:
  stage: push
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE/api-service:latest

# Step 3: Deploy to EKS Cluster
deploy:
  stage: deploy
  image: lachlanevenson/k8s-helm:latest
  script:
    - apk add --no-cache curl
    - curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    - unzip awscliv2.zip
    - ./aws/install

    - aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_DEFAULT_REGION

    - helm upgrade --install $HELM_RELEASE_NAME ./helm/flask-app --set ui.image=$CI_REGISTRY_IMAGE/ui-service:latest --set api.image=$CI_REGISTRY_IMAGE/api-service:latest
```

### Explanation

1. **Stages**: Defines the stages of the pipeline - build, push, and deploy.
2. **Variables**: Sets up necessary environment variables like `DOCKER_DRIVER`, `DOCKER_HOST`, `EKS_CLUSTER_NAME`, `AWS_DEFAULT_REGION`, and `HELM_RELEASE_NAME`.

#### Build Stage

- **build_ui**: 
  - Uses Docker to build the UI service Docker image.
  - Uses `docker:dind` service to enable Docker-in-Docker.
  - Artifacts save the Dockerfile for later stages.

- **build_api**:
  - Similar to `build_ui`, but builds the API service Docker image.

#### Push Stage

- **push_ui**:
  - Logs in to the GitLab Container Registry.
  - Pushes the UI service Docker image to the registry.

- **push_api**:
  - Similar to `push_ui`, but pushes the API service Docker image.

#### Deploy Stage

- **deploy**:
  - Uses a Helm image for deployment.
  - Installs AWS CLI to interact with AWS services.
  - Configures kubectl to use the EKS cluster.
  - Uses Helm to deploy or update the application in the EKS cluster with the latest Docker images.

This `.gitlab-ci.yml` file automates the process of building, pushing, and deploying the Docker images for the Flask application to an EKS cluster using Helm, ensuring a seamless CI/CD pipeline.
