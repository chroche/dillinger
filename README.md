# Jenkins deployment walk-through

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






