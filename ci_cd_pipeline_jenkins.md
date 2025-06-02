# Setting up a CI/CD pipeline in Jenkins (On Ramon's server)
## Contents
- [Setting up a CI/CD pipeline in Jenkins (On Ramon's server)](#setting-up-a-cicd-pipeline-in-jenkins-on-ramons-server)
  - [Contents](#contents)
  - [Before you make any jobs](#before-you-make-any-jobs)
    - [Setting up your GitHub repo for the project](#setting-up-your-github-repo-for-the-project)
    - [Setting up a Jenkins-GitHub key pair](#setting-up-a-jenkins-github-key-pair)
  - [Job 1: CI - Testing](#job-1-ci---testing)
  - [Job 2: CI - Merge](#job-2-ci---merge)
  - [Job 3: CD - Deploy](#job-3-cd---deploy)
    - [Job 3 on AWS](#job-3-on-aws)
    - [Job 3 on Jenkins](#job-3-on-jenkins)
  - [Handy to know](#handy-to-know)


## Before you make any jobs
### Setting up your GitHub repo for the project
1. Create a GitHub repo for the app you want to deploy with Jenkins
   - The repo should be completely unpackaged (no zips!)
   - The repo should contain an **app** directory
   - It should be named something like **sparta-app-jenkins**
### Setting up a Jenkins-GitHub key pair
2. Use the command `ssh-keygen -t rsa -b 4096 -C "[YOUR EMAIL ADDRESS]"` to create a new ssh key pair
   - Name it something like **nathan-jenkins-to-github**
3. Add the **public** key of the new pair in the GitHub repo settings
   - ![Where to add the public Jenkins-GitHub key](imgs/github_key.png)
   - Don't forget to tick **Allow write access**
4. You will add the **private** key to Jenkins soon... don't lose it!

## Job 1: CI - Testing
1. From the Jenkins dashboard, select **New Item** to create a new job
   1. Name it something appropriate (e.g. **nathan-job1-ci-testing**)
   2. Select **Freestyle project**
   3. Click **OK**
2. Select **Discard old builds** and set the **Max # of builds to keep** to **5**
3. Select **GitHub project** and add the **HTTPS** version of your GitHub repo's URL in the box.
   - Make sure to remove the ".git" from the end and replace it with a "/"
4. In the **Source Code Management** section, mark the project as **Git** and:
   1.  Add the **SSH** version of your GitHub repo's URL 
   2.  **Add** a new set of **Credentials** using the **private** key you created earlier
   3. Specify the dev branch with `*/dev`
5. To set up the **build trigger**:
   1. In the Jenkins job config, select **GitHub hook trigger for GITScm polling** as a **Build Trigger**
   2. In the GitHub project's settings, set up the webhook
      - ![Github Webhook settings](imgs/github_webhook.png)
6. In **Build Environment**, select "Provide Node & npm bin/ folder to PATH"
   - NodeJS v.20
7. Execute shell
   - cd app; npm install; npm test
8. **Save** it

## Job 2: CI - Merge
1. Select **New Item** again and make another **Freestyle project**, naming it something like **nathan-job2-ci-merge**
2. Set up the **Discard old builds**, **GitHub project**, and **Source Code Management** (using the previously made credentials) as before
3. In **Build Environment**, as well as the set-up you did for job 1, also select **SSH Agent** and give it the credentials you set-up earlier
4. In **Post-build Actions**, add a **Git Publisher** to merge the branch if the tests succeeded
   - ![Git Publisher](imgs/git_publisher.png)
5. Go back to job 1 and adjust the config by adding an extra **Post-build Action** to **Build other projects**, specifying job 2 as the project to build

## Job 3: CD - Deploy
To get job 3 working, some set-up needs to be done on **AWS** before the jenkins set-up can be done.
### Job 3 on AWS
1. Decide if you want to create a new key-pair for Jenkins to use to communicate with the app instance (recommended) and create that key pair
2. Set up an EC2 instance for jenkins to run the app on
   - Use the key pair you made/chose in the previous step
   - Use the usual app instance settings
   - Make sure the firewall rules allow SSH from anywhere, so Jenkins can get in (as Jenkins spins up worker nodes, the IPs of which we don't know)
3. Once the instance is running, take note of it's address (should look like ubuntu@ec2-[IP address].eu-west-1.compute.amazonaws.com) as we'll be using it in the Jenkins set-up
4. SSH into the instance to set-up dependencies (nginx, nodejs, pm2)
   - ```
      sudo apt update
      sudo apt upgrade -y
      sudo DEBIAN_FRONTEND=noninteractive apt-get install nginx -y
      sudo DEBIAN_FRONTEND=noninteractive bash -c "curl -fsSL https://deb.nodesource.com/setup_20.x | bash -"
      sudo DEBIAN_FRONTEND=noninteractive apt-get install nodejs -y
      sudo npm install pm2@latest -yg
      ```
### Job 3 on Jenkins
5. Create another job, with **Discard old builds** and **GitHub project** set up as before
6. Set up **Source Code Management** as before too, but now specify the branch as `*/main`
7. In **Build Environment**, set up in the same way as job 2, but give the SSH agent credentials for AWS with whichever private key goes with the pair you decided to use in step 1 of Job 3
   - ![Set up AWS credentials on Jenkins](imgs/aws_key.png)
8. In the **Build Steps**, create an **Execute shell** with the following commands:
   - ```
      scp -o StrictHostKeyChecking=no -B -r ./app ubuntu@ec2-[IP address].eu-west-1.compute.amazonaws.com:~
      ssh ubuntu@ec2-[IP address].eu-west-1.compute.amazonaws.com "pm2 delete all; cd app; npm install; pm2 start npm --name sparta-test-app -- start"
      ```
   - The `scp` command transfers files (specifically the directory (`-r`) located at `./app`) to the specified remote location, replacing any existing `./app` directory (`-B`)
   - The `ssh` command in this particular instance will stop pm2 processes, reinstall the app to accomodate changes, and rerun it

## Handy to know
- If you need to edit **credentials**, you can find that here:
  - ![Where to find credentials in Jenkins](imgs/find_credentials_jenkins.png)
- When creating a new **job**/project, you can copy all the config from a previous job:
  - ![How to copy job config](imgs/copy_jenkins_project.png)
- Test it works by pushing to the **dev** branch of the GitHub project