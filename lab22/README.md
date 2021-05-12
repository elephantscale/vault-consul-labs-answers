# Dynamic secrets

* Once a static secret leaks out, it's not a secret any longer
* Vault can generate secrets on-demand for some systems. For example, when an app needs to access an Amazon S3 bucket, it asks Vault for AWS credentials. Vault will generate an AWS credential granting permissions to access the S3 bucket. In addition, Vault will automatically revoke this credential after the time-to-live (TTL) expires.

## Lab Goals:

* Practice dynamic secret by using TTL

### Builds on:
* [Install lab](../lab01)

### Time:
    * 30 min

### Step 0) Lab plan

* Challenge
    * Data protection is a top priority, and database credential rotation is a critical part of any data protection initiative. Each role has a different set of permissions granted to access the database. When a system is attacked by hackers, continuous credential rotation becomes necessary and needs to be automated.

* Solution
    * Applications ask Vault for database credentials rather than setting them as environment variables. The administrator specifies the TTL of the database credentials to enforce its validity so that they are automatically revoked when they are no longer used.
  
![](../artwork/fig22-1.png)

* Each app instance can get unique credentials that they don't have to share. By making those credentials short-lived, you reduce the chance that they might be compromised. If an app was compromised, the credentials used by the app can be revoked rather than changing more global sets of credentials.

* Personas
  * The end-to-end scenario described in this lab involves two personas:
    * admin with privileged permissions to configure secrets engines
    * apps read the secrets from Vault
  
### Step 1) Start Postgres

* The lab requires a Postgres database. Docker provides a Postgres server image that satisfies this requirement.

  * NOTE: This lab works for an existing Postgres database given appropriate credentials and connection information.

  * Pull a Postgres server image with `docker`.
  
```shell
docker pull postgres:latest
```

* Create a Postgres database with a root user named root with the password rootpassword.

```shell
docker run \
      --name postgres \
      --env POSTGRES_USER=root \
      --env POSTGRES_PASSWORD=rootpassword \
      --detach  \
      --publish 5432:5432 \
      postgres
```

* The database is available.

* The credentials generated by the Vault role in the Create a role step requires a role named ro that has been granted the ability to read all tables.

* Connect to the Postgres database via the CLI within the postgres container.

```shell
docker exec -it postgres psql
```

* Your system prompt is replaced with a new prompt root=#. Commands issued at this prompt are executed against the Postgres database running within the container.

* Create a role named ro.

```shell
CREATE ROLE ro NOINHERIT;
```

* Grant the ability to read all tables to the role named ro

```shell
Grant the ability to read all tables to the role named ro.
```

* The role is created and assigned the appropriate permissions.

* Disconnect from the Postgres database.

```shell
\q
```

### Step 2) Start Vault
* In another terminal, start a Vault dev server with root as the root token.

```shell
vault server -dev -dev-root-token-id root
```

* The Vault dev server defaults to running at 127.0.0.1:8200. The server is initialized and unsealed.

* **Insecure operation**: Do not run a Vault dev server in production. This approach starts a Vault server with an in-memory database and runs in an insecure way.

* Export an environment variable for the vault CLI to address the Vault server.

```shell
export VAULT_ADDR=http://127.0.0.1:8200
```

* Export an environment variable for the vault CLI to authenticate with the Vault server.

```shell
export VAULT_TOKEN=root
```

* **NOTE**: For these tasks, you can use Vault's root token. However, it is recommended that root tokens are only used for enough initial setup or in emergencies. As a best practice, use an authentication method or token that meets the policy requirements.

* The Vault server is ready.

### Step 3) Scenario Introduction

* In this lab, you are going to configure the PostgreSQL secrets engine, and create a read-only database role. The Vault-generated PostgreSQL credentials will only have read permission.
  * Enable the database secrets engine
  * Configure PostgreSQL secrets engine
  * Create a role
  * Request PostgreSQL credentials
  * Manage leases
  * Define a password policy
  * Define a username template
  
### Step 4) Enable the database secrets engine

* Enable the database secrets engine at the database/ path.

```shell
vault secrets enable database
```

### Step 5) Configure PostgreSQL secrets engine

* (Persona: admin)

* The database secrets engine supports many databases through a plugin interface. To use a Postgres database with the secrets engine requires further configuration with the postgresql-database-plugin plugin and connection information.

* Configure the database secrets engine with the connection credentials for the Postgres database.

```shell
vault write database/config/postgresql \
     plugin_name=postgresql-database-plugin \
     connection_url="postgresql://{{username}}:{{password}}@localhost:5432/postgres?sslmode=disable" \
     allowed_roles=readonly \
     username="root" \
     password="rootpassword"
```

### Step 4) Create a role

* (Persona: admin)

* In configure Postgresql secrets engine step, you configured the PostgreSQL secrets engine with the allowed role named readonly. A role is a logical name within Vault that maps to database credentials. These credentials are expressed as SQL statements and assigned to the Vault role.

* Define the SQL used to create credentials.

```shell
tee readonly.sql <<EOF
CREATE ROLE "{{name}}" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' INHERIT;
GRANT ro TO "{{name}}";
EOF
```

* The SQL contains the templatized fields {{name}}, {{password}}, and {{expiration}}. These values are provided by Vault when the credentials are created. This creates a new role and then grants that role the permissions defined in the Postgres role named ro. This Postgres role was created when Postgres was started.

* Create the role named readonly that creates credentials with the readonly.sql.

```shell
vault write database/roles/readonly \
      db_name=postgresql \
      creation_statements=@readonly.sql \
      default_ttl=1h \
      max_ttl=24h
```

* The role generates database credentials with a default TTL of 1 hour and max TTL of 24 hours.

