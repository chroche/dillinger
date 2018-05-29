# Jenkins deployment walk-through

This document details the step-by-step actions taken to deploy a Jenkins server in a new Nomad cluster, using Docker containers for both the master server and the execution agents. It runs against the Recommenders Live account, although moving to a different account should be fairly straightforward.

## Terraform configuration

All actions against AWS are handled by Terraform and (mostly) implement the Core Engineering Team's [Compliance Framework](https://confluence.cbsels.com/display/TRCE/TIO+RP+Platform+%7C+Compliance+Framework) requirements.

### Account bootstrap and VPC creation
This step set up a new AWS account to be managed by Terraform and create the basic components like VPC, subnets, Route53 zones, etc. The corresponding code is in the [recs-core-infra](https://gitlab.et-scm.com/recs/recs-core-infra) repository. It comprises 3 sub-folders to be run in this order: `bootstrap`, `util`, `main`. Note that these folders share the `Makefile`, `dev.tfvars`, `live.tfvars` and `variables.tf` files through symbolic links.
#### Account bootstrap
The bootstrap procedure is described in the [README](https://gitlab.et-scm.com/recs/recs-core-infra#deployment) file. It creates the S3 bucket for storing Terraform state files, as well as a [KMS key](https://gitlab.et-scm.com/recs/recs-core-infra/blob/master/bootstrap/kms.tf) that will subsequently be used to encrypt secrets.
```bash
> cd recs-core-infra/bootstrap
> awsudo -u recs-live make init ENV=live
> awsudo -u recs-live make plan ENV=live
> awsudo -u recs-live make apply
```
 #### Utility VPC creation
 This creates the Utility VPC which will host the Nomad cluster. It creates a Bastion and a Puppet server using CE Terraform modules.
 Before running Terraform, you'll need to create a `puppetdb-pass` entry in the `recs-util-secrets` AWS Secrets Manager secret to store the Puppet DB password as explained [here|https://gitlab.et-scm.com/recs/recs-util-nomad#recommenders-utility-vpc-configuration].
 Also Terraform will need to be able to automatically launch an SSH connexion to the Puppet server through Bastion as detailed [here|https://gitlab.et-scm.com/recs/recs-core-infra#puppet-server].
```bash
> cd recs-core-infra/util
> awsudo -u recs-live make init ENV=live
> awsudo -u recs-live make plan ENV=live
> awsudo -u recs-live make apply
```
#### Main VPC creation
This phase will only be used to create the Route53 entries and a couple S3 buckets for now.
```bash
> cd recs-core-infra/main
> awsudo -u recs-live make init ENV=live
> awsudo -u recs-live make plan ENV=live
> awsudo -u recs-live make apply
```
### Nomad cluster creation
This phase creates the Nomad cluster in the Utility VPC.
#### Cluster configuration
As a first step, the cluster configuration needs to be updated as follows.
- A unique cluster ID must be generated, typically with ```python -c 'import uuid; print str(uuid.uuid4())'```, and updated in the [Terraform variables](https://gitlab.et-scm.com/recs/recs-util-nomad/blob/master/terraform/live.tfvars#L25) and Puppet configuration ([servers](https://gitlab.et-scm.com/recs/recs-util-puppet-config/blob/master/hiera/instance_classification/prod/roles/container_server.yaml#L26) and [worker nodes](https://gitlab.et-scm.com/recs/recs-util-puppet-config/blob/master/hiera/instance_classification/prod/roles/container_server.yaml#L26)).
- Certificates must be generated for the new environment and uploaded to S3 as described [here](https://gitlab.et-scm.com/recs/recs-util-nomad#certificates-generation).
- The [Puppet configuration](https://gitlab.et-scm.com/recs/recs-util-puppet-config/tree/master/hiera/instance_classification/prod/roles) needs to be manually updated with the content of the EYAML-encrypted files generated in the `tls` folder by the previous step. On Mac OS a convenient way of doing this is to copy a file to the paste bin (clipboard) using ```pbcopy < puppet-conf-server.yaml``` and then pasting it at the end of the corresponding config file (`recs-util-puppet-config/hiera/instance_classification/prod/roles/container_server.yaml` in this case).
- Then the new Puppet config needs to be pushed to Gitlab with
```bash
> git commit -am 'Nomad config for Live'
> git push origin master
```
- and the Puppet server made to load the new files:
```bash
> ssh centos@<puppet_server_ip> sudo /usr/bin/r10k_with_lock deploy environment -p -v debug
```
#### Cluster creation
Terraform can then be launched to build the cluster:
```bash
> cd recs-util-nomad/terraform
> awsudo -u recs-live make init ENV=live
> awsudo -u recs-live make plan ENV=live
> awsudo -u recs-live make apply
```
The Consul and Nomad servers GUI can then be accessed throughv an SSH tunnel to the Bastion server, which is autoamted as follows:
```bash
> cd recs-util-nomad/nomad
> make gui ENV=live
```
#### Troubleshooting
The most likely reason for which the Consul or Nomad servers would fail to start is that the Puppet configuration created above is incorrect.
In order to check this, SSH to one of the server or worker nodes and inspect the content of the `/etc/consul.d/config.json` and `/etc/nomad.d/config.json` files as described below. Check that the values are correct and consistent between nodes. If necessary correct the Puppet configuration and repeat the steps above.
```bash
> ssh admin@<cluster_instance_ip>
> sudo -s
# apt-get -y install jq
# jq . < /etc/consul.d/config.json
```
```json
{
  "acl_datacenter": "us-east-1",
  "acl_default_policy": "deny",
  "acl_down_policy": "extend-cache",
  "acl_master_token": "e60d406f-6c3c-48de-8948-d0891d0e28b9",
  "acl_token": "e60d406f-6c3c-48de-8948-d0891d0e28b9",
  "advertise_addr": "10.188.240.241",
  "bind_addr": "0.0.0.0",
  "ca_file": "/etc/consul.d/tls/recs-ca.crt",
  "cert_file": "/etc/consul.d/tls/consul-server.crt",
  "client_addr": "0.0.0.0",
  "data_dir": "/opt/consul",
  "datacenter": "us-east-1",
  "disable_remote_exec": true,
  "enable_syslog": true,
  "encrypt": "jniUSnfs+X14gKOUAaXzFg==",
  "key_file": "/etc/consul.d/tls/consul-server.key",
  "log_level": "INFO",
  "ports": {
    "http": -1,
    "https": 8500
  },
  "retry_join": [
    "provider=aws region=us-east-1 addr_type=private_v4 tag_key=ClusterName tag_value=container-f1fdef65-495e-4f99-8ab5-98dc59faffec"
  ],
  "server": true,
  "verify_incoming_https": false,
  "verify_incoming_rpc": true,
  "verify_outgoing": true
}
```
### Jenkins deployment

#### Creating Docker images
Docker images needs to be created for the Jenkins master and agents, and publish to the AWS ECR repositories. This is done with the following:
```bash
> cd recs-util-nomad/docker/jenkins-master
> make pull              # Retrieve latest version of base Docker image
> make publish ENV=live

> cd recs-util-nomad/docker/jenkins-agent-terraform
> make pull
> make publish ENV=live
etc
```
Note that the [Jenkins configuration files](https://gitlab.et-scm.com/recs/recs-util-nomad/tree/SDPR-1226/docker/jenkins-master/ref/init.groovy.d) in `recs-utils-nomad/docker/jenkins-master/ref/init.groovy.d` are fairly specific to the Recommenders team and would need to be adapted to a new environment.
#### Launching the Jenkins server
Jenkins is launched as a Nomad job as follows. We first launch [Fabio](https://github.com/fabiolb/fabio) which is a small load-balancer for Nomad jobs.
```bash
> cd recs-util-nomad/nomad
> make import ENV=live              # Import certificates required to connect to the cluster
> make deploy ENV=live JOB=fabio    
> make deploy ENV=live JOB=jenkins
```
At that point you should be able to connect to the Jenkins server at `https://jenkins.util.recs.d.elsevier.com`.
### Jenkins configuration
#### Initialisation scripts
When a new Jenkins image is laucnhed, the first thing that happens is that all files in the `/usr/share/jenkins/ref` folder are copied to Jenkins's home `/var/jenkins_home`. Additionally, if a file has the `.override` extension, it replaces its existing counterpart there. The files that are copied include the following directories, described in more detail below:
- `init.groovy.d`: initialisation scripts
- `plugins`: local plugins
- `jobs`: predefined jobs
##### Groovy scripts
Then when the Jenkins server starts, it first executes all `.groovy` files in the `/var/jenkins_home/init.groovy.d` directory. These files are as follows:
- `g10_security.groovy`: applies a number of security-related settings, including disabling all unsafe master-agent communication protocols;
- `g12_users.groovy`: parses the `/usr/share/jenkins/creds/jenkins_users` file and creates the corresponding lcoal users;
- `g14_credentials.groovy`: loads `.txt` and `.id_rsa` files present in the `/usr/share/jenkins/creds` folder and register them with the Jenkins credentials facility;
- `g20_jekins_url.groovy`: sets the public (for end users) and private (for agents) server URLs;
- `g30_shared_libs.groovy`: register the [shared libraries](https://jenkins.io/doc/book/pipeline/shared-libraries/) used for pipelines definition;
- `g32_nomad_cloud.groovy`: creates a Nomad Cloud for the [Nomad plugin](https://github.com/jenkinsci/nomad-plugin);
- `g40_slack.groovy`: allows publishing to Slack;
- `g99_jenkins_save.groovy`: saves the updated Jenkins configuration.
##### Plugins
This installation uses the [Nomad plugin](https://github.com/jenkinsci/nomad-plugin) for scheduling agents on the Nomad cluster. Normally required plugins are listed in the `ref/plugins.txt` file and retrieved automatically at startup with their dependencies  from the official [Jenkins plugins repository](https://plugins.jenkins.io/nomad). However, since the published version of the Nomad plugin is currently only `0.4`, which lacks features like volume mounts required for Docker-capable agents, we had to compile and install a more recent version as follows:
```
> git clone https://github.com/jenkinsci/nomad-plugin
> cd nomad-plugin
> vi pom.xml        # Bump version number to e.g. 0.5.1
> mvn package
> mvn install
```
then copy `target/nomad.hpi` to the `ref/plugins` folder.
##### Jobs
Only one `Seed` job is created there, using a classic XML configuration. It creates all other jobs at startup time by downloading the [recs-util-jenkins-jobs](https://gitlab.et-scm.com/recs/recs-util-jenkins-jobs) repository and executing all `.groovy` files there using the [Job DSL plugin](https://github.com/jenkinsci/job-dsl-plugin).
#### Creating Jenkins jobs
Jenkins jobs are written in [Groovy](http://groovy-lang.org/single-page-documentation.html) using the [Job-DSL](https://github.com/jenkinsci/job-dsl-plugin/wiki) and [Pipeline](https://jenkins.io/doc/book/pipeline) plugins.
Jobs for Recommenders are defined as pipelines in [recs-util-jenkins-jobs](https://gitlab.et-scm.com/recs/recs-util-jenkins-jobs) using  [shared libraries](https://jenkins.io/doc/book/pipeline/shared-libraries/#defining-declarative-pipelines).
