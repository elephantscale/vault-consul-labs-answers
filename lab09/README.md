# Deploy Vault

* Deploy Vault in the ways you would do it in production

## Lab Goals:

* Configure and deploy Vault 
* Practice real-life deployment 

### Builds on:
* [Install lab](../lab01)

### Time:
    * 30 min

### Step 1) Undo the `dev` deployment of Vault

* Up to this point, you interacted with the "dev" server, which automatically unseals Vault, sets up in-memory storage, etc. Now that you know the basics of Vault, it is important to learn how to deploy Vault into a real environment.

* Press **Ctrl+C** to terminate the dev server that is running at `http://127.0.0.1:8200` before start.

* Also, unset the VAULT_TOKEN environment variable.

```shell
unset VAULT_TOKEN
```

* In this lab, you will learn how to configure Vault, start Vault, use the seal/unseal process, and scale Vault.

### Step 2) Configuring Vault

