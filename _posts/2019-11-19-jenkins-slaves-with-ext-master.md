---
title: "Dynamic Jenkins Slaves on Openshift with External Master"
categories:
  - openshift
tags:
  - Openshift
  - OCP
excerpt: "Learn how to run dynamic Jenkins slaves on Openshift"
modified: {{ page.last_modified_at }}
comments: true
share: true
---
## Table of Contents
1. [Quick intro of Jenkins](#quick-intro-of-jenkins)
2. [Install Jenkins](#install-jenkins)
3. [Configure Jenkins](#configure-jenkins)
4. [Add Kubernetes and Openshift Client Plugins](#add-kubernetes-and-openshift-client-plugins)
5. [Enable JNLP TCP Port](#enable-jnlp-tcp-port)
6. [Prepare Openshift Environment](#prepare-openshift-environment)
7. [Prepare Jenkins to talk to Openshift](#prepare-jenkins-to-talk-to-openshift)
8. [Create a pipeline and run a build](#create-a-pipeline-and-run-a-build)
9. [Links](#links)

## Quick intro of Jenkins

[Jenkins](https://jenkins.io/doc/#what-is-jenkins) is one of the most popular open source [CI/CD](https://www.redhat.com/en/topics/devops/what-is-ci-cd) tool used by many. Jenkins is used primarily to execute _continuous delivery pipelines_.

> A continuous delivery pipeline is an automated expression of your process for getting software from version control right through to your users and customers. 

The distributed architecture of Jenkins involves: 
1. One or more masters
2. One or more slaves/agents


### What is a Jenkins Master ?
_At "master" operating by itself is the basic installation of Jenkins and in this configuration the master handles all tasks for your build system. In most cases installing an agent doesn't change the behavior of the master. It will serve all HTTP requests, and it can still build projects on its own._ <sup>1</sup>


### What is a Jenkins Slave/Agent ?
_An agent is a computer that is set up to offload build projects from the master and once setup this distribution of tasks is fairly automatic. The exact delegation behavior depends on the configuration of each project; some projects may choose to "stick" to a particular machine for a build, while others may choose to roam freely between agents._<sup>1</sup>

### Jenkins and Kubernetes 
With Kubernetes being a popular platform for running apps within Linux containers, one can run Jenkins too in few different ways. The most popular architectures being :
1. Jenkins masters and slaves on Kubernetes; allows for defining multiple masters and dynamic slaves both behind their respective service definitions.
2. Running Jenkins master(s) outside of Kubernetes and let the agents then be dynamically provisioned onto a Kubernetes platform. 

The latter (#2) is possible with [Jenkins Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin).

## Dynamic Jenkins slaves on Openshift with External master

In this post, we will focus on the running Jenkins agents in Openshift with an external master. 

### Prerequisites
1. A VM to run Jenkins master outside of Openshift
2. An Openshift environment

### Install Jenkins  

[Download Jenkins](https://jenkins.io/download/) and follow the instructions shown after selecting Jenkins for your specific VM/platform.

For Fedora/RHEL: 

```sh 
# Setup the repo
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat/jenkins.io.key

# Install Jenkins
sudo yum -y install jenkins 

# Enable Jenkins
sudo /usr/lib/systemd/systemd-sysv-install enable jenkins

# Start Jenkins (requires OpenJDK to be installed as a prerequisite)
sudo systemctl start jenkins
```

### Configure Jenkins

Navigate to **`http://<jenkins-master-vm-ip-address>:8080`** to start configuring Jenkins

Unlock Jenkins by following the instructions on the screen 

[![Unlock Jenkins](/images/jenkins-unlock-jenkins.png)](/images/jenkins-unlock-jenkins.png)

Install recommended plugins or all of them in the screen as shown below

[![Install plugins](/images/jenkins-install-plugins.png)](/images/jenkins-install-plugins.png) 

Wait till all plugins are installed

[![Plugins installation progress](/images/jenkins-plugins-installation-progress.png)](/images/jenkins-plugins-installation-progress.png)

Create an Admin user and continue by clicking on `Save and Continue`

[![Admin user](/images/jenkins-create-admin-user.png)](/images/jenkins-create-admin-user.png)

Note the Jenkins instance details and click on `Save and Finish` and in the screen following the one shown below, click on `Start using Jenkins`.

[![Jenkins Instance](/images/jenkins-instance-details.png)](/images/jenkins-instance-details.png)

### Add Kubernetes and Openshift Client Plugins

Click on **Manage Jenkins** 🠪  **Manage Plugins**

[![](/images/jenkins-manage-plugins.png)](/images/jenkins-manage-plugins.png)

Click on **Available** tab, select **Kubernetes** and **Openshift Client** plugins and proceed by clicking on **Install without restart**

[![](/images/jenkins-available-plugins.png)](/images/jenkins-available-plugins.png)

### Enable JNLP TCP Port

Click on **Manage Jenkins** 🠪  **Configure Global Security** 🠪  Agents 

Select **Fixed** and put in port number **50000** as shown below:

[![](/images/jenkins-enable-jnlp-port.png)](/images/jenkins-enable-jnlp-port.png)

### Prepare Openshift Environment

Prepare the Openshift environment to be able to take workloads from Jenkins master. 

```sh 
# First create a new project
oc new-project jenkins

# Create a ServiceAccount, Role and RoleBinding
oc create -f https://raw.githubusercontent.com/jenkinsci/kubernetes-plugin/master/src/main/kubernetes/service-account.yml

# Give default service account in jenkins namespace the permission to edit
oc policy add-role-to-user edit system:serviceaccount:jenkins:default

# Get the token for the jenkins-sa service account and save it
oc serviceaccounts get-token jenkins -n jenkins 
```

### Prepare Jenkins to talk to Openshift

As before, in Jenkins environment go to **Manage Jenkins** and then click on **Configure System**

Scroll down to the section **Openshift Client Plugin** and click on **Add Openshift Cluster**. Fill in the rest of the details:

* Cluster Name : `Openshift`
* API Server URL : `https://api.<cluster-name>.<domain-name>:6443` 
* Credentials : Click on **Add** 🠪  **Jenkins** and on the pop-up fill the form with following details and click on **Add** when finished
  * Kind : `Openshift Token for Openshift Client Plugin` 
  * Token : `<Paste the jenkins serviceaccount token captured in the previous step>`
  * ID : `jenkins-sa-token`
* Back on Jenkins system configuration page, change the Credentials to `jenkins-sa-token` 
* Check the box **Disable TLS Verify**
* In the **Default Project**, put in **jenkins**

[![](/images/jenkins-openshift-client-plugin.png)](/images/jenkins-openshift-client-plugin.png)

After finished with **Openshift Client Plugin**, save the config and scroll to the bottom of the page to configure a cloud. Depending on the Jenkins version, you might see the cloud configuration in the same page or a different page to which a link is provided.

Either way, your cloud configuration page should look something like below

[![](/images/jenkins-configure-clouds-blank.png)](/images/jenkins-configure-clouds-blank.png)

Click on the **Add a new cloud**, choose **Kubernetes** and start filling out the details as shown below:
* Name : `Openshift`
* Kubernetes URL : `https://api.<cluster-name>.<domain-name>:6443`
* Disable https certificate check: _checked_
* Kubernetes Namespace: `jenkins`
* Credentials : Click on **Add** 🠪  **Jenkins** and on the pop-up fill the form with following details and click on **Add** when finished
  * Kind : `Secret text`
  * Secret : `<Paste the jenkins serviceaccount token captured in the previous step>`
  * ID : `jenkins-token-as-secret-text`
* Back on Jenkins Cloud configuration page, change the Credentials to `jenkins-token-as-secret-text`
  * Click on **Test Connection** to verify the connection
* Jenkins URL : `http://<jenkins-master-vm-ip-address>:8080`
* Jenkins tunnel : `:50000`

The configuration should look like the image below:

[![](/images/jenkins-configure-clouds-1.png)](/images/jenkins-configure-clouds-1.png)

Find the line that starts with **Pod Templates** and click on **Add Pod Template** button next to it and fill it with the following details:
* Name : `maven`
* Namespace: `jenkins`
* Labels : `maven` _(used in the pipelines to invoke this specific Pod Template)_
* Usage: `Only build jobs with label expressions matching this node` 

Find the line that starts with **Containers** and click on the **Add Container** button and fill it with the following details
:
* Name : `jnlp` _(must be `jnlp`)_
* Docker Image : `registry.redhat.io/openshift4/ose-jenkins-agent-maven` _(only Openshift images can execute Openshift specific DSL in pipeline syntax)_
* Working directory: `/home/jenkins/agent`

and clear the following, the result should look like the image immediately below

* Command to run 
* Arguments to pass to the command
* Allocate pseudo-TTY

[![](/images/jenkins-configure-clouds-2.png)](/images/jenkins-configure-clouds-2.png)

The remaining portion of the pod template can be left intact with the defaults as shown below.

[![](/images/jenkins-configure-clouds-3.png)](/images/jenkins-configure-clouds-3.png)

### Create a pipeline and run a build 

On the Jenkins home page, click on **New Item**, give the item a name and choose type as a **Pipeline** and click on **Ok**.

[![](/images/jenkins-pipeline-create.png)](/images/jenkins-pipeline-create.png)

Scroll down to the bottom of the pipeline configuration page and paste the following pipeline code into the editor. Since the agent is explicitly configured as **maven**, the Jenkins master would not pick this pipeline and instead would look for the **PodTemplate** with the label **maven** and run that pod within Openshift.

More such pipeline code can be obtained [from here](https://blog.openshift.com/building-declarative-pipelines-openshift-dsl-plugin)

```
pipeline {
  agent { label 'maven'}
  stages {
    stage('Say Hello') {
        steps {
            echo "Hello World!"
        }
    }
    stage('Openshift New App HTTPD') {
        steps {
            script {
                openshift.withCluster() {
                    openshift.newApp('httpd')
                }
            }
        }
    }
  }
}
```
After pasting the pipeline DSL into the editor, click on **Apply** and then run a build by clicking on the pipeline name on the top left-hand corner and choosing **Build Now**, as shown below.

[![](/images/jenkins-pipeline-configure.png)](/images/jenkins-pipeline-configure.png)

## End Result

If everything goes well you should see an **httpd** deployment in the **jenkins** namespace. 

## Links 
1. <https://wiki.jenkins.io/display/JENKINS/Distributed+builds>
2. <https://jenkins.io/doc/pipeline/steps/kubernetes>
3. <https://blog.openshift.com/building-declarative-pipelines-openshift-dsl-plugin>
