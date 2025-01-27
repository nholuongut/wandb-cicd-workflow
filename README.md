# Wandb CI CD Workflow

![](https://i.imgur.com/waxVImv.png)
### [View all Roadmaps](https://github.com/nholuongut/all-roadmaps) &nbsp;&middot;&nbsp; [Best Practices](https://github.com/nholuongut/all-roadmaps/blob/main/public/best-practices/) &nbsp;&middot;&nbsp; [Questions](https://www.linkedin.com/in/nholuong/)
<br/>

ML cicd workflow with github actions and wandb framework

## step 1. hello world github action 
```
name: GitHub Actions Demo
on : [push]

jobs :
    Explore-GitHub-Actions:
        runs-on: ubuntu-latest
        steps:
            - run: echo "the job was automatically tiggered by a ${{github.event_name}} event."
            - run: echo "this job is now running on a ${{runner.os}} server hosted by GitHub!"
            - run: echo "the name of the branch is ${{github.ref}} and your repository is ${{github.repository}}"
```

## step 2. run python file using github actions

```
name: GitHub Actions Demo
on : [push]

jobs :
    Explore-GitHub-Actions:
        runs-on: ubuntu-latest
        steps:
            - run: echo "the job was automatically tiggered by a ${{github.event_name}} event."
            - run: echo "this job is now running on a ${{runner.os}} server hosted by GitHub!"
            - run: echo "the name of the branch is ${{github.ref}} and your repository is ${{github.repository}}"
            - name: Check out the repository code
              uses: actions/checkout@v3
            - run: echo "the ${{github.repository}} has been cloned to the runner"
            - name: List files in the repository
              run: |
                    ls ${{github.workspace}}
            - name: Run python file
              run: |
                pip install -r requirements.txt
                python ci.py

```

## step 3. add trigger for pull-request using github actions

```
name: trigger-demo
on: 
    push:
        branches:
            - main
    pull_request:
    workflow_dispatch:

jobs:
    trigger-demo:
        runs-on: ubuntu-latest
        steps:
            - run: echo "first pull request tigger github action"
```

## step 4. add pr issue github action

```
name: W&B Integration
on:
    issue_comment: # tigger on pr comment

env:
    MAGIC_COMMENT: "/wandb" # this is a parameter you can change

permissions:
    contents: read
    issues: write
    pull-requests: write

jobs:
```



## step 5. Setup EC2 instance

follow below commands 

```
sudo apt-get update -y
sudo apt-get upgrade

curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

sudo usermod -aG docker ubuntu
newgrp docker
```
## step 6. setup self-hosted runner

1a)
```
# Create a folder
$ mkdir actions-runner && cd actions-runner# Download the latest runner package

$ curl -o actions-runner-linux-x64-2.303.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.303.0/actions-runner-linux-x64-2.303.0.tar.gz

# Optional: Validate the hash
$ echo "e4a9fb7269c1a156eb5d5369232d0cd62e06bec2fd2b321600e85ac914a9cc73  actions-runner-linux-x64-2.303.0.tar.gz" | shasum -a 256 -c

# Extract the installer
$ tar xzf ./actions-runner-linux-x64-2.303.0.tar.gz
```

2b)

```
# Create the runner and start the configuration experience

$ ./config.sh --url https://github.com/nholuongut/wandb-cicd-workflow --token ANJU2HIWMOBGPDBS2VC6NATEJ564O# Last step, run it!

$ ./run.sh
```

## step 7. deployment github action

<ol>
<li> <details><summary>deployment github workflow</summary><blockquote>
  <details><summary>

    name: deployment
        on:
            issue_comment:

        env:
            MAGIC_COMMENT: "/deploy" # this is a parameter you can change

        permissions:
            contents: read
            issues: write
            pull-requests: write
            deployments: write

        jobs:
            integration:
                name: Continuous Integration
                runs-on: ubuntu-latest
                steps:
                - name: Checkout Code
                    uses: actions/checkout@v3
                    with:
                        ref: dev
            
                - name: Lint code
                    run: echo "Linting repository"
            
                - name: Run unit tests
                    run: echo "Running unit tests"

            build-and-push-ecr-image:
                if: (github.event.issue.pull_request != null) && (github.event.comment.body == '/deploy')
                name: Continuous Delivery
                needs: integration
                runs-on: ubuntu-latest
                steps:
                    -   name: Checkout Code
                        uses: actions/checkout@v3
                        with:
                            ref: dev
            
                    -   name: Install Utilities
                        run: |
                            pip install -r requirements.txt
                            sudo apt-get update
                            sudo apt-get install -y jq unzip
                            python src/model_deployment.py
                        env:
                            ENTITY: 'himasha'
                            PROJECT: 'mnist-experiment'
                            REGISTRY: 'mnist-registry'
                            TAG: 'production-candidate'
                            WANDB_API_KEY: ${{secrets.WANDB_API_KEY}}

                    -   name: Configure AWS credentials
                        uses: aws-actions/configure-aws-credentials@v1
                        with:
                            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
                            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
                            aws-region: ${{ secrets.AWS_REGION }}

                    -   name: Login to Amazon ECR
                        id: login-ecr
                        uses: aws-actions/amazon-ecr-login@v1
            
                    -   name: Build, tag, and push image to Amazon ECR
                        id: build-image
                        env:
                            ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
                            ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY_NAME }}
                            IMAGE_TAG: latest
                        run: |
                            # Build a docker container and
                            # push it to ECR so that it can
                            # be deployed to ECS.
                            docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
                            docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
                            echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

            Continuous-Deployment:
                name: AWS Deployment
                needs: build-and-push-ecr-image
                runs-on: aws-cicd-host
                steps:
                    -   name: Checkout
                        uses: actions/checkout@v3
                        with:
                            ref: dev

                    -   name: Configure AWS credentials
                        uses: aws-actions/configure-aws-credentials@v1
                        with:
                            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
                            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
                            aws-region: ${{ secrets.AWS_REGION }}

                    -   name: Login to Amazon ECR
                        id: login-ecr
                        uses: aws-actions/amazon-ecr-login@v1

                    -   name: Pull latest images
                        run: |
                            docker pull ${{secrets.AWS_ECR_LOGIN_URI}}/${{ secrets.ECR_REPOSITORY_NAME }}:latest
                        
                        # - name: Stop and remove  container if running
                        #   run: |
                        #    docker ps -q --filter "name=mltest" | grep -q . && docker stop mltest && docker rm -fv mltest
                    
                    -   name: Run Docker Image to serve users
                        run: |
                            docker run -d -p 8080:8080 --ipc="host" --name=mltest -e 'AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }}' -e 'AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }}' -e 'AWS_REGION=${{ secrets.AWS_REGION }}'  ${{secrets.AWS_ECR_LOGIN_URI}}/${{ secrets.ECR_REPOSITORY_NAME }}:latest
                    -   name: Clean previous images and containers
                        run: |
                            docker system prune -f
</summary><blockquote>
</li>
</ol>

# 🚀 I'm are always open to your feedback.  Please contact as bellow information:
### [Contact ]
* [Name: nho Luong]
* [Skype](luongutnho_skype)
* [Github](https://github.com/nholuongut/)
* [Linkedin](https://www.linkedin.com/in/nholuong/)
* [Email Address](luongutnho@hotmail.com)

![](https://i.imgur.com/waxVImv.png)
![](Donate.png)
[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/nholuong)

# License
* Nho Luong (c). All Rights Reserved.🌟




















