# Jenkins server deployment walk-through
This document details the step-by-step actions to deploy a Jenkins server in a new Nomad cluster, using Docker containers for both the master server and the execution agents. It currently runs against the Recommenders Live account, although moving to a different account should be fairly straightforward.

## Terraform configuration
All actions against AWS are handled by Terraform and (mostly) implement the Core Engineering Team's [Compliance Framework](https://confluence.cbsels.com/display/TRCE/TIO+RP+Platform+%7C+Compliance+Framework) requirements.

### Account bootstrap and VPC creation
This step configures a new AWS account managed by Terraform and creates the basic components like VPC, subnets, Route53 zones, etc. The corresponding code is in the [recs-core-infra](https://gitlab.et-scm.com/recs/recs-core-infra) repository. It comprises 3 sub-folders to be run in this order: `bootstrap`, `util`, `main`. Note that these folders share the `Makefile`, `dev.tfvars`, `live.tfvars` and `variables.tf` files through symbolic links.

#### Account bootstrap
The full bootstrap procedure is described in the `recs-core-infra` [README](https://gitlab.et-scm.com/recs/recs-core-infra#deployment) file. It creates among others the S3 bucket for storing Terraform state files, as well as a [KMS key](https://gitlab.et-scm.com/recs/recs-core-infra/blob/master/bootstrap/kms.tf) that will subsequently be used to encrypt secrets.

```bash
> cd recs-core-infra/bootstrap
> awsudo -u recs-live make init ENV=live
> awsudo -u recs-live make plan ENV=live
> awsudo -u recs-live make apply
```

#### Utility VPC creation
This step creates the Utility VPC which will host the Nomad cluster. It creates a [Bastion](https://gitlab.et-scm.com/elsevier-core-engineering/rp-terraform-bastion) and a [Puppet](https://gitlab.et-scm.com/elsevier-core-engineering/rp-terraform-puppetserver) server using standard CE Terraform modules.

Before running Terraform, a `puppetdb-pass` entry needs to be present in the `recs-util-secrets` AWS Secrets Manager secret to store the Puppet DB password as explained [here](https://gitlab.et-scm.com/recs/recs-util-nomad#recommenders-utility-vpc-configuration). Also Terraform needs to be able to automatically launch an SSH connexion to the Puppet server through Bastion as detailed [here](https://gitlab.et-scm.com/recs/recs-core-infra#puppet-server).

```bash
> cd recs-core-infra/util
> awsudo -u recs-live make init ENV=live
> awsudo -u recs-live make plan ENV=live
> awsudo -u recs-live make apply
```

#### Main VPC creation
This phase only creates the Route53 entries and a couple S3 buckets for now.

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
- The [Puppet configuration](https://gitlab.et-scm.com/recs/recs-util-puppet-config/tree/master/hiera/instance_classification/prod/roles) needs to be manually updated with the content of the EYAML-encrypted files generated in the `tls` folder by the previous step. On Mac OS a convenient way of doing this is to copy a file to the paste bin (clipboard) using ```pbcopy < puppet-conf-server.yaml``` for instance and then pasting it at the end of the corresponding config file (`recs-util-puppet-config/hiera/instance_classification/prod/roles/container_server.yaml` in this case).
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
The Consul and Nomad servers GUI can then be accessed through an SSH tunnel to the Bastion server, which is brought up as follows:

```bash
> cd recs-util-nomad/nomad
> make connect ENV=live     # Set up an SSh tunnel to the servers
> make gui ENV=live         # Launch the Consul and Nomad GUIs in a new Web Browser window
```

#### Troubleshooting
The most likely reason for which the Consul or Nomad servers would fail to start is that the Puppet configuration created above (in `recs-util-puppet-config`) is incorrect. In order to check this, SSH to one of the server or worker nodes and inspect the content of the `/etc/consul.d/config.json` and `/etc/nomad.d/config.json` files as described below. Check that the values are correct and consistent between nodes. If necessary correct the Puppet configuration and repeat the steps above.

```bash
> ssh admin@<cluster_instance_ip>
> sudo -s
> apt-get -y install jq
> jq . < /etc/consul.d/config.json
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
When the configuration is correct, the following CLI commands when ran on a server node should give a result similar to the following:

```bash
> ssh admin@<cluster_server_ip>
> sudo -Es consul members
Node               Address              Status  Type    Build  Protocol  DC         Segment
ip-10-188-240-156  10.188.240.156:8301  alive   server  1.1.0  2         us-east-1  <all>
ip-10-188-240-206  10.188.240.206:8301  alive   server  1.1.0  2         us-east-1  <all>
ip-10-188-241-241  10.188.241.241:8301  alive   server  1.1.0  2         us-east-1  <all>
ip-10-188-240-137  10.188.240.137:8301  alive   client  1.1.0  2         us-east-1  <default>
ip-10-188-241-185  10.188.241.185:8301  alive   client  1.1.0  2         us-east-1  <default>

> sudo -Es nomad server members
Name                       Address         Port  Status  Leader  Protocol  Build  Datacenter  Region
ip-10-188-240-156.us-east  10.188.240.156  4648  alive   false   2         0.8.3  us-east-1   us-east
ip-10-188-240-206.us-east  10.188.240.206  4648  alive   true    2         0.8.3  us-east-1   us-east
ip-10-188-241-241.us-east  10.188.241.241  4648  alive   false   2         0.8.3  us-east-1   us-east

> sudo -Es nomad node status
ID        DC         Name               Class    Drain  Eligibility  Status
c4950737  us-east-1  ip-10-188-240-137  private  false  eligible     ready
7eb9690c  us-east-1  ip-10-188-241-185  private  false  eligible     ready
```

### Jenkins deployment
At that point the Nomad cluster is up and ready to be used. The next step is to deploy the Jenkins server itself as a Nomad job in it.

#### Creating Docker images
Docker images need to be created for the Jenkins master and agents, and published to the corresponding AWS ECR repositories. This is done with the following commands:

```bash
> cd recs-util-nomad/docker/jenkins-master
> make pull                 # Retrieve latest version of the base Docker image
> make publish ENV=live     # Build and push Jenkins master image to ECR

> cd recs-util-nomad/docker/jenkins-agent-terraform
> make pull
> make publish ENV=live
etc
```
Note that the [Jenkins configuration files](https://gitlab.et-scm.com/recs/recs-util-nomad/tree/SDPR-1226/docker/jenkins-master/ref/init.groovy.d) in `recs-utils-nomad/docker/jenkins-master/ref/init.groovy.d` are fairly specific to the Recommenders team and would need to be adapted to a new environment. More about this below.

#### Launching the Jenkins server
Jenkins is launched as a Nomad job as follows. We first launch [Fabio](https://github.com/fabiolb/fabio) which is a small load-balancer for Nomad jobs.
```bash
> cd recs-util-nomad/nomad
> make import ENV=live              # Import certificates required to connect to the cluster
> make deploy ENV=live JOB=fabio    # Launch Fabio
> make deploy ENV=live JOB=jenkins  # Launch Jenkins
```
At that point you should be able to connect to the Jenkins server at `https://jenkins.util.recs.d.elsevier.com`. You can also have a lok at the Fabio routing table at `https://fabio.util.recs.d.elsevier.com/routes`.

### Jenkins configuration
As mentioned above, while the Nomad configuration described up to this point is fairly generic and should be reuseable with few changes, the Jenkins configuraiton itself is fairly specific to the Recommenders environment and would likely need to be extensively modified for a new deployment. The following paragraphs provide some high-level documenation that should help in doing that.

#### Initialisation scripts
When a new Jenkins image is laucnhed, the first thing that happens is that all files in the `/usr/share/jenkins/ref` folder are copied to Jenkins's home `/var/jenkins_home`. Additionally, if a file has the `.override` extension, it replaces its existing counterpart there. The files that are copied include the following directories, described in more detail below:
- `init.groovy.d`: initialisation scripts
- `plugins`: local plugins
- `jobs`: predefined jobs

##### Groovy scripts
When the Jenkins server starts, it first executes all `.groovy` files in the `/var/jenkins_home/init.groovy.d` directory. In our case the files are as follows:
- `g10_security.groovy`: applies a number of security-related settings, including disabling all unsafe master-agent communication protocols;
- `g12_users.groovy`: parses the `/usr/share/jenkins/creds/jenkins_users` file and creates the corresponding local users;
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

