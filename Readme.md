# GitLab Omnibus in docker documentation
This document describes how to operate with gitlab in docker container

# Motivation
This project was created to automated the update process of gtilab, to keep it up to date, simplify the restore procedure and moving all the infrastructure services to docker.

# Contents

- [Installation](#installation)
  * [Prerequisites](#prerequisites)
  * [Docker installation](#docker-installation)
  * [Run Gitlab in docker](#run-gitlab-in-docker)
  * [Gitlab CI runner installation](#gitlab-ci-runner-installation)
  * [Amazon S3-compatible object storage server installation](#Amazon_S3_compatible_object_storage_server_installation)
- [Gitlab Configuration](#gitlab-configuration)
  * [Config files and directories](#config-files-and-directories)
  * [Registry configuration](#registry-configuration)
  * [Backuping and rotating workflow](#backuping-and-rotating-workflow)
- [Maintenance and support](#maintenance-and-support)
  * [Docker container operations](#docker-container-operations)
  * [Docker cleanups](#docker-cleanups)
  * [Gitlab CI Runner support](#gitlab-ci-runner-support)
- [Update and restore Gitlab](#updating-gitlab)
  * [Update Gitlab](#update-gitlab)
  * [Restore Gitlab](#restore-gitlab)
- [Troubleshooting](#troubleshooting)
- [Usefull stuff](#usefull-stuff)
- [What to do on nearest update](#what-to-do)


# Installation

## Prerequisites

Minimum OS version Centos 7<br>
Our current setup has vms as described in table below:

| FQDN                                    | IP            |
| --------------------------------------- | ------------- |
| gitlab.mydomain.com                 | 172.29.49.12  |
| gitlab-ci-runner01.mydomain.com | 10.148.2.80   |
| gitlab-ci-runner02.mydomain.com | 10.148.0.215  |
| gitlab-ci-runner03.mydomain.com| 10.148.2.109  |
| gitlab-ci-runner04.mydomain.com | 10.148.2.110  |
| gitlab-ci-cache.mydomain.com | 10.148.0.232  |

Docker  host vm is accessible via port 2222 and it uses local pam authentication:
~~~bash
$ ssh gitlab.mydomain.com -p 2222
~~~

## Docker installation

1. Follow the instructions to uninstall all previous docker installs: <https://docs.docker.com/install/linux/docker-ee/centos/#uninstall-old-versions> <br>
2. Follow the instructions to install docker CE: <https://docs.docker.com/install/linux/docker-ce/centos/#install-docker-ce> <br />
3. Configure docker engine to allow insecure connection to registry, set proxy variables

~~~bash
$ mkdir -p /etc/systemd/system/docker.service.d/
$ vi /etc/systemd/system/docker.service.d/docker.conf

[Service]
ExecStart="--insecure-registry gitlab.mydomain.com:4567"
Environment="HTTP_PROXY=http://172.29.50.100:8080" "HTTPS_PROXY=http://172.29.50.100:8080" "NO_PROXY=*.mydomain.com"

$ systemctl daemon-reload
$ service docker restart
~~~

## Run Gitlab in docker

On you first run you should start a container manually:

~~~bash
export gitlab_version="gitlab-10.6.2-ce.0"

docker run -d \
	--name ${gitlab_version} \
	--hostname gitlab.mydomain.com \
	--publish 443:443 \
	--publish 80:80 \
	--publish 22:22 \
	--publish 4567:4567 \
	--restart always \
	--volume /storage/docker_containers_data/${gitlab_version}/config:/etc/gitlab \
	--volume /storage/docker_containers_data/${gitlab_version}/logs:/var/log/gitlab \
	--volume /storage/docker_containers_data/${gitlab_version}/data:/var/opt/gitlab \
    --volume /storage/git-data:/mnt/git-data \
	--volume /git_backups:/var/opt/gitlab/backups \
	--volume /etc/localtime:/etc/localtime \
	--volume /storage/gitlab-registry:/var/opt/gitlab/gitlab-rails/shared/registry \
	gitlab/gitlab-ce:$(echo $gitlab_version | sed -e 's/gitlab-//')


$ docker ps

CONTAINER ID        IMAGE                          COMMAND             CREATED             STATUS                PORTS                                                                                  NAMES
45bdd74ebbb5        gitlab/gitlab-ce:10.6.2-ce.0   "/assets/wrapper"   4 days ago          Up 4 days (healthy)   0.0.0.0:22->22/tcp, 0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:4567->4567/tcp   gitlab-10.6.2-ce.0
~~~

<b>Please note!</b><br />
You can stop and remove current container and start a new one in any time and do not loose any data.<br />
All the data a stored on mounted volumes.

## Gitlab CI runner installation

1. Install the docker on vm for runner [Docker installation](#Docker installation)
2. Install the runner:

~~~bash
$ ssh gitlab-ci-runner01.mydomain.com
$ yum install gitlab-ci-multi-runner
~~~

3. Configure the runner:

~~~bash
$ vi /etc/gitlab-runner/config.toml

concurrent = 4
check_interval = 0

[[runners]]
  name = "gitlab-ci-runner01.mydomain.com"
  url = "https://gitlab.mydomain.com/"
  token = "880788aacd70f4e62588c9b37fc7ee"
  executor = "docker"
  shell = "bash"
  environment = ["http_proxy=http://172.29.50.100:8080", "https_proxy=http://172.29.50.100:8080", "no_proxy='127.0.0.1, localhost, *.mydomain.com'", "GIT_SSL_NO_VERIFY=true", "ad_user_for_external_api_calls=ta_jenkins_portal", "ad_password_for_external_api_calls=kXMkYYd11"]
  [runners.docker]
    tls_verify = false
    image = "gitlab.mydomain.com:4567/po/ci-tests:latest"
    privileged = true
    disable_cache = false
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache", "/opt/docker_cache/ivy2/cache:/root/.ivy2/cache", "/opt/docker_cache/m2:/root/.m2"]
    shm_size = 0
  [runners.cache]
    Type = "s3"
    ServerAddress = "gitlab-ci-cache.mydomain.com:8000"
    AccessKey = "accessKey1"
    SecretKey = "verySecretKey1"
    BucketName = "runner"
    BucketLocation = ""
    Insecure = true
    Path = "gitlab/ci/cache"
    # To share the cache between two or more Runners, set the Shared flag to true.
    Shared = true

$ systemctl restart gitlab-runner.service
$ systemctl status gitlab-runner.service
~~~

ad_user_for_external_api_calls - variable is used to automatically upload artifcats to confluence in some jobs.


4. Create dirs for cache volumes
~~~bash
mkdir -p /cache \
mkdir -p /opt/docker_cache/ivy2/cache \
mkdir -p /opt/docker_cache/m2
~~~

5. Add gtilab certificates key chain to allow runner auth to gitlab registry. <br/>
You should use cert from https://gitlab.mydomain.com/po/ansible-gitlab-provisioning/blob/master/roles/post_configuring/files/ssl/certificate_bundle.cer
~~~bash
$ ls -1 /etc/gitlab-runner/certs/
gitlab.mydomain.com.crt

Also I suggest to add the chanin to os keystore

$ vi /etc/pki/ca-trust/source/anchors/gitlab.mydomain.com.crt
$ update-ca-trust
~~~

6. Register the new runner

Take a registration code from gtilab UI<br />
On vm with runner run command below:

~~~bash
$ gitlab-runner register

choose:
https://gitlab.mydomain.com/
docker
gitlab.mydomain.com:4567/po/ci-tests:latest
~~~

7. Add script for rotating old containers

~~~bash
$ vi /opt/scripts/clear-docker-cache.sh

#!/usr/bin/env bash


set -e

gitlab-ci-multi-runner stop >/dev/null 2>/dev/null

docker version >/dev/null 2>/dev/null

echo Clearing docker cache...

CONTAINERS=$(docker ps -a -q \
             --filter=status=exited \
             --filter=status=dead \
             --filter=label=com.gitlab.gitlab-runner.type=cache)

if [ -n "${CONTAINERS}" ]; then
    docker rm -v ${CONTAINERS}
fi

gitlab-ci-multi-runner start >/dev/null 2>/dev/null
~~~

8. Add cron task

~~~bash
$ sudo crontab -e
0 23 * * 0   /opt/scripts/clear-docker-cache.sh
~~~

[ToDo] - Add task to install and configure runner with ansible


## Amazon S3-compatible object storage server installation
Project repo: https://github.com/scality/S3

1. Install and configure docker on gitlab-ci-cache.mydomain.com [Docker installation](#docker-installation) <br />
2. Start a container with s3 service:

~~~bash
$ docker run -d --name s3-for-gitlabci -p 8000:8000 \
	-e ENDPOINT=gitlab-ci-cache.mydomain.com \
	-e SCALITY_ACCESS_KEY_ID=accessKey1 \
	-e SCALITY_SECRET_ACCESS_KEY=verySecretKey1 \
	-e S3BACKEND=mem \
	-e LOG_LEVEL=trace \
	--restart=always \
	scality/s3server:mem-latest
~~~

To check it just use


# Gitlab Configuration

## Config files and directories

Gitlab data files

~~~bash
/storage/
├── docker_containers_data
├── git-data
└── gitlab-registry
~~~

Dir with containers data

~~~bash
/storage/docker_containers_data/gitlab-10.6.2-ce.0/
├── config
├── data
└── logs
~~~

Dir for backups

~~~bash
$ ls -1 /git_backups/

1522621903_2018_04_02_10.6.2_gitlab_backup.tar
1522708338_2018_04_03_10.6.2_gitlab_backup.tar
1522794737_2018_04_04_10.6.2_gitlab_backup.tar
1522881087_2018_04_05_10.6.2_gitlab_backup.tar
1522967467_2018_04_06_10.6.2_gitlab_backup.tar
gitlab_backup_CONFIGS-2018_04_01.tar.gz
gitlab_backup_CONFIGS-2018_04_02.tar.gz
gitlab_backup_CONFIGS-2018_04_03.tar.gz
gitlab_backup_CONFIGS-2018_04_04.tar.gz
gitlab_backup_CONFIGS-2018_04_05.tar.gz
gitlab_backup_CONFIGS-2018_04_06.tar.gz
~~~

Gitlab data dir is specified in gitlab.rb
~~~bash
git_data_dirs({ "default" => { "path" => "/mnt/git-data" } })
~~~

Gitlab HTTPS support configuration.<br />
Nginx from omnibus gitlab container is offloading https traffic
~~~bash
$ vi gitlab.rb

external_url 'https://gitlab.mydomain.com'
nginx['redirect_http_to_https'] = true
~~~

All the required dirs a mounted during container start as volumes

~~~yaml
### Start new container
- name: Starting new container
  docker_container:
    name: "{{ next_img }}"
    image: "gitlab/gitlab-ce:{{ gitlab_img }}"
    hostname: "{{ hosti }}"
    restart_policy: always
    ports:
      - "22:22"
      - "80:80"
      - "443:443"
      - "4567:4567"
    volumes:
      - "/storage/docker_containers_data/{{ next_img }}/config:/etc/gitlab"
      - "/storage/docker_containers_data/{{ next_img }}/logs:/var/log/gitlab"
      - "/storage/docker_containers_data/{{ next_img }}/data:/var/opt/gitlab"
      - /storage/git-data:/mnt/git-data
      - /git_backups:/var/opt/gitlab/backups
      - /etc/localtime:/etc/localtime
      - /storage/gitlab-registry:/var/opt/gitlab/gitlab-rails/shared/registry
  when: mirroring|success
~~~

## Registry configuration
Gitlab container registry configuration is shown below:

~~~bash
$ vi gitlab.rb

registry_external_url 'https://gitlab.mydomain.com:4567' <br />
gitlab_rails['gitlab_default_projects_features_container_registry'] = false <br />
registry_nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.mydomain.com.crt" <br />
registry_nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.mydomain.com.key" <br />
~~~

## Backuping and rotating workflow

We have two types of backups:
- full snapshots of gitlab vm --> are stored for 2 days
- our own backups of gitlab application on separate mounted drive --> are stored for 5 days

~~~bash
$ crontab -l
5 1 * * * docker exec -i gitlab-10.6.2-ce.0 /bin/bash -c "/bin/tar czf /var/opt/gitlab/backups/gitlab_backup_CONFIGS-`date "+\%Y_\%m_\%d"`.tar.gz /etc/gitlab && /usr/bin/find /var/opt/gitlab/backups/ -type f -iname "gitlab_backup_CONFIGS-*" -mtime +5 | xargs rm -f"
#Ansible: Gitlab rake backup
10 1 * * * docker exec -t gitlab-10.6.2-ce.0 gitlab-rake gitlab:backup:create SKIP=artifacts,builds,uploads,registry
~~~

Our own backups are stored on mounted into docker volume, on host machine it is a separate mounted disk: 
~~~bash
$ ls -1 /git_backups/
1522364301_2018_03_30_10.1.0_gitlab_backup.tar
1522434879_2018_03_30_10.1.0_gitlab_backup.tar
gitlab_backup_CONFIGS-2018_03_29.tar.gz
gitlab_backup_CONFIGS-2018_03_31.tar.gz
gitlab_backup_CONFIGS-2018_04_01.tar.gz
~~~

? What about configure raid for /git_backups/ https://erikugel.wordpress.com/2010/04/11/setting-up-linux-with-raid-faster-slackware-with-mdadm-and-xfs/

Rotating of backups is regulated in gitlab.rb config file by the following variable in seconds:
~~~bash
gitlab_rails['backup_keep_time'] = 432000
~~~

Backup of configs is made simply by creating an archive and rotating based on mtime value:
~~~bash
5 1 * * * docker exec -i gitlab-10.6.2-ce.0 /bin/bash -c "/bin/tar czf /var/opt/gitlab/backups/gitlab_backup_CONFIGS-`date "+\%Y_\%m_\%d"`.tar.gz /etc/gitlab && /usr/bin/find /var/opt/gitlab/backups/ -type f -iname "gitlab_backup_CONFIGS-*" -mtime +5 | xargs rm -f"
~~~

Provisioning of the cron jobs is implemented by ansible automatically and there is no need to do it manually after update.

## Gitlab pages

[ToDo]

# Maintenance and support

## Docker container operations

- Start stop, tail logs within a container

~~~bash
$ sudo docker exec -ti CONTAINER_NAME bash
root@gitlab:/# gitlab-ctl stop
root@gitlab:/# gitlab-ctl start
root@gitlab:/# gitlab-ctl tail
Ctr+D
~~~

- Tail logs of container from host vm

~~~bash
$ sudo docker logs -f CONTAINER_NAME
~~~

- Reconfigure gitlab in container (required if you have changed configuration, e.g. gitlab.rb)<br />
And you have two options here.

- Reconfigure from host machine:

~~~bash
$ sudo docker restart gitlab
~~~

- Reconfigure withing a container:

~~~bash
$ sudo docker exec -ti CONTAINER_NAME bash
root@gitlab:/# sudo gitlab-ctl reconfigure
~~~

- Connect to postgress databae

~~~bash
$ docker exec -ti CONTAINER_NAME bash
$ su - gitlab-psql
$ psql -h /var/opt/gitlab/postgresql -d gitlabhq
~~~

## Docker cleanups

1. Remove not running containers

~~~bash
$ docker ps -a --filter "status=created" | awk '{ print $1 }' |xargs docker rm
~~~

2. Remove unused images that is not in use by any container anymore 

~~~bash
$ docker image prune -a
~~~

3. Remove containers data dirs from host machine

~~~bash
$ ls -1 /storage/docker_containers_data/
gitlab-10.1.0-ce.0
gitlab-10.1.1-ce.0
gitlab-10.1.2-ce.0
gitlab-10.1.3-ce.0
gitlab-10.1.4-ce.0
gitlab-10.1.5-ce.0
gitlab-10.1.6-ce.0
gitlab-10.1.7-ce.0
gitlab-10.2.0-ce.0
gitlab-10.2.1-ce.0
gitlab-10.2.2-ce.0
gitlab-10.2.3-ce.0
gitlab-10.2.4-ce.0
gitlab-10.2.5-ce.0
gitlab-10.2.6-ce.0
gitlab-10.2.7-ce.0
~~~

### Cleanup old blobs from gitalb registry

~~~bash
# docker exec -it CONTAINER_NAME /opt/gitlab/embedded/bin/registry garbage-collect /var/opt/gitlab/registry/config.yml

Or withing a container:

root@gitlab:/# gitlab-ctl registry-garbage-collect
~~~
There exist some solution for cleanup old blobs but it was not tested yet:<br />
maybe this one works and should be tested https://gitlab.com/gitlab-org/docker-distribution-pruner


## Gitlab CI Runner support

### Cleanup old containers with cache volumes

We have a cron job installed on all runner vms earlier, so there is no need to cleanup it manually.<br />
But in case you need to clean the docker follow the steps below.

~~~bash
Pause the runner via Gitlab UI and then follow the steps below

$ ssh gitlab-ci-runner01.mydomain.com
$ systemctl stop gitlab-runner.service

First remove all containers
$ docker rm `docker ps -aq`

Then remove orphaned volumes
$ docker volume rm $(docker volume ls -qf dangling=true)
$ systemctl start gitlab-runner.service
~~~

## Gitlab container registry for FE tests
All the images used by Core and projects fro CI are stored in https://gitlab.mydomain.com/po/ci-tests/container_registry <br />
How to push or modify existing images you can find thre: https://gitlab.mydomain.com/po/ci-tests/tree/latest

# Update and restore Gitlab

## Update Gitlab

This process is now automated.<br>
You have two options:
- run the commands manually
- run jenkins job

### Run the commands manually

If you are going to run the script manually then you should follow the steps:

~~~bash
Pull the freshest code with job http://jenkins-ci.mydomain.com/jenkins/view/service-jobs/job/ansible_gitlab_provisioning/

$ ssh to jenkins-ci.mydomain.com
$ cd /jobs/ansible_gitlab_provisioning/workspace

Get all available versions
$ export http_proxy=http://172.29.50.100:8080 https_proxy=http://172.29.50.100:8080
$ curl -s https://registry.hub.docker.com/v1/repositories/gitlab/gitlab-ce/tags | python -mjson.tool | grep name | sort

Modify file with list of versions to update
$ vi update.list

Run the command
$ while read gitlab_img; do \
	echo $gitlab_img; \
	ansible-playbook -i hosts/hosts.ini gitlab_updating.yml -e "hosti=gitlab.mydomain.com gitlab_img=$gitlab_img"; \
	sleep 60; \
done < update.list
~~~

### Run jenkins job

Please NOTE!!! Be careful and follow jenkins job console logs for versions update<br/ >
To run jenkins job go to <http://jenkins-ci.mydomain.com/jenkins/view/service-jobs/job/ansible_gitlab_provisioning/>
select versions to update and press build button.

## Restore Gitlab

The official documentain is there: https://docs.gitlab.com/ee/raketasks/backup_restore.html<br />
Please note that we are using Omnibus installation

Here we have two choices:
- Restore to certain version of the docker container (most usable)
- Full restore with all repository data

<b>Restore to certain version of the docker container</b><br />

This will return us to certain state of postgres db and gitlab configuration

1. Make sure the container is stopped
2. Start the required container with command described there [Run Gitlab in docker](#run-gitlab-in-docker)
3. Check logs if there no errors:

~~~bash
$ sudo docker logs -f CONTAINER_NAME
~~~

<b>Full restore with all repository data</b>

1. Make sure the container is stopped
2. Start the container of certain version. The container version should match your backup version. [Run Gitlab in docker](#run-gitlab-in-docker)
3. Stop gitlab inside a container:

~~~bash
root@gitlab:/# sudo gitlab-ctl stop unicorn
root@gitlab:/# sudo gitlab-ctl stop sidekiq

Verify
root@gitlab:/# sudo gitlab-ctl status
~~~

4. Make sure the required backup archive is visible within a container:
~~~bash
$ docker exec -ti CONTAINER_NAME bash -c "ls -1 /var/opt/gitlab/backups/"
1522621903_2018_04_02_10.6.2_gitlab_backup.tar
~~~ 

5. Restore all gitlab data from archive with rake:

~~~bash
# This command will overwrite the contents of your GitLab database!
$ docker exec -ti CONTAINER_NAME bash -c "gitlab-rake gitlab:backup:restore BACKUP=1522621903_2018_04_02_10.6.2"
~~~

6. Next, restore /etc/gitlab/gitlab-secrets.json if necessary.

7. Restart and check GitLab:

~~~bash
$ sudo docker exec -ti CONTAINER_NAME bash
root@gitlab:/# gitlab-ctl restart
root@gitlab:/# sudo gitlab-rake gitlab:check SANITIZE=true
~~~

If there is a GitLab version mismatch between your backup tar file and the installed version of GitLab, the restore command will abort with an error.<br />
Run the correct GitLab container version and try again [Run Gitlab in docker](#run-gitlab-in-docker)

# Troubleshooting

- Error produced by ansible: Either path or file obj needs to be provided.
____________
Resource: https://www.reddit.com/r/ansible/comments/7cq9q5/issue_with_docker_container_error_pulling_image/

Fix:
on your ansible server install the docker-python.x86_64


- If your docker do not trust CA then add it:

~~~bash
$ vi /etc/docker/certs.d/pt_cas.crt
$ systemctl restart docker
~~~

[ToDo]

# Usefull stuff

~~~bash
If you need to run certain task
$ ansible-playbook playbook.yml --start-at-task="install packages" <br />

If you need to run playbook with confirmation of each step/task:
$ ansible-playbook playbook.yml --step <br />
~~~



# What to do on nearest update

1. Enable in gtilab.rb

~~~bash
##! **Recommended by: https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html
##!                   https://cipherli.st/**
nginx['ssl_protocols'] = "TLSv1 TLSv1.1 TLSv1.2"
~~~

2. Fix https issue - Fixed

2.1 Rollback Anton Nazin's change in nginx conf i docker:

~~~bash
root@gitlab:/# vi /var/opt/gitlab/nginx/conf/gitlab-http.conf +83

#  add_header Strict-Transport-Security "max-age=31536000";
~~~

2.1 We need a DNS zone for *.gitlab.mydomain.com, to be able to serve on the same ip requests with HOST header like:
gitlab.mydomain.com,, vitalii.rakovtsii.gitlab.mydomain.com

2.2 We need a single TLS certificate for *.gitlab.mydomain.com and gitlab.mydomain.com with Alternative Name record inside (certificate+private key), Multidomain/wildcard certificate.

X509v3 Subject Alternative Name: 
       DNS:*.gitlab.mydomain.com, DNS:gitlab.mydomain.com

This is needed to resolve chrome's issue:  "The certificate for this site does not contain a Subject Alternative Name extension containing a domain name or IP address."

https://www.digicert.com/subject-alternative-name.htm

SD ticket: IN0801458

3. reconfigure docker too use devicemapper driver or overlay, read the suggestions for centos 7.4
~~~bash
$ vi /etc/docker/daemon.json
{
  "storage-driver": "devicemapper",
  "storage-opts": [
    "dm.directlvm_device=???",
    "dm.thinp_percent=95",
    "dm.thinp_metapercent=1",
    "dm.thinp_autoextend_threshold=80",
    "dm.thinp_autoextend_percent=20",
    "dm.directlvm_device_force=false"
  ]
}

Or:

{
    "data-root": "/mnt/docker_data",
    "storage-driver": "overlay"
}

systemctl daemon-reload
systemctl restart docker

~~~

4. Mount a dir for gitlab pages as a volume:
5. Add to Readme about Gitlab pages configuration in .rb file
6. How to test s3 from client: use aws cli agent, Please note that to be able to store cahce you need to create a bucket on s3 manualy!