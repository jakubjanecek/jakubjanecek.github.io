---
title: 'Running Scala Steward On-premise'
date: 2020-05-18T09:00:00-00:00
draft: false
---

> This blog post was originally published at [Avast's Engineering blog](https://engineering.avast.io/running-scala-steward-on-premise/).

[Scala Steward](https://github.com/fthomas/scala-steward) is a bot that helps you keep your project’s dependencies and SBT plugins always up-to-date.
It is a very useful tool because the world of Scala is evolving rapidly and it can become quite hard to keep up with all the updates. And most of the
updates are just small so it is worth the effort to automate this process.

We have been using Scala Steward in [our open source projects](https://github.com/avast/scala-server-toolkit) and had a very good experience with it.
However we lacked a tool like that for our internal projects (in our GitHub Enterprise instance). So we decided to make it run for internal projects
and this article describes how we’ve done it.

There is [some documentation](https://github.com/fthomas/scala-steward/blob/master/docs/running.md) on how to run Steward which is a great start, but
not everything is well-documented and we needed to make some tweaks.

## Custom Docker Image

Steward can be run either directly via SBT or as Docker image. I do not think that build tools should be used to run applications so I chose the
latter option. There is no official Scala Steward image so the first thing you need to do is build one.

```bash
git clone git@github.com:fthomas/scala-steward.git
cd scala-steward
sbt docker:publishLocal
docker push docker.yourcompany.com/fthomas/scala-steward:<VERSION>
```

We made some changes to the built Docker image. We use a different base image (internal one used for most JVM projects) and we
install [awscli](https://aws.amazon.com/cli/) (I will talk about that later). These changes can be made inside build.sbt so you need to make an
internal fork of the Scala Steward repository.

```scala
dockerBaseImage := "docker.yourcompany.com/base/openjdk-8-build:0.0.1",
dockerCommands ++= Seq(
  Cmd("USER", "root"),
  Cmd("RUN", "yum", "-y", "install", "awscli")
)
```

Once you have the Docker image you need some place to run Steward periodically. There are many options but we run Steward in our internal Kubernetes
cluster as a cron job.

## Deployment to Kubernetes

You need to create an YAML file describing how Steward should be deployed. This is our configuration:

```yaml
kind: CronJob
apiVersion: batch/v1beta1
metadata:
  name: scala-steward
  namespace: avast
spec:
  schedule: 0 * * * *
  concurrencyPolicy: Forbid
  suspend: false
  jobTemplate:
    metadata:
      labels:
        app: scala-steward
    spec:
      template:
        spec:
          containers:
            - name: app
              image: docker.yourcompany.com/fthomas/scala-steward:0.5.0-16e5d3bd
              command: [ "/bin/bash" ]
              args:
                - "-c"
                - >-
                  export WORKSPACE=/opt/docker/workspace &&
                  echo "Cloning git.yourcompany.com/scala/scala-steward-repos.git" &&
                  git clone https://scala-steward:${SCALA_STEWARD_PERSONAL_ACCESS_TOKEN}@git.yourcompany.com/scala/scala-steward-repos.git &&
                  echo "Configuring awscli to access S3-like storage to sync Scala Steward's workspace" &&
                  aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID} &&
                  aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY} &&
                  echo "Restoring Scala Steward's workspace ${WORKSPACE} from S3: ${S3_URL} s3://scala-steward" &&
                  mkdir ${WORKSPACE} &&
                  aws --endpoint-url=${S3_URL} --no-verify-ssl s3 cp s3://scala-steward/workspace.tar.gz workspace.tar.gz &&
                  tar xf workspace.tar.gz -C ${WORKSPACE} &&
                  echo "Running Scala Steward" &&
                  /opt/docker/bin/scala-steward --workspace ${WORKSPACE} --repos-file /opt/docker/scala-steward-repos/repos.md --git-author-name scala.steward --git-author-email scala-steward@avast.com --vcs-api-host https://git.yourcompany.com/api/v3 --vcs-login scala-steward --git-ask-pass /opt/docker/scala-steward-repos/.github/askpass/scala-steward.sh --do-not-fork --disable-sandbox &&
                  echo "Saving Scala Steward's workspace ${WORKSPACE} to S3: ${S3_URL} s3://scala-steward" &&
                  tar czf workspace.tar.gz -C ${WORKSPACE} . &&
                  aws --endpoint-url=${S3_URL} --no-verify-ssl s3 cp workspace.tar.gz s3://scala-steward/ &&
                  echo "Scala Steward Finished"
              env:
                - name: SCALA_STEWARD_PERSONAL_ACCESS_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: scala-steward
                      key: github-personal-access-token
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: scala-steward
                      key: aws-access-key-id
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: scala-steward
                      key: aws-secret-access-key
                - name: S3_URL
                  value: https://s3like.yourcompany.com
              resources:
                limits:
                  cpu: 4000m
                  memory: 4Gi
                requests:
                  cpu: 2000m
                  memory: 2Gi
          restartPolicy: Never
          terminationGracePeriodSeconds: 30
```

As you can see it is a definition of a Kubernetes CronJob resource scheduled to run every hour. It takes the Docker image we built internally and runs
a series of commands to achieve what we set out to do in the beginning.

## Persistence

The first thing that is a bit complicated is persistence. Steward is able to run without any storage but certain features do not work without it (e.g.
[choosing PR frequency](https://github.com/fthomas/scala-steward/blob/master/docs/repo-specific-configuration.md)). Our Kubernetes cluster does not
support persistent volumes so we needed some remote storage. Thankfully we have an internal S3-like storage that we could use. That is the reason for
installing awscli into our Docker image.

Basically Steward’s workspace archive is downloaded before the run and uploaded back after Steward finishes. It does not support parallel run of
multiple Steward instances but we do not need that so it is absolutely fine for us.

The run command seems to be complex mostly because we need to do workspace synchronization. It would be much simpler without it.

## Configuration

The last missing piece to describe is how Steward is configured.

We have a repository with a list of internal repositories that should be updated by Steward. It looks the same as
the [public variant](https://github.com/scala-steward-org/repos/blob/master/repos-github.md).

This repository is cloned at the beginning of the update process and it also contains the `git-ask-pass` script:

```bash
#!/bin/bash

echo ${SCALA_STEWARD_PERSONAL_ACCESS_TOKEN}
```

We are using the following command-line options to run Steward:

```
--workspace ${WORKSPACE}
--repos-file /opt/docker/scala-steward-repos/repos.md
--git-author-name scala.steward
--git-author-email scala-steward@avast.com
--vcs-api-host https://git.yourcompany.com/api/v3
--vcs-login scala-steward
--git-ask-pass /opt/docker/scala-steward-repos/.github/askpass/scala-steward.sh
--do-not-fork
--disable-sandbox
```

We are setting where the Steward’s workspace is, where the repos.md file is and several options to configure Git (GitHub) access using a service
account and its personal access token.

The OSS version of Scala Steward does not have direct access to the repositories it updates so it needs to create forks. We have the luxury that we
can give Steward the proper access so the `–do-not-fork` option is enabled. We also don’t need to run Steward in a sandbox so it is disabled.

## Logging

Logs from standard out are pushed to Kibana so we can always see how Steward is doing. This is quite important because we have encountered several
issues in the past which we could not resolve without some information from logs (the issues were mostly related to the workspace synchronization).

## Conclusion

We have been running Scala Steward since February 2020. It has created around 750 PRs since then. It is a huge amount of work we would have to do
manually otherwise and it helps us keep up with the rapid pace of development – yes, we do also evolve our internal libraries quite fast.

It can sometimes be a bit overwhelming to receive so many PRs from Steward. First, it gets better over time as you update most of your dependencies.
Second, some developers use the PR frequency setting to make it more manageable. Overall I think most Scala developers are happy that we have Steward
running inside our company.
