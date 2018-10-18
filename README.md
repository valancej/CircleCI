# Integrating Anchore scanning with CircleCI

## Introduction HERE

This will walkthrough integrating Anchore scanning into a CircleCI pipeline. During the first step, a Docker image will be built from a Dockerfile. Following this, during the second step Anchore will scan the image, and depending on the result of the policy evaluation, proceed to the final step. During the final step the built image will be pushed to a Docker registry.

## Prerequisites

- CircleCI Account: https://circleci.com/
- Anchore Engine Service: https://github.com/anchore/anchore-engine

## Setup

Prior to setting up your CircleCI build pipeline, an Anchore Engine service needs to be accessible from the pipeline. Typically this is on port 8228. In this example, I have an Anchore Engine service on AWS EC2 with standard configuration. I also have a Dockerfile in a Github repository that I will build an image from during the first step of the pipeline. In the final step, I will be pushing the built image to an image repository in my personal Dockerhub.

The Github repository can be referenced here: https://github.com/valancej/CircleCI

Repository contents:

- .circleci directory (Contains config.yml needed to define CircleCI build)
- Dockerfile

Most typically, we advise on having a staging registry and production registry. Meaning, being able to push and pull images freely from the staging/dev registry, while maintaining more control over images being pushed to the production registry. In this example, I am using the same registry for both.

I've added the following environment variables via the configuration settings page within Circle:

- `ANCHORE_CLI_PASS`
- `ANCHORE_CLI_URL`
- `ANCHORE_CLI_USER`
- `ANCHORE_FAIL_ON_POLICY`
- `ANCHORE_RETRIES`
- `ANCHORE_SCAN_IMAGE`
- `DOCKER_PASSWORD`
- `DOCKER_USERNAME`

## Build Image

In the first step of the pipeline, we build a Docker image from a Dockerfile and push it to a registry as defined in our `config.yml`:

```
build:
    machine: true
    steps:
      - checkout
      - run:
          name: Build and push Docker image
          command: |
            docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
            docker build -t $DOCKER_USERNAME/sampledockerfiles:latest .
            docker push $DOCKER_USERNAME/sampledockerfiles:latest
```

## Scan Image with Anchore

In the second step of the pipeline, we scan the built image with Anchore as definied in our `config.yml`:

```
scan:
    docker:
      - image: anchore/engine-cli:latest
    steps:
      - run:
          name: Anchore Scan
          command: |
            echo "Adding image to Anchore Engine"
            anchore-cli image add $ANCHORE_SCAN_IMAGE
            echo "Waiting for image analysis to complete"
            counter=0
            while (! (anchore-cli image get $ANCHORE_SCAN_IMAGE | grep 'Status\:\ analyzed') ) ; do echo -n "." ; sleep 10 ; if [ $counter -eq $ANCHORE_RETRIES ] ; then echo " Timeout waiting for analysis" ; exit 1 ; fi ; counter=$(($counter+1)) ; done
            echo "Analysis complete"
            if [ "$ANCHORE_FAIL_ON_POLICY" == "true" ] ; then anchore-cli evaluate check $ANCHORE_SCAN_IMAGE  ; fi
```

Depending on the output of the policy evaluation, the pipeline may or may not fail. In this case, I have set `ANCHORE_FAIL_ON_POLICY` to true and exposed port 22. This is in violation of a policy rule, so the build will fail during this step.

## Push Image

In the final step of the pipeline, we push the Docker image to a registry as defined in the `config.yml`:

```
push:
    machine: true
    steps:
      - run:
          name: Push image to Docker hub
          command: |
            docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
            docker build -t $DOCKER_USERNAME/sampledockerfiles:latest .
            docker push $DOCKER_USERNAME/sampledockerfiles:latest
```


## CircleCI Workflow

Putting all the steps together, we define a sequential worklow in our `config.yml` as follows:

```
workflows:
  version: 2
  build_scan_push:
    jobs:
      - build
      - scan:
          requires:
            - build
      - push:
          requires:
            - scan
```

As a reminder, we advise having separate Docker registries for images that are being scanned with Anchore, and images that have passed an Anchore scan. For example, a registry for dev/test images, and a registry to certified, trusted, production-ready images. 