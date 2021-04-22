# Dynamic Secrets

* 
* 

## Lab Goals:

* Learn to work with dynamic secrets

### Builds on:
* Previous labs

### Time:
    * 15 min

* Now that you've experimented with the kv secrets engine, it is time to explore another feature of Vault: dynamic secrets.

* Unlike the kv secrets where you had to put data into the store yourself, dynamic secrets are generated when they are accessed. Dynamic secrets do not exist until they are read, so there is no risk of someone stealing them or another client using the same secrets. Because Vault has built-in revocation mechanisms, dynamic secrets can be revoked immediately after use, minimizing the amount of time the secret existed.

Note: Before starting this page, please register for an AWS account. You won't be using any features that cost money, so you shouldn't be charged for anything. However, we are not responsible for any charges you may incur.

### Step 1) Register for an AWS account (or use one if you have it)

![](../artwork/aws-account.png)

---

### Step 2) Enable the AWS secrets engine

* Unlike the `kv` secrets engine which is enabled by default, the AWS secrets engine must be enabled before use. This step is usually done via a configuration management system.

```shell
vault secrets enable -path=aws aws
```

* Vault should acknowledge success, like this:

![](../artwork/enable-aws.png)

* The AWS secrets engine is now enabled at aws/. Different secrets engines allow for different behavior. In this case, the AWS secrets engine generates dynamic, on-demand AWS access credentials.

### Step 3) Configure the AWS secrets engine

* After enabling the AWS secrets engine, you must configure it to authenticate and communicate with AWS. 
This requires privileged [AWS account credentials](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html).
  
* Set an AWS_ACCESS_KEY_ID environment variable to hold your AWS access key ID.