# Secrets Engines

* There are many secrets engines in Vault
* You must be conscious about which engine you are talking to

## Lab Goals:

* Learn to work with secrets engines

### Builds on:
* ["Write a secret" lab](../lab03)

### Time:
    * 15 min

### Step 1) Note the use of the `secrets` engine

* Please note that in the previous lab, all requests started with `secret/`
* Try to do without it, by running the following command (which should give you an **error**)

```shell
vault kv put foo/bar a=b

```

* You should get this

![](../artwork/vault-error.png)

### Why is that?

* The path prefix tells Vault which secrets engine to which it should route traffic. When a request comes to Vault, it matches the initial path part using a longest prefix match and then passes the request to the corresponding secrets engine enabled at that path. Vault presents these secrets engines similar to a filesystem.

* By default, Vault enables Key/Value version2 secrets engine (kv-v2) at the path secret/ when running in dev mode. The key/value secrets engine reads and writes raw data to the backend storage. Vault supports many other secrets engines, and this feature makes Vault flexible and unique.

* **NOTE**
* The key/value secrets engine has two versions: kv (version 1) and kv-v2 (version 2). The kv-v2 is versioned kv secrets engine which can retain a number of secrets versions.
* This page discusses secrets engines and the operations they support. This information is important to both operators who will configure Vault and users who will interact with Vault.

### Step 2) Enable a Secrets Engine

* To get started, enable the `kv` secrets engine. 
Each path is completely isolated and cannot talk to other paths. For example, a `kv` secrets engine enabled at foo has no ability to communicate with a `kv` secrets engine enabled at bar.
  
* Now, enable a secrets engine by running the following command

```shell
vault secrets enable -path=kv kv
```

* You should get `success`

![](../artwork/enable-secrets-engine.png)



