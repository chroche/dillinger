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
        *  make veryclean
        *  make import ENV=live
        *  make all ENV=live
        *  make export ENV=liv

6. Update Puppet config manually pbcopy < puppet-conf-server.yaml etc.
    * Update Puppet server 
```
ssh centos@<puppet_ip> sudo /usr/bin/r10k_with_lock deploy environment -p -v debug
```


