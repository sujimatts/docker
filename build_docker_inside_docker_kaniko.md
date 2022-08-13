# What is Kaniko

kaniko is a tool to build container images from a Dockerfile, inside a container or Kubernetes cluster.
kaniko solves two problems with using the Docker-in-Docker build method:

  - Docker-in-Docker requires privileged mode to function, which is a significant security concern.
  - Docker-in-Docker generally incurs a performance penalty and can be quite slow.

Kaniko is a tool to build container images from a Dockerfile. Unlike Docker, Kaniko doesn't require the Docker daemon.

Since there's no dependency on the daemon process, this can be run in any environment where the user doesn't have root access like a Kubernetes cluster.

Kaniko executes each command within the Dockerfile completely in the userspace using an executor image: gcr.io/kaniko-project/executor which runs inside a container; for instance, a Kubernetes pod. It executes each command inside the Dockerfile in order and takes a snapshot of the file system after each command.

If there are changes to the file system, the executor takes a snapshot of the filesystem change as a “diff” layer and updates the image metadata.

## Building a Docker image with kaniko
When building an image with kaniko and GitLab CI/CD, you should be aware of a few important details:

The kaniko debug image is recommended ```(gcr.io/kaniko-project/executor:debug)``` because it has a shell, and a shell is required for an image to be used with GitLab CI/CD.
The entrypoint needs to be overridden, otherwise the build script doesn’t run.
A Docker ```config.json``` file needs to be created with the authentication information for the desired container registry.

In the following example, kaniko is used to:

  1. Build a Docker image.
  2. Then push it to GitLab Container Registry.
  
The job runs only when a tag is pushed. A ```config.json``` file is created under ```/kaniko/.docker``` with the needed GitLab Container Registry credentials taken from the predefined CI/CD variables GitLab CI/CD provides.

In the last step, kaniko uses the ```Dockerfile``` under the root directory of the project, builds the Docker image and pushes it to the project’s Container Registry while tagging it with the Git tag:

```
build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"${CI_REGISTRY}\":{\"auth\":\"$(printf "%s:%s" "${CI_REGISTRY_USER}" "${CI_REGISTRY_PASSWORD}" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json
    - >-
      /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}"
  rules:
    - if: $CI_COMMIT_TAG
```

If you authenticate against the Dependency Proxy, you must add the corresponding CI/CD variables for authentication to the ```config.json``` file:


```
- echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "${CI_REGISTRY_USER}" "${CI_REGISTRY_PASSWORD}" | base64 | tr -d '\n')\"},\"$CI_DEPENDENCY_PROXY_SERVER\":{\"auth\":\"$(printf "%s:%s" ${CI_DEPENDENCY_PROXY_USER} "${CI_DEPENDENCY_PROXY_PASSWORD}" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json

```

READ: https://docs.gitlab.com/ee/ci/docker/using_kaniko.html
