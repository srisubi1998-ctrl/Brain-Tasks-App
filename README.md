# Brain Tasks App: Deployment on AWS EKS with CI/CD

This project deploys the React application **Brain Tasks App** to a production-ready state on AWS. The app is containerized with Docker, stored in Amazon ECR, run on Amazon EKS behind a Network Load Balancer, and deployed automatically through AWS CodePipeline and CodeBuild. Logs are tracked in Amazon CloudWatch.

Source application: https://github.com/Vennilavanguvi/Brain-Tasks-App

## Final output

| Item | Value |
|---|---|
| Region | us-east-1 (N. Virginia) |
| Application URL | http://afe5ed353844d4fdba41328d1b7f611e-4a1ded5041c2a1c7.elb.us-east-1.amazonaws.com |
| Load Balancer ARN | `arn:aws:elasticloadbalancing:us-east-1:196390795947:loadbalancer/net/afe5ed353844d4fdba41328d1b7f611e/4a1ded5041c2a1c7` |
| ECR repository | `196390795947.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app` |
| EKS cluster | `brain-tasks-cluster` |
| CodeBuild project | `brain-tasks-build` |
| CodePipeline | `brain-tasks-pipeline` |

![Application running through the load balancer](screenshots/app-via-loadbalancer.png)

## Architecture

```
GitHub (main) --> CodePipeline --> CodeBuild --> Amazon ECR
                       |               |
                       |               +--> docker build + push, update manifest
                       v
                  Deploy stage (Amazon EKS) --> EKS cluster --> Network Load Balancer --> Users
                                                                   
CloudWatch Logs: CodeBuild logs, pipeline history, application (pod) logs
```

## Repository contents

| File | Purpose |
|---|---|
| `dist/` | Built React application |
| `Dockerfile` | Serves the app with nginx on port 3000 |
| `nginx.conf` | nginx configuration (listens on 3000, single-page app routing) |
| `k8s/deployment.yaml` | Kubernetes Deployment (2 replicas, container port 3000) |
| `k8s/service.yaml` | Kubernetes Service of type LoadBalancer (NLB, port 80 to 3000) |
| `buildspec.yml` | CodeBuild instructions: build, push, deploy |

## 1. EC2 work server

An Ubuntu Server 22.04 LTS `t3.medium` instance with a 20 GiB disk was used to run Docker, AWS CLI, kubectl and eksctl.

- Security group: SSH (22) from my IP, custom TCP 3000 for testing
- IAM role `brain-tasks-ec2-role` attached to the instance, so no access keys are needed

![EC2 AMI and instance type](screenshots/ec2-ami-and-instance-type.png)
![EC2 storage and network](screenshots/ec2-storage-network.png)
![EC2 security group](screenshots/ec2-security-group.png)
![IAM role for EC2](screenshots/iam-role-ec2.png)
![EC2 instance running](screenshots/ec2-instance-running.png)
![SSH connected](screenshots/ec2-ssh-connected.png)

Tools installed: Docker, AWS CLI v2, kubectl, eksctl.

![Tool versions](screenshots/tools-installed-versions.png)

## 2. Dockerize and test locally

```bash
git clone https://github.com/Vennilavanguvi/Brain-Tasks-App.git
cd Brain-Tasks-App
docker build -t brain-tasks-app:v1 .
docker run -d -p 3000:3000 --name brain-test brain-tasks-app:v1
```

`Dockerfile`:
```dockerfile
FROM public.ecr.aws/nginx/nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY dist/ /usr/share/nginx/html/
EXPOSE 3000
CMD ["nginx", "-g", "daemon off;"]
```

![git clone and ls](screenshots/git-clone-and-ls.png)
![Dockerfile](screenshots/dockerfile.png)
![Docker build success](screenshots/docker-build-success.png)
![docker ps](screenshots/docker-ps.png)
![App on port 3000](screenshots/app-running-on-port-3000.png)

## 3. Amazon ECR

```bash
aws ecr create-repository --repository-name brain-tasks-app --region us-east-1
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com
docker tag brain-tasks-app:v1 <account-id>.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app:v1
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app:v1
```

![ECR docker push](screenshots/ecr-docker-push.png)
![ECR repository image](screenshots/ecr-repository-image.png)

## 4. Amazon EKS cluster

```bash
eksctl create cluster --name brain-tasks-cluster --region us-east-1 \
  --zones us-east-1a,us-east-1b --nodegroup-name brain-nodes \
  --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 3 --managed
```

