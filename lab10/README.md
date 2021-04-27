# Using the HTTP APIs with Authentication

* Access Vault with HTTP APIs
* All of Vault's capabilities are accessible via the HTTP API in addition to the CLI. 
* In fact, most calls from the CLI actually invoke the HTTP API. In some cases, Vault features are not available via the CLI and can only be accessed via the HTTP API.
  
## Lab Goals:

*  
*  

### Builds on:
* [Install lab](../lab01)

### Time:
    * 30 min

### Step 1) Prepare
  
* Press Ctrl+C to terminate the dev server if you are running it at at http://127.0.0.1:8200 before proceeding.

* Machines that need access to information stored in Vault will most likely access Vault via its REST API. For example, if a machine were using AppRole for authentication, the application would first authenticate to Vault which would return a Vault API token. The application would use that token for future communication with Vault.

### Step 2) Start the Vault server

* Create a server configuration file, config.hcl.

* For the purpose of this lab, you will use the following configuration which disables TLS and uses a file-based backend. TLS is disabled here only for example purposes; it should **never** be disabled in production.

```shell
tee config.hcl <<EOF
storage "file" {
  path = "vault-data"
}

listener "tcp" {
  tls_disable = "true"
}
EOF
```

* You might have already had the `config.hcl`, and this command will overwrite it. Alternatively, create the file using an editor.

* Start a new Vault instance using the newly created configuration.

```shell
vault server -config=config.hcl
```

* At this point, you can use Vault's HTTP API for all your interactions.

* Launch a new terminal session, and use curl to initialize Vault with the API.

* **NOTE**: This example uses jq to process the JSON output for readability.

```shell
curl \
    --request POST \
    --data '{"secret_shares": 1, "secret_threshold": 1}' \
    http://127.0.0.1:8200/v1/sys/init | jq
```

* The response should be JSON and looks something like this:

![](../artwork/fig10-1.png)

* This response contains your initial root token. It also includes the unseal key. You can use the unseal key to unseal the Vault and use the root token perform other requests in Vault that require authentication.

* To make this lab easy to copy-and-paste, you will be using the environment variable $VAULT_TOKEN to store the root token.

Example:

```shell
export VAULT_TOKEN="s.iJbNzzIR0T5xUkZ1p2vQMVq9"
```

* Using the unseal key (not the root token) from above, you can unseal the Vault via the HTTP API.

```shell
curl \
    --request POST \
    --data '{"key": "MAWmakiRveqpNkjv7VHs/4UXfbyBB7tMraQyQXt5Sw0="}' \
    http://127.0.0.1:8200/v1/sys/unseal | jq
```

* Note that you should replace the `MAWmakiRveqpNkjv7VHs/4UXfbyBB7tMraQyQXt5Sw0=` with the generated key from your output. This will return a JSON response:

* You should get success

![](../artwork/fig10-2.png)


