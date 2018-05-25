# Jenkins deployment walk-through

This document details the step-by-step actions to deploy a Jenkins server in a Nomad cluster, using Docker containers both for the master server and the execution agents. It runs against the Recommenders Live account, although moving to a different account should be fairly easy.

## Terraform configuration

All actions against AWS are handled by Terraform and (mostly) implement the Core Engineering Team's [Compliance Framework](https://confluence.cbsels.com/display/TRCE/TIO+RP+Platform+%7C+Compliance+Framework) requirements.

### Account bootstrrap and VPC creation
These steps set up a new AWS account to be managed by Terraform and create the basic components like VPC, subnets, Route53 zones, etc. The corresponding code is in the [recs-core-infra](https://gitlab.et-scm.com/recs/recs-core-infra) repository.
1. Bootstrap account, cf README
2. Create Utility VPC, be careful of being able to connect to Puppet (ssh-add with only one key, .ssh/config set up properly, VPN on)
3. Create main VPC

4. Create secrets	
        * PuppetDB password
        * Slack token
        * Deploy Key
        * > python -c 'import uuid; print str(uuid.uuid4())'
        * > consul encrypt
        * > nomad encrypt

    * Upload secrets to AWS SM
    Secrets created manually
    Files uploaded with
```bash
> aws secretsmanager --profile recs-live create-secret --name recs-util-file-recs-ca.crt --kms-key-id alias/recs/master-live --secret-binary fileb://recs-ca.crt
> aws secretsmanager --profile recs-live create-secret --name recs-util-file-gitlab.id_rsa --kms-key-id alias/recs/master-live --secret-binary fileb://gitlab.id_rsa
> aws secretsmanager --profile recs-live create-secret --name recs-util-file-jenkins_users --kms-key-id alias/recs/master-live --secret-binary fileb://local_users
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