![eksctl cluster created](screenshots/eksctl-cluster-created.png)
![EKS cluster active](screenshots/eks-cluster-active.png)
![kubectl get nodes](screenshots/kubectl-get-nodes.png)

## 5. Kubernetes manifests

- `k8s/deployment.yaml`: runs 2 replicas of the image from ECR, with a readiness probe on port 3000.
- `k8s/service.yaml`: type `LoadBalancer` with the annotation `service.beta.kubernetes.io/aws-load-balancer-type: "nlb"`. This creates a Network Load Balancer (which has an ARN) that forwards port 80 to container port 3000.

![Kubernetes YAML files](screenshots/k8s-yaml-files.png)
![kubectl get pods](screenshots/kubectl-get-pods.png)
![kubectl get svc](screenshots/kubectl-get-svc.png)
![Load balancer ARN (CLI)](screenshots/loadbalancer-arn-cli.png)
![Load balancer in console](screenshots/loadbalancer-console.png)

## 6. Version control

The code is pushed to GitHub with CLI commands:

```bash
git remote add origin https://github.com/srisubi1998-ctrl/Brain-Tasks-App.git
git add .
git commit -m "Add Dockerfile, k8s manifests and buildspec"
git branch -M main
git push -u origin main
```

![git push](screenshots/git-push-cli.png)
![GitHub repository](screenshots/github-repo.png)
![buildspec in GitHub](screenshots/buildspec-in-github.png)

## 7. AWS CodeBuild

The project `brain-tasks-build` uses GitHub as the source and an Ubuntu managed image (`aws/codebuild/standard:8.0`) with **privileged mode** enabled (needed for Docker builds). Environment variables: `AWS_ACCOUNT_ID` and `AWS_DEFAULT_REGION`.

The service role has `AmazonEC2ContainerRegistryPowerUser`, an inline policy allowing `eks:DescribeCluster`, and an EKS access entry with `AmazonEKSClusterAdminPolicy`.

`buildspec.yml` explained:

| Phase | What it does |
|---|---|
| install | Reads the cluster version and installs a matching `kubectl` |
| pre_build | Logs in to ECR and sets the image tag from the commit ID |
| build | Builds the Docker image and tags it `latest` |
| post_build | Pushes the image to ECR, updates the image in `k8s/deployment.yaml`, runs `kubectl apply` and waits for the rollout |

![CodeBuild source and environment](screenshots/codebuild-source-environment.png)
![CodeBuild environment variables and buildspec](screenshots/codebuild-env-vars-buildspec.png)
![CodeBuild role permissions](screenshots/codebuild-role-permissions.png)
![CodeBuild build succeeded](screenshots/codebuild-build-succeeded.png)
![CodeBuild build logs](screenshots/codebuild-build-logs.png)

## 8. AWS CodePipeline

The pipeline `brain-tasks-pipeline` has three stages:

1. **Source**: GitHub (via GitHub App), branch `main`. Every push starts the pipeline.
2. **Build**: AWS CodeBuild project `brain-tasks-build`. Builds and pushes the image and produces the updated manifests as the build artifact.
3. **Deploy**: Amazon EKS action (kubectl) on cluster `brain-tasks-cluster`, using the manifests from the build artifact.

The pipeline service role also has an EKS access entry and `eks:DescribeCluster` permission.

![CodePipeline configuration](screenshots/codepipeline-configuration.png)
![All stages succeeded](screenshots/codepipeline-all-stages-succeeded.png)

**CI/CD test:** changing `replicas` from 2 to 3 and pushing to `main` started a new pipeline run automatically, and 3 pods were running afterwards.

![Pipeline triggered by push](screenshots/codepipeline-triggered-by-push.png)
![App after pipeline update](screenshots/app-after-pipeline-update.png)

## 9. Monitoring with CloudWatch

- **Build logs:** log group `/aws/codebuild/brain-tasks-build`.
- **Deploy tracking:** CodePipeline execution history.
- **Application logs:** the `amazon-cloudwatch-observability` EKS add-on sends pod logs to `/aws/containerinsights/brain-tasks-cluster/application`.

![CloudWatch CodeBuild logs](screenshots/cloudwatch-codebuild-logs.png)
![CodePipeline execution history](screenshots/codepipeline-execution-history.png)
![CloudWatch application logs](screenshots/cloudwatch-application-logs.png)

## Clean-up

```bash
kubectl delete -f k8s/
eksctl delete cluster --name brain-tasks-cluster --region us-east-1
aws ecr delete-repository --repository-name brain-tasks-app --force --region us-east-1
```
