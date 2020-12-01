# Docker Pull Limits

Docker pull limits can cause disruption to the application build and deploy process when container images are needed. In the coming weeks AWS plans to release a public container image registry that can be used in place of the more common docker public registry avoiding this issue altogether. In the interim these pull limits can be avoided by using GitHub Actions as the build mechanism, pushing the resulting images to a private registry.

## Types of container registries

### Public

- Docker Hub - docker.io
  - This document will use “docker” to mean Docker Hub (docker.io) when in the context of a container registry.
- AWS public registry
  - Will be available “within weeks”.

### Private (container registry owned by BC Gov)

- Shared service
  - Artifactory, OpenShift, others?
  - AWS ECR - LZ scoped.
    - Access provided to all accounts in a LZ.
- Team/Account
  - ECR scoped to an AWS account in an LZ.

## Types of container images

### Base

- A base image is an image commonly used to build application specific images.
- These can broadly be put into three categories:
  - OS
    - e.g. alpine, ubuntu, centos.
  - Language
    - e.g. node, python, php.
  - Application
    - e.g. mysql, mongo, nginx.

### Custom

- An image that has been built specifically for a particular application.
- These are often built on top of base images from public registries, most commonly docker.

## Docker Hub pull limits

Docker has implemented pull limits from it’s public container registry [[1](https://docs.docker.com/docker-hub/download-rate-limit/)], these limits are:

- Unauthenticated
  - 100 pulls per six hours per IP address.
- Authenticated
  - 200 pulls per six hours per free account.
  - Paid plans are available to enable unlimited pulls on the account.

**Update (2020/11/16) [[2](https://www.docker.com/increase-rate-limits)]:**

Docker seems to have increased the limit but this is not reflected on all of their public documentation at this time.

- Unauthenticated
  - 250 pulls per six hours per IP address.
- Authenticated
  - 250 pulls per six hours per free account.
  - Paid plans are available to enable more pulls on the account.

### Impact

Many container images are built or deployed by systems that pull from docker while unauthenticated. If the system is a shared service such as AWS CodePipeline, CodeBuild, ECS/Fargate, or even the BC Gov OpenShift then it is possible that the IP address pool used by that service will rapidly consume all of the allowed unauthenticated docker pulls. **This will cause builds and deploys to fail if not addressed.**

### Mitigation

New public container registries are likely to appear. AWS for example has stated its intention to create one. However, until a new standard has been established by the community it is wise to create an “in house” container registry strategy. Several of the possible mitigation strategies are:

- Create docker accounts for some defined unit (application, team, etc) and configure the build system to authenticate on docker.io.
  - Paid plans for unlimited pulls are $5/month USD per account
  - Pros
    - Free accounts can be used. It is unlikely that any unit defined in this way will consume more than 200 pulls in six hours.
  - Cons
    - This is a pattern that must be repeated for every new and existing system that uses the docker public registry.
    - Creates another account to manage.
    - Requires an email address per account.
- Use GItHub Actions to build images and push to the accounts private registry. GitHub has reached a deal with docker to avoid the pull limits [[3](https://github.com/actions/virtual-environments/issues/1445#issuecomment-729675520)].
  - Pros
    - Low effort
    - Consistent with existing build and release workflows.
  - Cons
    - Only works when using GitHub actions.
- Mirror base images that are used into a private registry and configure the Dockerfiles to reference the base images in the private registry.
  - Pros
    - One time effort.
  - Cons
    - Creates a centralized system to maintain.
    - Developers may incorrectly use the docker registry. If this is not discovered in code review this can lead to build and deploy failures.
- Run a docker registry mirror [[4](https://docs.docker.com/registry/recipes/mirror/)][[5](https://about.gitlab.com/blog/2020/10/30/mitigating-the-impact-of-docker-hub-pull-requests-limits/)].
  - Pros
    - Works with any build system.
  - Cons
    - Creates a centralized system to maintain.
- Use the AWS public registry when it is available “within weeks” [[2](https://aws.amazon.com/blogs/containers/advice-for-customers-dealing-with-docker-hub-rate-limits-and-a-coming-soon-announcement/)].
  - Pros
    - Low effort.
  - Cons
    - Not yet available. **Update (2020/12/1): AWS has released the public registry** [[6](https://aws.amazon.com/blogs/aws/amazon-ecr-public-a-new-public-container-registry)]
    - Developers may incorrectly use the docker registry. If this is not discovered in code review this can lead to build and deploy failures.

## Recommendation

Use GitHub Actions as the primary build mechanism. This avoids the docker pull limits while building applications in GitHub Actions and the resulting images can be pushed directly to a private container registry (e.g. AWS ECR) in the application’s account. The application deployment configuration (e.g. ECS/Fargate) would reference the resulting image in AWS ECR.

## References

1. [Docker announcement on rate limits](https://docs.docker.com/docker-hub/download-rate-limit/)
2. [AWS advice for customers regarding docker rate limits](https://aws.amazon.com/blogs/containers/advice-for-customers-dealing-with-docker-hub-rate-limits-and-a-coming-soon-announcement/)
3. [GitHub issue indicating that GitHub has struck a deal to prevent pull limits on GitHub Actions](https://github.com/actions/virtual-environments/issues/1445#issuecomment-729675520)
4. [Docker registry mirrors](https://docs.docker.com/registry/recipes/mirror/)
5. [Caching Docker images to reduce the number of calls to DockerHub from your CI/CD infrastructure](https://about.gitlab.com/blog/2020/10/30/mitigating-the-impact-of-docker-hub-pull-requests-limits/)
6. [AWS released their new public docker registry](https://aws.amazon.com/blogs/aws/amazon-ecr-public-a-new-public-container-registry)

### Other relevant links

- ECS/Fargate
  - [https://docs.aws.amazon.com/AmazonECS/latest/userguide/fargate-task-networking.html](https://docs.aws.amazon.com/AmazonECS/latest/userguide/fargate-task-networking.html)
  - [https://docs.amazonaws.cn/en_us/AmazonECS/latest/userguide/task_cannot_pull_image.html](https://docs.amazonaws.cn/en_us/AmazonECS/latest/userguide/task_cannot_pull_image.html)
  - [https://github.com/aws/containers-roadmap/issues/696](https://github.com/aws/containers-roadmap/issues/696)
- CodeBuild
  - [https://aws.amazon.com/blogs/devops/reducing-docker-image-build-time-on-aws-codebuild-using-an-external-cache/](https://aws.amazon.com/blogs/devops/reducing-docker-image-build-time-on-aws-codebuild-using-an-external-cache/)
