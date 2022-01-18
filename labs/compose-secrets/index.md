# Vault Secrets

# Overview
The following tutorial details how to create and access secrets in Vault.

## Secrets
There are two types of secrets in Vault: static and dynamic.

1. Static secrets (think encrypted Redis or Memcached) have refresh intervals but they do not expire unless explicitly revoked. They are defined ahead of time with the `Key/Value` backend and then shared.

2. Dynamic secrets are generated on demand. They have enforced leases and generally expire after a short period of time. Since they do not exist until they are accessed, there's less exposure -- so dynamic secrets are much more secure. Vault ships with a number of dynamic backends -- i.e. AWS, Databases, Google Cloud, Consul, and RabbitMQ.


## Static Secrets
Vault can be managed through the CLI, HTTP API, or UI.

### CLI
Still within the bash session in the container, we can create, read, update, and delete secrets. We'll also look at how to version and roll back secrets.

Enable secrets with following command:
```
bash-5.1# vault secrets enable kv

Success! Enabled the kv secrets engine at: kv/
```

Create a new secret with a key of `bar` and value of `precious` within the `kv/foo` path:

```
bash-5.1# vault kv put kv/foo bar=precious

Success! Data written to: kv/foo
```

Read the newly created secret

```
bash-5.1# vault kv get kv/foo

=== Data ===
Key    Value
---    -----
bar    precious
```

Vault supports versioning secrets. This requires enabling version 2 of the Key/Value backend:

```
bash-5.1# vault kv enable-versioning kv/

Success! Tuned the secrets engine at: kv/
```

Update the `foo` key with a new value:

```
bash-5.1# vault kv put kv/foo bar=copper

Key              Value
---              -----
created_time     2021-09-08T18:23:14.4154928Z
deletion_time    n/a
destroyed        false
version          2
```


Now you have two versions of the secret with different values.

Read version 1:
```
bash-5.1# vault kv get -version=1 kv/foo

====== Metadata ======
Key              Value
---              -----
created_time     2021-09-08T18:22:37.2548824Z
deletion_time    n/a
destroyed        false
version          1

=== Data ===
Key    Value
---    -----
bar    precious
```

Vault returns `precious` which is the version 1 value.

Read version 2:

```
bash-5.1# vault kv get -version=2 kv/foo

====== Metadata ======
Key              Value
---              -----
created_time     2021-09-08T18:23:14.4154928Z
deletion_time    n/a
destroyed        false
version          2

=== Data ===
Key    Value
---    -----
bar    copper
```


Now Vault returns `copper` for the value.

Delete the latest version (version 2)
```
bash-5.1# vault kv delete kv/foo

Success! Data deleted (if it existed) at: kv/foo
```

Try to read version 2

```
bash-5.1# vault kv get -version=2 kv/foo
======= Metadata =======
Key                Value
---                -----
created_time       2022-01-15T02:13:44.314700364Z
custom_metadata    <nil>
deletion_time      2022-01-15T02:14:37.929808384Z
destroyed          false
version            2
```

You can see the value is not available.

Now delete version 1
```
bash-5.1# vault kv delete -versions=1 kv/foo

Success! Data deleted (if it existed) at: kv/foo
```

Vault also supports 'undeleting' a secret

Undelete version 1
```
bash-5.1# vault kv undelete -versions=1 kv/foo

Success! Data written to: kv/undelete/foo
```

Delete is a soft delete. If you want to remove the underlying metadata, you'll have to use the `destroy` command.

Destroy version 1
```
bash-5.1# vault kv destroy -versions=1 kv/foo

Success! Data written to: kv/destroy/foo
```

> All of the above requests were logged to the audit log.

### API

You can also interact with Vault via the HTTP API. We'll make requests against v2 of the API. Open a new terminal tab, and then set the root token as an environment variable:

```bash
export VAULT_TOKEN=your_token_goes_here
```

Create a new secret called `foo` with a value of `world`:

```bash
curl \
    -H "X-Vault-Token: $VAULT_TOKEN" \
    -H "Content-Type: application/json" \
    -X POST \
    -d '{ "data": { "foo": "world" } }' \
    http://127.0.0.1:8200/v1/kv/data/hello
```

Read the secret:

```bash
curl \
    -H "X-Vault-Token: $VAULT_TOKEN" \
    -X GET \
    http://127.0.0.1:8200/v1/kv/data/hello
```

The JSON response should contain a `data` key with a value similar to:

```json
"data": {
  "data":{
    "foo": "world"
  },
  "metadata": {
    "created_time": "2021-09-08T18:30:32.5140484Z",
    "deletion_time": "",
    "destroyed": false,
    "version": 1
  }
}
```

Play around with Vault secrets: Create new versions, Delete, and Destroy on your own.

# Congrats
