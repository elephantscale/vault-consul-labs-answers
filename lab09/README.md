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

* Vault is configured using [HCL](https://github.com/hashicorp/hcl) files. 
  The configuration file for Vault is relatively simple:
  
```HCL
storage "raft" {
  path    = "./vault/data"
  node_id = "node1"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = "true"
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui = true
```

* **NOTE** Although the listener stanza disables TLS (tls_disable = "true") for this tutorial, Vault should always be used with TLS in production to provide secure communication between clients and the Vault server. It requires a certificate file and key file on each Vault host.

* Within the configuration file, there are two primary configurations:

  * `storage` - This is the physical backend that Vault uses for storage. Up to this point the dev server has used "inmem" (in memory), but the example above uses integrated storage (raft), a much more production-ready backend.

  * `listener` - One or more listeners determine how Vault listens for API requests. The example above listens on localhost port 8200 without TLS. In your environment set VAULT_ADDR=http://127.0.0.1:8200 so the Vault client will connect without TLS.

  * `api_addr` - Specifies the address to advertise to route client requests.

  * `cluster_addr` - Indicates the address and port to be used for communication between the Vault nodes in a cluster.

* For now, copy and paste the configuration above to a file called config.hcl.

### Step 2) Starting the Server

* The ./vault/data directory that raft storage backend uses must exist

```shell
 mkdir -p ./vault/data
```
  
* Set the -config flag to point to the proper path where you saved the configuration above.

```shell
vault server -config=config.hcl
```

* The server should start:

![](../artwork/fig9-1.png)

* **NOTE**: If you get a warning message about mlock not being supported, that is okay. However, for maximum security you should run Vault on a system that supports mlock.

* Vault outputs some information about its configuration, and then blocks. This process should be run using a resource manager such as systemd or upstart.

* For potential problem if running on Linux, see [here](../note1.md)

### Step 3) Initializing the Vault

* Initialization is the process of configuring Vault. This only happens once when the server is started against a new backend that has never been used with Vault before. When running in HA mode, this happens once per cluster, not per server. During initialization, the encryption keys are generated, unseal keys are created, and the initial root token is setup.

* Launch a new terminal session, and set VAULT_ADDR environment variable.

```shell
export VAULT_ADDR='http://127.0.0.1:8200'
```

* To initialize Vault use vault operator init. This is an unauthenticated request, but it only works on brand new Vaults without existing data:

```shell
vault operator init
```

* You will get this kind of output

![](../artwork/fig9-2.png)

* Initialization outputs two incredibly important pieces of information: the unseal keys and the initial root token. This is the only time ever that all of this data is known by Vault, and also the only time that the unseal keys should ever be so close together.

* For the purpose of this lab, save all of these keys somewhere, and continue. In a real deployment scenario, you would never save these keys together. Instead, you would likely use Vault's PGP and Keybase.io support to encrypt each of these keys with the users' PGP keys. This prevents one single person from having all the unseal keys. 
  Please see the documentation on using [PGP, GPG, and Keybase](https://www.vaultproject.io/docs/concepts/pgp-gpg-keybase) for more information.
  
* You will see the output like below - please read the notes

### Step 3) Seal/Unseal

* Every initialized Vault server starts in the sealed state. From the configuration, Vault can access the physical storage, but it can't read any of it because it doesn't know how to decrypt it. The process of teaching Vault how to decrypt the data is known as unsealing the Vault.

* Unsealing has to happen every time Vault starts. It can be done via the API and via the command line. To unseal the Vault, you must have the threshold number of unseal keys. In the output above, notice that the "key threshold" is 3. This means that to unseal the Vault, you need 3 of the 5 keys that were generated.

* **Note**: Vault does not store any of the unseal key shards. Vault uses an algorithm known as Shamir's Secret Sharing to split the master key into shards. Only with the threshold number of keys can it be reconstructed and your data finally accessed.

* Begin unsealing the Vault:

```shell
vault operator unseal
```



