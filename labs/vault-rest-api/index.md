# Configure and Test Vault REST API

# Overview
In this lab you will create a token along with a policy allowing creation and management of secrets at specific paths. This is achieved by making various requests to the API endpoints.

This lab is more advanced and does not provide step-by-step instructions. Use the concepts taught previously to accomplish the objectives. 

* Log in with the root token and unseal Vault
* Enable `kv` Secrets Engine at: 
  * `secrets-kv-A`
  * `secrets-kv-B`
* Create a Policy for `secrets-kb-A`, and `secrets-kv-B` with `CREATE`, `READ`, and `UPDATE` capabilites
  * Create a policy file
  * Populate a policy file
  * Write a policy 
* Create a token with the previously created policy and test.
  * Create a token and assign the newly created policy 
  * Create secrets
  * Get secrets 
  * Delete a secret
  * Try to get a deleted secret

* BONUS:
  * Create an additional policy with only `READ` access to `secrets-kv-B`
    * Create a token and assign the read-only policy 
    * Confirm you can only read secrets from `secrets-kb-B`

## Congrats