### Step 5) Request PostgreSQL credentials

* (Persona: apps)

* The applications that require the database credentials read them from the secret engine's readonly role.

* Read credentials from the readonly database role.

```shell
vault read database/creds/readonly
```

* You will get something like this

![](../artwork/fig22-2.png)

* The Postgres credentials are displayed as username and password. The credentials are identified within Vault by the lease_id.

### Step 5) Validation

* Connect to the Postgres database via the CLI within the postgres container.

* Your system prompt is replaced with a new prompt root=#. Commands issued at this prompt are executed against the Postgres database running within the container.

* List all the database users.

```shell
SELECT usename, valuntil FROM pg_user;
```

* You will see this kind of output

![](../artwork/fig22-3.png)

* The output displays a table of all the database credentials generated. The credentials that were recently generated appear in this list.

* Disconnect from the Postgres database.

```shell
\q
```

### Step 6) Manage leases

* (Persona: admin)

* The credentials are managed by the lease ID and remain valid for the lease duration (TTL) or until revoked. Once revoked the credentials are no longer valid.

* List the existing leases.

```shell
 vault list sys/leases/lookup/database/creds/readonly
```

* All valid leases for database credentials are displayed.

* Create a variable that stores the first lease ID.

```shell
LEASE_ID=$(vault list -format=json sys/leases/lookup/database/creds/readonly | jq -r ".[0]")
```

* Renew the lease for the database credential by passing its lease ID.

```shell
vault lease renew database/creds/readonly/$LEASE_ID
```

![](../artwork/fig22-4.png)

* Note that the TTL of the renewed lease is set to 1h.

* Revoke the lease without waiting for its expiration.

```shell
vault lease revoke database/creds/readonly/$LEASE_ID
```

* Observe success status

* List the existing leases.

```shell
vault list sys/leases/lookup/database/creds/readonly
```

* The lease is no longer valid and is not displayed.

* Read new credentials from the readonly database role.

```shell
vault read database/creds/readonly
```

![](../artwork/fig22-5.png)

* All leases associated with a path may be removed.

* Revoke all the leases with the prefix database/creds/readonly.

```shell
vault lease revoke -prefix database/creds/readonly
```

* The prefix flag matches all valid leases with the path prefix of database/creds/readonly.

* List the existing leases.

```shell
vault list sys/leases/lookup/database/creds/readonly
```

* All the leases with this path as a prefix have been revoked.

### Step 7) Define a password policy

* The database secret engines generate passwords that adhere to a default pattern that may be overridden with a new password policy. A policy defines the rules and requirements that the password must adhere to and can provide that password directly through a new endpoint or within secrets engines.

* The passwords you want to generate adhere to these requirements.

  * length of 20 characters
  * at least 1 uppercase character
  * at least 1 lowercase character
  * at least 1 number
  * at least 1 symbol
  
* Define this password policy in a file named example_policy.hcl.

```shell
tee example_policy.hcl <<EOF
length=20

rule "charset" {
  charset = "abcdefghijklmnopqrstuvwxyz"
  min-chars = 1
}

rule "charset" {
  charset = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
  min-chars = 1
}

rule "charset" {
  charset = "0123456789"
  min-chars = 1
}

rule "charset" {
  charset = "!@#$%^&*"
  min-chars = 1
}
EOF
```

* The policy is written in HashiCorp Configuration Language (HCL). The length field sets the length of the password returned to 20 characters. Each rule stanza defines a character set and the minimum number of occurrences those characters need to appear in the generated password. These rules are cumulative so each one adds more requirements on the password generated.

* Create a Vault password policy named example with the password policy rules defined in example_policy.hcl.

```shell
vault write sys/policies/password/example policy=@example_policy.hcl
```

* This policy can now be accessed directly to generate a password or referenced by its name example when configuring supported secrets engines.

* Generate a password from the example password policy.

```shell
vault read sys/policies/password/example/generate
```

```text
Key         Value
---         -----
password    zUKMWPOr821D%DZHklt%

```

* The password generated adheres to the defined requirements.

### Step 8) Apply the password policy

* Configure the database secrets engine with the example password policy.

```shell
vault write database/config/postgresql \
     password_policy="example"
```

* The same connection information is used to establish the connection with the database server. The difference is that the password_policy has been set to the example policy.

* Read credentials from the readonly database role.

```shell
vault read database/creds/readonly
```

* You will get similar output

![](../artwork/fig22-6.png)

* The credentials display the username and password generated. The password generated adheres to the example password policy defined in the secrets engine's configuration.

### Step 9) Define a username template

* The database secret engines generate usernames that adhere to a default pattern. A customized username template may be provided to meet the needs of your organization.
  * Ensure that custom username templates include enough randomness to prevent the same username being generated multiple times.
  
* Read credentials from the readonly database role.

```shell
vault read database/creds/readonly
```

![](../artwork/fig22-7.png)

* The generated username, v-token-readonly-SqapepE0EGKAQyQcXxDN-1620277969, uses the default pattern expressed as a Go template, {{ printf "v-%s-%s-%s-%s" (.DisplayName | truncate 8) (.RoleName | truncate 8) (random 20) (unix_time) | truncate 63 }}.

* Configure the database secrets engine with the username template.

```shell
vault write database/config/postgresql \
    username_template="myorg-{{.RoleName}}-{{unix_time}}-{{random 8}}"
```

* This username template is prefixed with myorg-, uses the name of role, readonly, the unix timestamp in seconds, and a random sequence of 8 characters.

* Read credentials from the readonly database role.

```shell
vault read database/creds/readonly
```
![](../artwork/fig22-8.png)

* The username generated adheres to the template provided to the configuration.

### Congratulations!









