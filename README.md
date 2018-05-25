# Jenkins deployment walk-through

This document details the step-by-step actions to deploy a Jenkins server in a Nomad cluster, using Docker containers both for the master server and the execution agents. It runs against the Recommenders Live account, although moving to a different account should be fairly easy.

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
### Cluster creation
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
### Troubleshooting
The most likely reason for which the Consul or Nomad servers fail to start is that the Puppet configuration created above is incorrect.
In order to check this, SSH to one of the server or worker nodes and inspect the content of the `/etc/consul.d/config.json` and `/etc/nomad.d/config.json` files. Check that the values are consistent between nodes.
```bash
> ssh <nomad_server_ip>
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
  "advertise_addr": "10.149.0.166",
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

8. Build and publish the Docker images
```bash
> cd recs-util-nomad/docker/jenkins-master
> make pull
> make publish ENV=live
> cd recs-util-nomad/docker/jenkins-agent-terraform
> make pull
> make publish ENV=live
```

9. Launch Jenkins
```bash
> cd recs-util-nomad/nomad
> make import ENV=live
> make deploy ENV=live JOB=fabio
> make deploy ENV=live JOB=jenkins
```


