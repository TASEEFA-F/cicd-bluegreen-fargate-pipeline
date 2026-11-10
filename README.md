# CI/CD Pipeline with Blue-Green Deployment to AWS Fargate

I built a fully managed CI/CD pipeline that releases a containerised web application to Amazon ECS on AWS Fargate. A code change pushed to the repository is built into a new Docker image, stored in Amazon ECR and rolled out with a **blue-green deployment**, so the running site is never taken down during a release.

Completed as a hands-on lab in the **AWS re/Start** program with **ICT Academy** (AWS lab: *Create a CI/CD pipeline to deploy your app to AWS Fargate*).

## The result

The application is an "AWS News" web page running as three Fargate tasks behind an Application Load Balancer. Its background was dark grey (`#282F3D`). I changed it to green (`#1D8102`) and released the change through the pipeline. After CodeDeploy switched traffic, refreshing the load balancer URL showed the **green** version while the old tasks were still standing by for rollback.

```css
/* static/css/app.css */
- background-color: #282F3D;   /* blue environment (old) */
+ background-color: #1D8102;   /* green environment (new) */
```

## Architecture

```
Code push to AWS CodeCommit (repo: pipeline-source-code, branch: main)
        |
        v
AWS CodePipeline (AwsNewsPipeline)
  1. Source   CodeCommit
  2. Build    AWS CodeBuild (awsnews-build-project)
                - docker build
                - tag image with the commit ID
                - push to Amazon ECR (tag immutability on)
                - write imageDetail.json
  3. Deploy   Amazon ECS (Blue/Green) via AWS CodeDeploy
                (awsnews-application / awsnews-deployment-group)
        |
        v
Application Load Balancer (FargateALB), listener HTTP 80
   |                              |
   v                              v
BlueTargetGroup               GreenTargetGroup
(original task set)           (replacement task set)
   \                              /
    ECS service FargateService on cluster FargateCluster (AWS Fargate)
```

## AWS services used

| Service | What it does in this project |
| --- | --- |
| AWS CodeCommit | Git repository that holds the application and pipeline files |
| AWS CodePipeline | Runs Source, Build and Deploy automatically on every push |
| AWS CodeBuild | Builds the Docker image and pushes it to ECR |
| Amazon ECR | Private image registry, with image tag immutability turned on |
| Amazon ECS on AWS Fargate | Runs the application containers without servers to manage |
| AWS CodeDeploy | Creates the green task set and reroutes load balancer traffic |
| Elastic Load Balancing | Application Load Balancer with blue and green target groups |
| Amazon CloudWatch Logs | Build and container logs (log group `cicd-logs`) |
| AWS IAM | Service roles for CodePipeline, CodeBuild, CodeDeploy and ECS tasks |

## Files in this repository

| File | Purpose |
| --- | --- |
| `buildspec.yaml` | CodeBuild instructions: log in to ECR, build and tag the image with the commit ID, push it, and write `imageDetail.json` for the deploy stage |
| `appspec.yaml` | Tells CodeDeploy which container (`application`) and port (80) to switch. `<TASK_DEFINITION>` is filled in at deployment time |
| `taskdef.json` | Task definition for the green environment. `<IMAGE_NAME>` is replaced with the new image URI by the pipeline |

The application source code (Node.js app, Dockerfile and static files) was provided by the AWS lab and is not included here, as it belongs to AWS. Placeholders such as `<ACCOUNT_ID>` and `<AWS_REGION>` replace the values from the lab account.

## What I did

**1. Reviewed the existing deployment**
- Opened the application through the load balancer URL.
- In Amazon ECS, checked that `FargateService` was running 3/3 tasks behind `FargateALB`, and that its deployment controller was `CODE_DEPLOY`, which blue-green on Fargate requires.
- Reviewed a task's launch type (Fargate) and network mode (`awsvpc`, giving each task its own network interface).
- Turned on **image tag immutability** in the ECR repository so an existing image tag can never be overwritten.

**2. Prepared the source repository**
- Wrote `buildspec.yaml`, `appspec.yaml` and `taskdef.json`.
- Changed the background colour in `static/css/app.css` from `#282F3D` to `#1D8102`.
- Created the CodeCommit repository `pipeline-source-code` and pushed the code to `main`.

**3. Created the CodeDeploy application and deployment group**
- Application `awsnews-application` (compute platform: Amazon ECS).
- Deployment group `awsnews-deployment-group`: cluster `FargateCluster`, service `FargateService`, load balancer `FargateALB` on HTTP 80, target groups `BlueTargetGroup` and `GreenTargetGroup`.
- Traffic rerouting: immediately. Original task set kept for 5 minutes so a rollback stays possible.

**4. Built the pipeline in CodePipeline (`AwsNewsPipeline`)**
- Source: CodeCommit, `pipeline-source-code`, branch `main`.
- Build: new CodeBuild project `awsnews-build-project` on the Amazon Linux standard image, with **privileged mode** on so it can run Docker, using `buildspec.yaml` and logging to `cicd-logs`.
- Deploy: Amazon ECS (Blue/Green) using the CodeDeploy application and group, with the task definition, AppSpec and image details all taken from the build artifact and the placeholder `IMAGE_NAME`.

**5. Watched the blue-green deployment**
- The build logged in to ECR, built the image, tagged it with the commit ID and pushed it.
- CodeDeploy then went through its four steps: create the new task definition and green task set, reroute traffic, keep the original tasks for testing, and terminate the original task set.
- During step 3, I refreshed the application: the background was green. I then chose **Terminate original task set**, and all lifecycle events showed Succeeded.

## Why blue-green

- The old version (blue) keeps serving visitors until the new version (green) is running and healthy.
- Traffic moves in one step at the load balancer, so there is no downtime.
- For a few minutes both versions exist, so going back is a quick rollback instead of a redeploy.
- Tagging every image with its commit ID, plus immutable tags in ECR, gives a clear record of exactly which code is running.

## What I learned

- A pipeline turns a release into a `git push`, so every release follows the same tested path.
- CodeBuild, CodeDeploy and ECS pass information between stages through small files (`imageDetail.json`, `taskdef.json`, `appspec.yaml`) with placeholders filled in at run time.
- Blue-green deployment depends on the load balancer having two target groups and the ECS service using the CodeDeploy deployment controller.
