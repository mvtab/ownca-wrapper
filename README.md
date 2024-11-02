# ownca-wrapper
Wrapper for a few Ansible x509 modules to ease up the management of CAs.  
Managing public certificates has become very easy nowadays. Managing certificates signed by an own CA, however, still has it's costs. I made this for myself, to ease up the management of CAs and their certs in test environments.

## Requirements
- Ansible - [tutorial here](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html),
- Ansible community.crypto collection.

## Usage
To spin up a demo PKI infrastructure, simply do the following:
```bash
# Certs.
git clone https://github.com/mvtab/ownca-wrapper
cd ownca-wrapper
mkdir certs
ansible-playbook ownca-wrapper
```

To spin up a test nginx container with the freshly created certificates, run the following:
```bash
# Nginx test.
cd nginx-test
docker compose up -d
cat << EOF | sudo tee -a /etc/hosts

# ownca-wrapper BEGIN
127.0.0.1 server0.example.com
127.0.0.1 server1.example.com
127.0.0.1 server2.example.com
127.0.0.1 server3.example.com
127.0.0.1 server4.example.com
127.0.0.1 server5.example.com
127.0.0.1 server6.example.com
# ownca-wrapper END

EOF

# trust RootCA how you know better.

# Browse, for example, https://server0.example.com. Certificate should be instantly trusted and the connection secure.

```

## Configuration
All configurations are to be found under `group_vars/all/all.yaml`.  
All sensitive information should be kept under `group_vars/all/vault.yaml`.  
The directory `certs/` controls the flow of the playbook. If you define a root CA with name "RootCA" and it's respective `certs/RootCA.key` is present, it will not be overwritten. It will be used to sign any referenced certificates. Same for any other file involved: if it's present the playbook will simply let it be and use it. While this brings a lot of freedom, it also creates the possibility of very tricky issues, like mismatching public certificates and private keys.  
> Example:
> In order to recreate a private key, move it away from the `certs/` directory. Same with public keys and full chains. Keep in mind, if you change your private key, you should also remove their public key and full chain, otherwise the public key won't match the private key. Same if you change your pem: you have to change your full chain too. The full chain also has to be renewed when the identity provider changes public certificate. 

### System
All the identities must follow the following scheme:
```yaml
- name: <Ansible name of entity>
  passphrase: "<passphrase for private key>"
  sans:
  - 'DNS:<SAN>'
  key_usage:
  - <cert usages>
  extended_key_usage:
  - <extended cert usages>
  cert_duration: <duration with measure, i.e. 365d>
  private_key_type: <type of private key, i.e. RSA, ECC>
  # If private key type is not "ECC"
  private_key_size: 4096
  # If private key type is "ECC"
  private_key_ecc_curve: secp256r1
  ca: <True/False>
  provider: <who should sign the identity>
  provider_passphrase: "<password of the signer of the identity>"
```
All entities are created in the listed order. An example scheme is built in.  
If you have CAs they should be listed before the identities that need to be signed by that CA.  

### Sensitive information
The sensitive information should be further protected using, for example, ansible-vault:  
```bash
ansible-vault encrypt ./group_vars/all/vault.yaml
```
And then run playbooks so:  
```bash
ansible-playbook ownca-wrapper --ask-vault-pass
```

### Permissions
The certificates get created as owned by the running user, typically not root.  
You might want to correct that.  

## Limitations

### No warranty
Not intensively tested, I wouldn't trust it too much.
