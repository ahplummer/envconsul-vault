

NOTE: Much of this is taken from https://testdriven.io/blog/managing-secrets-with-vault-and-consul/ and https://github.com/hashicorp/envconsul.

Note: When seeing a prompt, `$` is understood to be the host system, `#` is understood to be executed from a container.

# Introduction & Concepts
This tuturial will have several outcomes:
1) You will see how envconsul interacts between Vault and a running app that requires environment variables containing secrets.  
2) You will be exposed to concepts such as Vault initialization, unsealing, ACL additions, secrets concepts, and token authentication.
3) You will have two running containers - one running Vault, one running envconsul with a test script.  These containers will be orchestrated and run by `docker-compose`.
4) The Vault file store will be stored in a directory on the host machine, to prevent data loss of secrets.
5) You will see secrets that originate with Vault being passed to a child process using `envconsul`.  

# To run from the repos
1) Get your IP loaded up in `env` file.
2) The command `$ docker-compose up -d --build` will build and run the containers.  You'll then want to ensure that Vault is init'd and unsealed, configured as instructed below.  You can then execute `docker-compose exec envconsul envconsul -upcase -config=/envconsul/config.json testapp.sh`, or simply `env` at the end (instead of testapp.sh).

# Manual Project Setup for this tutorial

* New, empty root working folder
* Add following folders:
  
```
-envconsul
-vault
--config
--data
--logs
--policies
```

## EnvConsul Setup
* Add Dockerfile to `envconsul` directory.
  
```
TODO: Add contents of Dockerfile for EnvConsul, see repo.
```

* Add `testapp.sh` file to `envconsul` directory.
  
```
#!/usr/bin/env bash
cat <<EOT
My gmailcreds read from Vault are:
user    : "${USER}"
password: "${PASSWORD}"
EOT
```

* Add file in `<working>/vault/policies` named `gmailcreds-policy.json` Take note of the trailing slash:
```
{
    "path": {
      "secret/homestuff/gmailcreds*": {
        "policy": "read"
      }
    }
}
```

* Add `.env` file to `envconsul directory.
  
```
HOST_IP=<your host dev local IP>
```

* Add `config.json` to `envconsul` directory. (Optionally, `config.hcl` also.)  
  
  config.json

```
{
    "secret": {
        "no_prefix": true,
        "path": "secret/homestuff/gmailcreds"
    }
}
```

  config.hcl

```
secret {
    no_prefix = true
    path = "secret/homestuff/gmailcreds"
}
```

## Vault Setup

* Add Dockerfile to `vault` dir.

```
TODO: contents of Dockerfile for vault, see repo.
```

* Add `vault-config.js` to `vault\config` folder.
  
```
{
    "backend": {
      "file": {
        "path": "vault/data"
      }
    },
    "listener": {
      "tcp":{
        "address": "0.0.0.0:8200",
        "tls_disable": 1
      }
    },
    "ui": true
}
```

## Docker Compose Setup
* Add docker-compose.yml to root working folder.
  
```
TODO: contents of docker-compose.yml, see repo.
```

* Build and run it: `$ docker-compose up -d --build`


## Init vault (initial setup)
NOTE: Jump into the container `$ docker-compose exec vault bash`, or simply run the command from the host with the preface `$ docker-compose exec vault <container command>`. *You cannot run the 'init' operation from outside the container.*

* Execute `# vault operator init` from inside the container. Take note of the output and paste it somewhere safe:

```
Unseal Key 1: 8K4VSJeCVqJjHMWZdgnlwP7e+XR5/KvsKQs6zLVIarnx
Unseal Key 2: hRgnS3GW3LFz72pis/nJc/gCDNcIvuFgTqZd7KvV4CEc
Unseal Key 3: U9EuA5FT9y1hdf6ts976MjmwNAl4bXQHNEzgbHckqbWC
Unseal Key 4: s4fUmmlutVVOGoHH3Kj0/VnBXNl3k3UV9quM+/nQ7C+B
Unseal Key 5: vtTGJybCc8j/MQgbDa6GtpBVexXgg7wsQMrHuifcg3xB

Initial Root Token: 8426d664-54fe-29ed-617d-574635a9e5ce
...
```

* Unseal Vault by using three of the unseal keys: `# vault operator unseal <key>` (NOTE: If shutting down the container, upon re-entry, it'll be sealed.)
* You can also unseal via the HOST: `$ docker-compose exec vault vault unseal <key>`.
* Login to vault: `# vault login <root token>`
* Setup auditing: `# vault audit enable file file_path=/vault/logs/audit.log` (NOTE: this will use a volume, therefore will persist to host computer, outside of container.)
* Verify auditing is setup: `# vault audit list`
* Go to file system on host at `<working>/vault/logs/audit.log` and review for validation.

### Get secrets installed/verified

* Store static secrets: `# vault kv put secret/homestuff/gmailcreds user=bogus@gmail.com password=supersecret`
* Validate static secrets: `# vault kv get secret/homestuff/gmailcreds` The output:
  
```
====== Data ======
Key         Value
---         -----
password    supersecret
user        bogus@gmail.com
```

### Setup policy & create token for app to use

* Execute `# vault policy write gmailcreds_acl /vault/policies/gmailcreds-policy.json`
* Execute `# vault token create -policy=gmailcreds_acl`. Take note of output:
  
```
Key                  Value
---                  -----
token                88503e0c-d3ca-e344-fd24-57fab7a31f49
token_accessor       6aacdb75-2211-c9bb-215c-fe3bcee7d097
token_duration       768h
token_renewable      true
token_policies       ["default" "gmailcreds_acl"]
identity_policies    []
policies             ["default" "gmailcreds_acl"]
```

### Test the token:

* open new terminal, type `$ export VAULT_TOKEN=<token from above>`
* Curl the data:
  
``` 
curl -H "X-Vault-Token: $VAULT_TOKEN" -X GET http://127.0.0.1:8200/v1/secret/homestuff/gmailcreds
```

The output:

```
{
    "request_id": "26dee453-aa45-6766-7889-0178e3c3c9f7",
    "lease_id": "",
    "renewable": false,
    "lease_duration": 2764800,
    "data": {
        "password": "supersecret",
        "user": "bogus@gmail.com"
    },
    "wrap_info": null,
    "warnings": null,
    "auth": null
}
```

## Execute final test

* Run this from host: `docker-compose exec envconsul envconsul -upcase -config=/envconsul/config.json testapp.sh`

Your output should look like this:

```
2019/02/23 19:21:55.512496 looking at vault secret/homestuff/gmailcreds
My gmailcreds read from Vault are:
user    : "bogus@gmail.com"
password: "supersecret"
```

