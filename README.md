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
- Certificates must be generated for the new environment as described [here](https://gitlab.et-scm.com/recs/recs-util-nomad#certificates-generation).
- The [Puppet configuration](https://gitlab.et-scm.com/recs/recs-util-puppet-config/tree/master/hiera/instance_classification/prod/roles) needs to be manually updated with the content of the EYAML-encrypted files generated in the `tls` folder. On Mac OS a convenient way of doing this is to copy the files in the paste bin (clipboard) using ```pbcopy < puppet-conf-server.yaml``` and then pasting it at the end of the corresponding config file.
- Then the Puppet config needs to be pushed to Gitlab (```git push origin master```) and the Puppet server made to load the new files:
```bash
> ssh centos@10.188.240.244 sudo /usr/bin/r10k_with_lock deploy environment -p -v debug
```



```
    * Update Nomad config with UUID

    * Copy Puppet keys from /etc/puppetlabs/puppet/keys, upload to S3 keys/
    * Upload CA files to S3 ca/
5. Generate certificates
```
> cd recs-util-nomad/tls
> make veryclean
> make import ENV=live  # Import Root CA & Puppet certificates and keys
> make all    ENV=live  # Create all certificates and EAMML-encrypted config files
> make export ENV=live  # Upload certificates to S3
```

6. Update Puppet config manually pbcopy < puppet-conf-server.yaml etc.
Push config to repo and update Puppet server

```bash
> git push origin master
> ssh centos@<puppet_server_ip> sudo /usr/bin/r10k_with_lock deploy environment -p -v debug
```

7. Create Nomad cluster

```bash
> cd recs-util-nomad/terraform
> awsudo -u recs-live make init ENV=live
```
# Troubleshooting

Most likely the Puppet config is incorrect
Check that the config is consistent between servers and worker nodes

```bash
> jq . < /etc/consul.d/config.json
> jq . < /etc/nomad.d/config.json
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


