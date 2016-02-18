# GitLab (CE) on OpenShift Enterprise 3
---

## Prerequisites
* OpenShift Enterprise 3.1.1

## Fundamentals
The purpose of this repository is to explain how to get [GitLab CE](https://about.gitlab.com/) running on [OpenShift Enterprise 3](https://www.openshift.com/enterprise/). This guide makes use of some basic OSE concepts and uses the [official GitLab CE](https://hub.docker.com/r/gitlab/gitlab-ce/) docker image. 

## Rationale
There has been a tremendous amount of activity within the "DevOps" and "CICD" communitites that has empowered developers to more quickly build and test applications without requiring increased administrative overhead. Development teams can focus on creating and delivering applications while Operations teams can focus on broader issues than provisioning ephemeral resources.

To further the goal of developer enablement, especially with respect to streamlining development pipelines, it would be a *good thing*^(tm) to host your entire CICD pipeline within your PaaS/Application Hosting/Process Virtualization environment. This guide aims to allow you to host your source control (though GitLab is useful for other things too) within OpenShift Enterprise which already is capable of hosting Jenkins as part of a continuous integration pipeline.

Future plans include installing [Eclipse Che](https://eclipse.org/che/) so that the development IDE can also be hosted within the cloud. This represents the "perfect storm" of being able to develop anything, anywhere.

## Limitations
This guide is for the OmniBus edition of GitLab CE. It may make more sense, in the long run, to have custom GitLab images so that you can connect to an external database and properly configure HA. This simple guide is just a starting point more suited to smaller-scale development environments. This guide is also written by someone more at home with development. It is probably more cavalier with permissions than most people would like.

## GitLab on OSE

### Task Overview
This guide consists of the following steps
* Clone this repository
* Create a suitable NFS mount point for GitLab's data.
* Import the GitLab CE image stream
* Import the GitLab CE template
* Modify your SCC so that GitLab can run
* Create the GitLab deployment
* Attach NFS volumes directly to the created deployment configuration
* Finishing Up

### Clone the `gitlab-ose` Repository
On a host that has the `oc` command for OSE installed:
```
git clone https://github.com/chrisruffalo/ose-gitlab.git
cd ose-gitlab
```

### NFS Mounts
If you wish to use persistent volumes and persistent volume claims that is outside the scope of this guide. If you are more familiar with NFS mount points then you will probably want to accomplish things your own way. In this example it is assumed that `/opt/nfs` is the root of your NFS exports and `/opt/nfs/ose` has been created specifically for OSE mounts.

The GitLab CE docker container uses the mounts [described in the documentation](http://doc.gitlab.com/omnibus/docker/README.html). As a result we need to create a place for those files to be mounted so that the data for GitLab will persist. First you will need to ensure that your NFS server is exporting the correct files. Keep in mind that one of the first things that GitLab will do is change permissions and ownership so the `root_squash` option will cause the execution to fail.

Make the GitLab mount point
```
$ mkdir -p /opt/nfs/ose/gitlab/data
$ mkdir -p /opt/nfs/ose/gitlab/data
$ mkdir -p /opt/nfs/ose/gitlab/data
$ chown -R nfsnobody:nfsnobody /opt/nfs/ose/gitlab
$ chmod -R 775 /opt/nfs/ose/gitlab
```

Add GitLab mount point to /etc/exports
```
$ /opt/nfs/ose/gitlab *(rw,no_root_squash,sync)
```
The permissions here are broad. You should replace `*` with a pattern that better replicates the hostnames or IP ranges of where GitLab will be hosted.

### Import the GitLab CE Stream
This will add a GitLab CE image stream to your OSE instance. This will allow your OSE to 'track' the GitLab public images as they are made.
```
$ oc create -f gitlab-ce-image-stream.json -n openshift
```
It is important to create in the openshift namespace if you want it to be available globally.

### Import the GitLab CE template
This will add the application template that gives OSE the structure it needs to deploy the GitLab cartridge inside of a pod.
```
$ oc create -f gitlab-persistent-template.json -n openshift
```
The same thing about the namespace.

### Modify the SCC
Because GitLab relies on Chef, changing permissions, and a few other quirks it is necessary to run it as a privileged container. In order to do this you will need to make some modifications to what OpenShift allows. Edit the scc for restricted and change the following lines:
```
$ oc edit scc restricted

allowPrivilegedContainer: true
runAsUser:
  type: RunAsAny
```

Similarly, edits need to be made to the privileged scc:
```
$ oc edit scc privileged

allowPrivilegedContainer: true
runAsUser:
  type: RunAsAny
users:
- ${your user name}
```
Once this is complete that should be sufficient to allow the GitLab container to run.

### Create the GitLab deployment
You can do this in the OSE UI by clicking "Add to project" within your project and searching for "git" which will bring up the GitLab entry.

You can also produce the same effect by creating a new application within your project's namespace.
```
$ oc project ${your project}
Now using project "${your project}" on server "https://${master domain}:8443".

$ oc oc new-app gitlab-persistent
--> Deploying template gitlab-persistent in project openshift for "gitlab-persistent"
     With parameters:
      GITLAB_SERVICE_NAME=gitlab
--> Creating resources ...
    Service "gitlab" created
    Route "gitlab" created
    DeploymentConfig "gitlab" created
--> Success
    Run 'oc status' to view your app.
```

### Attaching NFS Volumes
This step will attach NFS volumes from your NFS server's mount point. The `nfs server` variable should be the hostname of your NFS server. The paths in the "source" block are based on what was used in the earlier step that created the NFS mount points.
```
$ oc volume dc/gitlab --add --name=gitlab-data --mount-path=/var/opt/gitlab --source='{"nfs": { "server": "${nfs server}", "path": "/opt/nfs/ose/gitlab/data"}}'
$ oc volume dc/gitlab --add --name=gitlab-logs --mount-path=/var/log/gitlab --source='{"nfs": { "server": "${nfs server}", "path": "/opt/nfs/ose/gitlab/logs"}}'
$ oc volume dc/gitlab --add --name=gitlab-config --mount-path=/etc/gitlab --source='{"nfs": { "server": "${nfs server}", "path": "/opt/nfs/ose/gitlab/config"}}'
```
Each time you modify the deployment config (`dc`) a new deployment will be created. After the third volume is added it should stabilize and begin the deployment in earnest.

### Finishing Up
Now GitLab CE is running and you can finish up your configuration **mainly** in the same way that is suggested by the [GitLab documentation](http://doc.gitlab.com/omnibus/docker/README.html). Instead of using the command in the documentation to login to the container you will need to execute the bash command on the OSE pod with `oc exec -ti gitlab-X-#### /bin/bash`. (Where X is the deploy number and #### is the unique pod ID.) Once you are inside the pod you can configure the application normally.

You can also edit `/opt/nfs/ose/gitlab/config/gitlab.rb` as normal.

The most important field to edit is `external_url` to change that to match the external hostname for the GitLab service. Many other features can be accomplished through the UI. If you chose to add TLS in some way then you will need to make sure that your external URL starts with "https://" otherwise it will need "http://".

Restarting GitLab is pretty easy. The simplest way is to delete the currently running pod and then just let OSE (the replication controller, anyway) restart it.

## Ports and SSH
If you want to get SSH working you will need to make a few edits. The first, in the `gitlab.rb` is to make sure that the SSH host and port are set correctly. 

```
gitlab_rails['gitlab_shell_ssh_port'] = 32222
gitlab_rails['gitlab_ssh_host'] = 'git.cloud.ruffalo.org'
```
**Note:** setting the 'gitlab_shell_ssh_port' *does not* modify what port SSH/gitlab_shell runs on within the container. It simply changes the port that is shown in the GitLab UI.

The port is the port that we will use later to make a service entry for passing information from any node to the GitLab instance. The SSH host can be the DNS entry for any host in your node cluster.

Next you will need to import the `NodePort` service entry for GitLab to expose port `32222` to the outside world.
```
$ oc project ${your project}
Now using project "${your project}" on server "https://${master domain}:8443".

$ oc create -f gitlab-direct-ssh-port.yaml
```

This will open a port on each of the nodes that will connect the external port (`32222`) to the internal service port for GitLab (`22`). After this is complete you will need to open your firewall and allow traffic to enter the node.

## Troubleshooting
The main problems that you will have during this process are issues with privileged containers and permissions and issues because OSE puts a lot of distance between the user and the docker container.