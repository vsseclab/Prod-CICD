#### How does a service reach production?

First, the code is built using a build  stage. Then comes the testing stage where testing is conducted. After that, the code is deployed in the Dev environment and goes through the UAT approval process. Once it is approved, it is deployed in the UAT environment. Then it goes through production approval before being deployed to production environments. These stages are included in a general CICD pipeline, but they may vary depending on the use case.

Let's create an end-to-end CICD pipeline using Harness CI. It will consist of Build, Deploy, and Approval stages and will also demonstrate different types of deployment such as canary and rolling deployment using Harness CD.

#### Prerequisite :

Before getting started with this CICD pipeline, it is essential to have a basic understanding of the Harness BUILD and Deploy stages. Please refer to the following resources to gain a better understanding.

*   [Docs Harness CI](https://developer.harness.io/docs/continuous-integration) (required while setting up connectors and delegate)
*   [Docs Harness CD](https://developer.harness.io/docs/continuous-delivery) (required while setting up Deployment infra and Service)
*   [youtube.com/@harnesscommunity](https://www.youtube.com/@harnesscommunity) (Go through Harness CI CD videos to get started if required)
*   [Harness Community Github](https://github.com/harness-community)

#### High-level workflow of the pipeline is as follows:

*   build\_test\_and\_run stage:
    *   Code\_compile step: Compiles the Python code.
    *   Create\_image step: Creates a Docker image of the Python code and pushes it to a Docker registry.
    *   Build\_and\_Push\_an\_image\_to\_docker\_registry step: Builds and pushes the Docker image to a Docker registry.
*   test stage:
    *   curl step: Sends a curl request to the local server running the Docker image and tests it on different operating systems (Linux, MacOS, and Windows).
*   DEV\_ENV stage:
    *   rolloutDeployment step: Deploys the application to the development environment.
*   Prod\_Approval stage:
    *   prod\_approval step: Requests approval from designated users or user groups to proceed to the next stage.
*   UAT\_ENV stage:
    *   rolloutDeployment step: Deploys the application to the user acceptance testing (UAT) environment.
*   UAT\_Approval stage:
    *   uat\_approval step: Requests approval from designated users or user groups to proceed to the next stage.
*   Prod1\_ENV stage:
    *   rolloutDeployment step: Deploys the application to the first production environment.
*   Prod2\_ENV stage:
    *   canaryDeployment step: Deploys the application to the second production environment
*   Update\_Jira  stage:
    *   Update the Jira issue as completed

#### Setting up the Pipeline YAML:

*   Fork [ronakforgit/Prod-CI-CD](https://github.com/ronakforgit/Prod-CI-CD) repository 
*   Go to the Harness platform
*   Click on `Create Pipeline`. 
*   Give a `Name` to your pipeline.
*   Optionally, you can provide a `Description` and `Tags`
*   Select `Remote` option on `How do you want to setup your pipeline`.
*   Replace  reference to connectors, repositories, and registries  before executing the pipeline
*   Add the pipeline.yaml  in the YAML editor

```plaintext
  properties:
    ci:
      codebase:
        connectorRef: ronakforgit
        repoName: Prod-CI-CD
        build: <+input>
  stages:
    - stage:
        identifier: build_test_and_run
        type: CI
        name: build
        spec:
          cloneCodebase: true
          repoName: Prod-CI-CD
          execution:
            steps:
              - step:
                  identifier: Code_compile
                  type: Run
                  name: Code compile
                  spec:
                    connectorRef: account.harnessImage
                    image: python:3.10.6-alpine
                    shell: Sh
                    command: python -m compileall ./
              - step:
                  identifier: Create_image
                  type: Run
                  name: Create image
                  spec:
                    connectorRef: account.harnessImage
                    image: alpine
                    shell: Sh
                    command: |-
                      touch pythondockerfile
                      cat > pythondockerfile <<- EOM
                      FROM python:3.10.6-alpine
                      WORKDIR /py-sample-proj
                      ADD . /py-sample-proj
                      RUN pip install -r requirements.txt
                      ENTRYPOINT ["python", "app.py"]
                      EOM
                      cat pythondockerfile
              - step:
                  identifier: Build_and_Push_an_image_to_docker_registry
                  type: BuildAndPushDockerRegistry
                  name: Build and Push an image to docker registry
                  spec:
                    connectorRef: <+input>
                    repo: <+input>
                    tags:
                      - latest
                    dockerfile: pythondockerfile
                    optimize: true
          infrastructure:
            type: KubernetesDirect
            spec:
              connectorRef: CIk8GCPRonak
              namespace: default
              automountServiceAccountToken: true
              nodeSelector: {}
              os: Linux
        variables:
          - name: container
            type: String
            description: ""
            value: docker
    - stage:
        identifier: test
        type: CI
        name: test
        spec:
          cloneCodebase: false
          execution:
            steps:
              - step:
                  identifier: curl
                  type: Run
                  name: curl
                  spec:
                    connectorRef: account.harnessImage
                    image: curlimages/curl
                    shell: Sh
                    command: |-
                      echo " <+matrix.testos> "
                      curl localhost:5000
                  failureStrategies: []
                  strategy:
                    matrix:
                      testos:
                        - macos
                        - windows
                        - linux
                    maxConcurrency: 3
          serviceDependencies:
            - identifier: docker
              type: Service
              name: docker
              spec:
                connectorRef: <+input>
                image: <+input>
                imagePullPolicy: Always
          infrastructure:
            useFromStage: build_test_and_run
        variables:
          - name: container
            type: String
            description: ""
            value: docker
    - stage:
        identifier: DEV_ENV
        type: Deployment
        name: DEV ENV
        description: ""
        spec:
          deploymentType: Kubernetes
          service:
            serviceRef: microservice1
            serviceInputs:
              serviceDefinition:
                type: Kubernetes
                spec:
                  artifacts:
                    primary:
                      primaryArtifactRef: docker image artifact
                      sources:
                        - identifier: docker image artifact
                          type: DockerRegistry
                          spec:
                            tag: <+input>
          environment:
            environmentRef: dev
            deployToAll: false
            infrastructureDefinitions:
              - identifier: dev
          execution:
            steps:
              - step:
                  identifier: rolloutDeployment
                  type: K8sRollingDeploy
                  name: Rollout Deployment
                  timeout: 10m
                  spec:
                    skipDryRun: false
                    pruningEnabled: false
            rollbackSteps:
              - step:
                  identifier: rollbackRolloutDeployment
                  type: K8sRollingRollback
                  name: Rollback Rollout Deployment
                  timeout: 10m
                  spec:
                    pruningEnabled: false
        tags: {}
        failureStrategies:
          - onFailure:
              errors:
                - AllErrors
              action:
                type: StageRollback
    - stage:
        identifier: Prod_Approval
        type: Approval
        name: UAT Approval
        description: ""
        spec:
          execution:
            steps:
              - step:
                  identifier: prod_approval
                  type: HarnessApproval
                  name: UAT approval
                  timeout: 1d
                  spec:
                    approvalMessage: |-
                      Please review the following information
                      and approve the pipeline progression
                    includePipelineExecutionHistory: true
                    approvers:
                      minimumCount: 1
                      disallowPipelineExecutor: false
                      userGroups:
                        - pravin
                        - org._organization_all_users
                    approverInputs:
                      - name: release No
                        defaultValue: ""
        tags: {}
    - stage:
        identifier: UAT_ENV
        type: Deployment
        name: UAT ENV
        description: ""
        spec:
          deploymentType: Kubernetes
          service:
            useFromStage:
              stage: DEV_ENV
          environment:
            environmentRef: UAT
            deployToAll: false
            infrastructureDefinitions:
              - identifier: UAT
          execution:
            steps:
              - step:
                  identifier: rolloutDeployment
                  type: K8sRollingDeploy
                  name: Rollout Deployment
                  timeout: 10m
                  spec:
                    skipDryRun: false
                    pruningEnabled: false
            rollbackSteps:
              - step:
                  identifier: rollbackRolloutDeployment
                  type: K8sRollingRollback
                  name: Rollback Rollout Deployment
                  timeout: 10m
                  spec:
                    pruningEnabled: false
        tags: {}
        failureStrategies:
          - onFailure:
              errors:
                - AllErrors
              action:
                type: StageRollback
    - stage:
        identifier: UAT_Approval
        type: Approval
        name: Prod Approval
        description: ""
        spec:
          execution:
            steps:
              - step:
                  identifier: create_a_issue
                  type: JiraCreate
                  name: create a issue
                  spec:
                    connectorRef: jira
                    projectKey: CME
                    issueType: Task
                    fields:
                      - name: Assignee
                        value: ronak.patil@harness.io
                      - name: Description
                        value: approve this  deployment for prod
                      - name: Summary
                        value: some summary what you want - i cant think of any now - edit  this is in jira create section for approval
                  timeout: 1d
              - step:
                  identifier: approve_the_issue
                  type: JiraApproval
                  name: approve the issue
                  spec:
                    connectorRef: jira
                    issueKey: <+pipeline.stages.UAT_Approval.spec.execution.steps.create_a_issue.issue.id>
                    approvalCriteria:
                      type: KeyValues
                      spec:
                        matchAnyCondition: true
                        conditions:
                          - key: Status
                            operator: equals
                            value: In Progress
                    rejectionCriteria:
                      type: KeyValues
                      spec:
                        matchAnyCondition: true
                        conditions: []
                  timeout: 1d
        tags: {}
    - parallel:
        - stage:
            identifier: Prod_1_ENV
            type: Deployment
            name: Prod 1 ENV
            description: ""
            spec:
              deploymentType: Kubernetes
              service:
                useFromStage:
                  stage: DEV_ENV
              environment:
                environmentRef: PROD1
                deployToAll: false
                infrastructureDefinitions:
                  - identifier: PROD1
              execution:
                steps:
                  - step:
                      identifier: rolloutDeployment
                      type: K8sRollingDeploy
                      name: Rollout Deployment
                      timeout: 10m
                      spec:
                        skipDryRun: false
                        pruningEnabled: false
                rollbackSteps:
                  - step:
                      identifier: rollbackRolloutDeployment
                      type: K8sRollingRollback
                      name: Rollback Rollout Deployment
                      timeout: 10m
                      spec:
                        pruningEnabled: false
            tags: {}
            failureStrategies:
              - onFailure:
                  errors:
                    - AllErrors
                  action:
                    type: StageRollback
        - stage:
            identifier: PROD_Can
            type: Deployment
            name: PROD 2 ENV
            description: ""
            spec:
              deploymentType: Kubernetes
              service:
                useFromStage:
                  stage: DEV_ENV
              environment:
                environmentRef: PROD2
                deployToAll: false
                infrastructureDefinitions:
                  - identifier: PROD2
              execution:
                steps:
                  - stepGroup:
                      identifier: canaryDepoyment
                      name: Canary Deployment
                      steps:
                        - step:
                            identifier: canaryDeployment
                            type: K8sCanaryDeploy
                            name: Canary Deployment
                            timeout: 10m
                            spec:
                              instanceSelection:
                                type: Count
                                spec:
                                  count: 1
                              skipDryRun: false
                        - step:
                            identifier: canaryDelete
                            type: K8sCanaryDelete
                            name: Canary Delete
                            timeout: 10m
                            spec: {}
                  - stepGroup:
                      identifier: primaryDepoyment
                      name: Primary Deployment
                      steps:
                        - step:
                            identifier: rollingDeployment
                            type: K8sRollingDeploy
                            name: Rolling Deployment
                            timeout: 10m
                            spec:
                              skipDryRun: false
                rollbackSteps:
                  - step:
                      identifier: rollbackCanaryDelete
                      type: K8sCanaryDelete
                      name: Canary Delete
                      timeout: 10m
                      spec: {}
                  - step:
                      identifier: rollingRollback
                      type: K8sRollingRollback
                      name: Rolling Rollback
                      timeout: 10m
                      spec: {}
            tags: {}
            failureStrategies:
              - onFailure:
                  errors:
                    - AllErrors
                  action:
                    type: StageRollback
    - stage:
        identifier: update_jira
        type: Approval
        name: update jira
        description: ""
        spec:
          execution:
            steps:
              - step:
                  identifier: update_issue_to_done
                  type: JiraUpdate
                  name: update issue to done
                  timeout: 5m
                  spec:
                    connectorRef: jira
                    issueKey: <+pipeline.stages.UAT_Approval.spec.execution.steps.create_a_issue.issue.id>
                    transitionTo:
                      transitionName: ""
                      status: Done
                    fields: []
        tags: {}
  variables: []
  notificationRules:
    - identifier: Pipeline_Notification
      name: Pipeline Notification
      pipelineEvents:
        - type: StageFailed
          forStages:
            - build_test_and_run
            - test
            - DEV_ENV
            - UAT_Approval
            - UAT_ENV
            - Prod_Approval
            - Prod_1_ENV
            - Prod2_Env
        - type: StageSuccess
          forStages:
            - build_test_and_run
            - test
            - DEV_ENV
            - UAT_Approval
            - UAT_ENV
            - Prod_Approval
            - Prod_1_ENV
            - Prod2_Env
      notificationMethod:
        type: Email
        spec:
          userGroups: []
          recipients:
            - ronak.patil@harness.io
      enabled: true
  identifier: CICD_user_input
  name: CICD - user input
```

*   Execute the pipeline after setting up all the delegates, connectors, service, and deployment infrastructure.
