# Work with Vault policies

# Overview
The following tutorial walks through using Vault to configure policies and encryption.

## Objectives 
By the end of this tutorial, you should be able to:
1. Create a Vault policy to limit access to a specific path
2. Use the Transit backend as an "encryption as a service"

## Policies
In the previous labs we have used the root policy to interact with the API. This is something that should never be done in production. The following steps set up a policy that only has read access. 

Add a new config file called `app-policy.json` to "vault/policies":
```json
{
  "path": {
    "kv/data/app/*": {
      "policy": "read"
    }
  }
}
```

Enter the Vault container:
```bash
 docker-compose exec vault bash
```

Create a new policy:
```
bash-5.1# vault policy write app /vault/policies/app-policy.json

Success! Uploaded policy: app
```

Then create a new token: 
```
bash-5.1# vault token create -policy=app

Key                  Value
---                  -----
token                s.ZOUMx3RIhVRhI4ijlZg8KXRQ
token_accessor       TT53xOxbIfGjI7l4392gjXcg
token_duration       768h
token_renewable      true
token_policies       ["app" "default"]
identity_policies    []
policies             ["app" "default"]
```

In another terminal window set the `VAULT_TOKEN` environment variable with the new token:
```
export VAULT_TOKEN=<token from step above>
```

Try to read the `foo` secret that we created earlier: 
```
curl \
    -H "X-Vault-Token: $VAULT_TOKEN" \
    -X GET \
    http://127.0.0.1:8200/v1/kv/data/hello
```

You should not have the correct permissions to view that secret:
```
{
  "errors":[
    "1 error occurred:\n\t* permission denied\n\n"
  ]
}
```

Why can't we even read it? Jump back to the policy config in `vault-config.json` `kv/data/app/*` indicates that the policy can only read from the `app` path.

Back within the bash session in the container, add a new secret to the `app/test` path:
```
bash-5.1# vault kv put kv/app/test ping=pong

Key              Value
---              -----
created_time     2021-09-08T18:40:35.2694047Z
deletion_time    n/a
destroyed        false
version          1
```


You should be able to view the secret using the token associated with the `app` policy:
```
curl \
    -H "X-Vault-Token: $VAULT_TOKEN" \
    -X GET \
    http://127.0.0.1:8200/v1/kv/data/app/test
```

## Encryption as a Service
Let's review the Transit backend, which can be used as an "encryption as a service" for:

* Encrypting and decrypting data "in-transit" without storing it inside Vault
* Easily integrating encryption into your application workflow

Back within the bash session in the container, enable the Transit engine:
```
bash-5.1# vault secrets enable transit

Success! Enabled the transit secrets engine at: transit/
```

Configure a named encryption key:
```
bash-5.1# vault write -f transit/keys/foo

Success! Data written to: transit/keys/foo
```

Encrypt a string:
```
bash-5.1# vault write transit/encrypt/foo plaintext=$(base64 <<< "my precious")
Key            Value
---            -----
ciphertext     vault:v1:UhhlG+GEpoDsKwCrLuUBunJKmExMd3GVZuCDdG0/PyvcwdM2z/1IXQ==
```

Decrypt the string from above:
```
bash-5.1# vault write transit/decrypt/foo ciphertext=vault:v1:UhhlG+GEpoDsKwCrLuUBunJKmExMd3GVZuCDdG0/PyvcwdM2z/1IXQ==
Key          Value
---          -----
plaintext    bXkgcHJlY2lvdXMK
```

To confirm the decrypted string is correct use `base64` to decode it. 
```
base64 -d <<< "bXkgcHJlY2lvdXMK"

my precious
```

## Congrats
