# Install Vault with Docker Compose

# Overview
The following tutorial details how to set up and use Vault to securely store and manage secrets.

We'll start by spinning up a single instance of Vault using Docker Compose. After we have started Vault, we will initialize, unseal and authenticate. Finally we will enable an Audit backend. 

## Objectives 
By the end of this tutorial, you should be able to:

1. Explain what Vault is and why you may want to use it
2. Describe the basic Vault architecture along with dynamic and static secrets, the various backends (storage, secret, auth, audit), and how Vault can be used as an "encryption as a service"
3. Spin up Vault with the Filesystem backend
4. Init and unseal Vault
5. Authenticate against Vault
6. Configure an Audit backend to log all interactions with Vault

## Filesystem Backend 
To get up and running quickly, we'll use the Filesystem backend to store secrets at rest.

> The filesystem backend should only be used for local development or a single-server Vault deployment since it does not support high availability.

```bash
mkdir vault-consul-docker && cd vault-consul-docker
```

Create the following folder structure:

```
└── vault
    ├── config
    ├── data
    ├── logs
    └── policies
```

Add a `Dockerfile` to the "vault" directory:

```dockerfile
FROM alpine:3.14

# set vault version
ENV VAULT_VERSION 1.9.2

# create a new directory
RUN mkdir /vault

# download dependencies
RUN apk --no-cache add \
      bash \
      curl \
      jq \
      ca-certificates \
      wget

# download and set up vault
RUN wget --quiet --output-document=/tmp/vault.zip https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip && \
    unzip /tmp/vault.zip -d /vault && \
    rm -f /tmp/vault.zip && \
    chmod +x /vault

# update PATH
ENV PATH="PATH=$PATH:$PWD/vault"

# add the config file
COPY ./config/vault-config.json /vault/config/vault-config.json

# expose port 8200
EXPOSE 8200

# run vault
ENTRYPOINT ["vault"]
```

In the `vault-consul-docker` directory create `docker-compose.yml with the following:

```yml
version: '3.3'

services:

  vault:
    build:
      context: ./vault
      dockerfile: Dockerfile
    ports:
      - 8200:8200
    volumes:
      - ./vault/config:/vault/config
      - ./vault/policies:/vault/policies
      - ./vault/data:/vault/data
      - ./vault/logs:/vault/logs
    environment:
      - VAULT_ADDR=http://127.0.0.1:8200
      - VAULT_API_ADDR=http://127.0.0.1:8200
    command: server -config=/vault/config/vault-config.json
    cap_add:
      - IPC_LOCK
```

Add a config file called `vault-config.json` to the "vault/config" directory:

```json
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

In the file we configured Vault to the Filesystem backend, defined the listener for Vault, and disabled TLS. This is a simple dev configuration and should NOT be run in Production. 

Now build the image and spin up the container with `docker-compose`. 

```bash
docker-compose up -d --build 
```

Check the logs to confirm there were no errors. 
```bash
docker-compose logs
```

You should see something similar to:

```
Attaching to vault-consul-docker_vault_1
vault_1  | ==> Vault server configuration:
vault_1  |
vault_1  |              Api Address: http://127.0.0.1:8200
vault_1  |                      Cgo: disabled
vault_1  |          Cluster Address: https://127.0.0.1:8201
vault_1  |               Go Version: go1.17.5
vault_1  |               Listener 1: tcp (addr: "0.0.0.0:8200", cluster address: "0.0.0.0:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
vault_1  |                Log Level: info
vault_1  |                    Mlock: supported: true, enabled: true
vault_1  |            Recovery Mode: false
vault_1  |                  Storage: file
vault_1  |                  Version: Vault v1.9.2
vault_1  |              Version Sha: f4c6d873e2767c0d6853b5d9ffc77b0d297bfbdf
vault_1  |
vault_1  | ==> Vault server started! Log data will stream in below:
```

## Initializing and Unsealing
Start a bash session within the container
```bash
docker-compose exec vault bash 
```

Within the shell, initialize Vault 
```bash
bash-5.1# vault operator init -key-shares=1
```

Note the unseal key and root token. These are required to unseal Vault every time the server is restarted. We will also be using the token for access to the Vault. 

Now unseal the Vault using the key
```bash
bash-5.1# vault operator unseal
```

Paste the unseal key into the prompt (there will be no output). After executing you should see the `Sealed` field says `false` like below.

```
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
```

Now that the Vault is unsealed you can authenticate using the root token. 
```bash
vault login
```

You should see something similar to:

```
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.c0kYHWiOTqQvtR8JuSeTz6sZ
token_accessor       3FQJVxOY5C1brzlHHQSFaCdZ
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

> Keep in mind this uses the root policy. In production you'll want to set up policies with different levels of access. We'll look at how to do this in a later lab.


Vault is now unsealed and ready for use.

## Auditing
Before we test out the functionality, let's enable an Audit Device:
```
bash-5.1# vault audit enable file file_path=/vault/logs/audit.log

Success! Enabled the file audit device at: file/
```

You should now be able to view the logs locally in "vault/logs". To test, run the following command to view all enabled Audit Devices:

```
bash-5.1# vault audit list

Path     Type    Description
----     ----    -----------
file/    file    n/a
```

Confirm the request and response is being written to "vault/logs": 
```bash
cat vault/logs/audit.log | jq
```

# Congrats
