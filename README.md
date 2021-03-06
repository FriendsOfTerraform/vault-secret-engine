# Vault Secret Engine Module

This module will create and configure a [Vault secret enging][secret-engine].

## Table of Contents

- [Example Usage](#example-usage)
    - [AWS](#aws)
    - [Database](#database)
    - [KV](#kv)
    - [PKI](#pki)
    - [Terraform Cloud](#terraform-cloud)
- [Argument Reference](#argument-reference)
- [Outputs](#outputs)

## Example Usage

### AWS

This example creates an AWS secret mount at the `aws` path. Multiple roles are then created to demostrate two different scenarios this secret engine can be use in; Dynamically creates `iam_user` and dynamically returns temporary credential via `assume_role`.

```terraform
module "aws" {
  source = {{PLACE_HOLDER}}

  secret_engine = "aws"
  mount_path    = "aws"

  aws_config = {
    access_key_id       = "AKIAXXXXXXXXXXXX"
    secret_access_key   = "5LoxXag/XXXXXXXXXXXXXXXX"
    region              = "us-east-1"
    max_ttl_seconds     = 1800
    default_ttl_seconds = 300
  }

  aws_secret_backend_roles = {
    # scenario: a GitOps operator such as terraform request token with access to S3 and EC2 services
    aws-gitops-operator = {
      aws_managed_policy_arns = ["arn:aws:iam::aws:policy/AmazonS3FullAccess"]
      inline_policy_document  = file("${path.root}/ec2-full-access.json")
    }
    # scenario: an application request dynamic access to resources localed in an AWS account owned by someone else
    partner-account = {
      role_arns = ["arn:aws:iam::111122223333:role/partner-account-application"]
    }
  }
}
```

#### Notes

1. The specification of the AWS credential is optional. You may also use alternative methods such as environment variables or local aws credential file to pass the necessary AWS credential to this secret engine
2. This secret engine [requires these IAM policy][aws-iam-policy] at minimum to dynamically manage IAM entities (`iam_user` and `assume_role`)
3. The assumed_role credential type is typically used for cross-account authentication scenario. [Additional IAM permissions are needed][aws-iam-policy-cross-account] to allow IAM to assume role to another AWS account

### Database

This example creates a Database secret mount at the `database` path. A `test-role` static role will be created to map the `test-role` user from a PostgreSQL database to a Vault role.

```terraform
module "database" {
  source = {{PLACE_HOLDER}}

  secret_engine = "database"
  mount_path    = "database"

  database_config = {
    postgres = {
      demo-database = {
        allowed_roles  = ["test-role"]
        connection_url = "postgresql://postgres:password@postgresql.friendsofterraform.com:5432/postgres?sslmode=disable"
      }
    }
  }

  database_static_backend_roles = {
    "test-role" = {
      database_name = "demo-database"
    }
  }
```

### KV

This example creates a Key/Value secret mount at the `kv` path.

```terraform
module "kv" {
  source = {{PLACE_HOLDER}}

  secret_engine = "kv"
  mount_path    = "kv"
}
```

### PKI

This example creates a Root CA certificate and mounts it in the `pki/root-ca` path, then creates an intermediate CA certificate with the generated Root CA and mounts it in the `pki/intermediate-ca` path.

This example also create two roles to allow the intermediate CA to sign certificates for both the `friendsofterraform.sh` and the `friendsofterraform.com` domains.

```terraform
module "root_ca" {
  source = {{PLACE_HOLDER}}

  secret_engine       = "pki"
  mount_path          = "pki/root-ca"
  default_ttl_seconds = 31536000  # 1 year
  max_ttl_seconds     = 315360000 # 10 years  

  pki_config = {
    cert_type = "root"
  }

  pki_root_cert = {
    common_name       = "root-ca"
    ttl_seconds       = 315360000 # 10 years
    vault_address     = "https://vault.friendsofterraform.sh"
  }
}

module "intermediate_ca" {
  source = {{PLACE_HOLDER}}

  secret_engine       = "pki"
  mount_path          = "pki/intermediate-ca"
  default_ttl_seconds = 31536000  # 1 year
  max_ttl_seconds     = 157680000 # 5 years

  pki_config = {
    cert_type           = "intermediate"
  }

  pki_intermediate_ca = {
    signing_ca_mount_path = module.root_ca.mount_path # signs the intermediate CA with this CA
    common_name           = "intermediate-ca"
    ttl_seconds           = 157680000 # 5 years
  }

  pki_secret_backend_roles = {
    friendsofterraform-sh = {
      ttl_seconds      = 259200 # 72 hours
      max_ttl_seconds  = 259200 # 72 hours
      allowed_domains  = ["friendsofterraform.sh"]
    }
    friendsofterraform-com = {
      ttl_seconds      = 259200 # 72 hours
      max_ttl_seconds  = 259200 # 72 hours
      allowed_domains  = ["friendsofterraform.com"]
    }
  }
}
```

### Terraform Cloud

This example creates the Terraform Cloud secret mount at the `terraform` path and configures it to dynamically manages API tokens from a user identity.

```terraform
module "terraform_cloud" {
  source = {{PLACE_HOLDER}}

  secret_engine = "terraform"
  mount_path    = "terraform"

  terraform_config = {
    token = "Q3JtWYZK5Zx81w......"
  }

  terraform_secret_backend_roles = {
    user-role = {
      token_identity  = "user-xxxxxxxxxxxx"
    }
  }
}
```

## Argument Reference

- (string) **`mount_path`** _[since v0.0.1]_

    The mount path of the secret engine

- (string) **`secret_engine`** _[since v0.0.1]_

    The Vault secret engine to configure, currently the following values are supported:
    - [aws](https://www.vaultproject.io/docs/secrets/aws)
    - [database](https://www.vaultproject.io/docs/secrets/databases)
    - [pki](https://www.vaultproject.io/docs/secrets/pki)
    - [kv](https://www.vaultproject.io/docs/secrets/kv/kv-v2)
    - [terraform](https://www.vaultproject.io/docs/secrets/terraform)

- (object) **`aws_config = null`** _[since v0.0.8]_

    Configuration of an AWS secret engine. This is **OPTIONAL** even if `secret_engine = aws` since Vault can also read AWS credential from other methods such as enviornment variables and local AWS credential file.
    
    ```terraform
    aws_config = {
      access_key_id       = "AKIAXXXXXXXXXXXX"
      secret_access_key   = "5LoxXag/XXXXXXXXXXXXXXXX"
      region              = "us-east-1"
      max_ttl_seconds     = 1800
      default_ttl_seconds = 300
    }
    ```
    
    - (string) **`access_key_id`** _[since v0.0.8]_

        AWS access key ID

    - (string) **`region`** _[since v0.0.8]_

        AWS region

    - (string) **`secret_access_key`** _[since v0.0.8]_

        AWS secret access key
    
    - (number) **`default_ttl_seconds = null`** _[since v0.0.8]_
    
        Default TTL (time-to-live) for new IAM credential created by this this secret engine

    - (number) **`max_ttl_seconds = null`** _[since v0.0.8]_
    
        Max TTL (time-to-live) for new IAM credential created by this this secret engine

- (map(object)) **`aws_secret_backend_roles = {}`** _[since v0.0.8]_

    [Configure multiple roles][aws-role] that maps a name in Vault to an IAM entity (IAM User or IAM Role) to create new credentials. When users or machines create new credentials, they are created against this role. Input must be in `role_name = role_config` format.
    
    ```terraform
    aws_secret_backend_roles = {
      s3-operator = {
        aws_managed_policy_arns = ["arn:aws:iam::aws:policy/AmazonS3FullAccess"]
      }
    }
    ```
    
    - (list(string)) **`aws_managed_policy_arns = null`** _[since v0.0.8]_

        Specifies a list of AWS managed policy ARNs that will be attached to the IAM user generated
    
    - (list(string)) **`iam_group_names = null`** _[since v0.0.8]_
    
        A list of IAM group names. IAM users generated against this vault role will be added to these IAM Groups. This option is mutually exclusive with `aws_managed_policy_arns`

    - (string) **`inline_policy_document = null`** _[since v0.0.8]_
    
        An AWS IAM policy document that will be attached to the IAM user generated as an inline policy

    - (list(string)) **`role_arns = null`** _[since v0.0.8]_
    
        Specifies the ARNs of the AWS roles this Vault role is allowed to assume. This option is mutually exclusive with `iam_group_names`

- (map(object)) **`database_config = null`** _[since v0.0.10]_

    Configuration of a Database secret engine. This is required if `secret_engine = database`.
    
    ```terraform
    database_config = {
      postgres = {
        test = {
          allowed_roles  = ["test-role"]
          connection_url = "postgresql://postgres:password@postgresql.friendsofterraform.com:5432/postgres?sslmode=disable"
        }
      }
    }
    ```
    
    - (map(object)) **`postgres = null`** _[since v0.0.10]_

        Establish a connection to a Postgres database
    
      - (string) **`connection_url`** _[since v0.0.10]_
      
          The connection string for the database, in this formation `postgresql://{{username}}:{{password}}@localhost:5432/postgres?sslmode=disable`

      - (number) **`max_connection_lifetime_seconds = null`** _[since v0.0.10]_
      
          The maximum number of seconds to keep a connection alive for

      - (number) **`max_idle_connections = null`** _[since v0.0.10]_
      
          The maximum number of idle connections to maintain

      - (number) **`max_open_connections = null`** _[since v0.0.10]_
      
          The maximum number of open connections to use

- (map(object)) **`database_static_backend_roles = null`** _[since v0.0.10]_

    Configures the mapping of a Vault role to a database user, in `username = {configuration}`
    
    ```terraform
    database_static_backend_roles = {
      "test-role" = {database_name = "test"}
    }
    ```
    
    - (string) **`database_name`** _[since v0.0.10]_

        The name of the database to manage
    
    - (number) **`rotation_period_seconds = 86400`** _[since v0.0.10]_

        The rotation period for the password of the managed user

- (number) **`default_ttl_seconds = null`** _[since v0.0.6]_

    Global default TTL (time-to-live) in seconds for all secrets within this secret mount

- (string) **`description = "Managed by Terraform"`** _[since v0.0.1]_

    Description of the secret engine

- (number) **`max_ttl_seconds = null`** _[since v0.0.6]_

    Global max TTL (time-to-live) in seconds for all secrets within this secret mount

- (object) **`pki_config = null`** _[since v0.0.1]_

    Configuration of a PKI secret engine. This is required if `secret_engine = pki`. This secret engine only allows you to configure a root CA or an intermediate CA.
    
    ```terraform
    pki_config = {
      cert_type = "root"
    }
    ```
    
    - (string) **`cert_type`** _[since v0.0.1]_

        The type of certificate to configure, valid values are `root` or `intermediate`

- (object) **`pki_intermediate_ca = null`** _[since v0.0.1]_

    Options for configuring an intermediate CA, this is required if `pki_config.cert_type = intermediate`
    
    ```terraform
    pki_intermediate_ca = {
      signing_ca_mount_path = module.root_ca.mount_path # signs the intermediate CA with this CA
      common_name           = "intermediate-ca"
      ttl_seconds           = 157680000 # 5 years
    }
    ```
    
    - (string) **`common_name`** _[since v0.0.1]_
    
        Specifies the common name for this intermediate CA certificate
    
    - (string) **`signing_ca_mount_path`** _[since v0.0.1]_
    
        Specifies the the secret engine mount path of a CA certificate that will be used to sign this intermediate CA certificate

    - (list(string)) **`alternative_names = null`** _[since v0.0.1]_

        Specifies a list of SAN (server alternative names) for this intermediate CA certificate

    - (number) **`ttl_seconds = null`** _[since v0.0.1]_
    
        Specifies the TTL (time-to-live) for this intermediate CA certificate

- (object) **`pki_root_cert = null`** _[since v0.0.1]_

    Options for configuring a Root CA, this is required if `pki_config.cert_type = root`
    
    ```terraform
    pki_root_cert = {
      common_name       = "root-ca"
      ttl_seconds       = 315360000 # 10 years
      vault_address     = "https://vault.friendsofterraform.sh"
    }
    ```
    
    - (string) **`common_name`** _[since v0.0.1]_
    
        Specifies the common name for this Root CA certificate

    - (string) **`vault_address`** _[since v0.0.3]_
    
        Specifies the address of the Hashicorp Vault server. issuing certificate endpoints, CRL distribution points, and OCSP server endpoints that will be encoded into issued certificates. Please refer to [this doc][pki-config-urls] for more information. The generated endpoints will be in this format `<vault_address>/v1/<pki_mount_path>/ca`

    - (list(string)) **`alternative_names = null`** _[since v0.0.1]_

        Specifies a list of SAN (server alternative names) for this Root CA certificate  
  
    - (number) **`ttl_seconds = null`** _[since v0.0.1]_
    
        Specifies the TTL (time-to-live) for this Root CA certificate

- (map(object)) **`pki_secret_backend_roles = {}`** _[since v0.0.2]_

    [Configure multiple roles][pki-secret-setup] that maps a name in Vault to a procedure for generating a certificate. When users or machines generate credentials, they are generated against this role. Input must be in `role_name = role_config` format.
    
    ```terraform
    pki_secret_backend_roles = {
      friendsofterraform-sh = {
        ttl_seconds      = 259200 # 72 hours
        max_ttl_seconds  = 259200 # 72 hours
        allowed_domains  = ["friendsofterraform.sh"]
      }
    }
    ```
    
    - (list(string)) **`allowed_domains = null`** _[since v0.0.2]_

        A list of domain names this role is allowed to sign the certificate for
    
    - (list(string)) **`allowed_uri_sans = null`** _[since v0.0.2]_
    
        A list of URI SANs (Subject alternative names) this role is allowed to sign the certificate for
    
    - (number) **`max_ttl_seconds = null`** _[since v0.0.2]_
    
        Specifies the max TTL (time-to-live) for certificates generated from this role

    - (number) **`ttl_seconds = null`** _[since v0.0.2]_
    
        Specifies the TTL (time-to-live) for certificates generated from this role

- (object) **`terraform_config = null`** _[since v0.0.7]_

    Configures the Terraform Cloud secret engine. This is required if `secret_engine = terraform`
    
    ```terraform
    terraform_config = {
      token = "Q3JtWYZK5Zx81w........"
    }
    ```
    
    - (string) **`token`** _[since v0.0.7]_

        The Terraform Cloud management token this backend should use to issue new tokens

- (map(object)) **`terraform_secret_backend_roles = {}`** _[since v0.0.7]_

    [Configure multiple roles][terraform-role] that maps a name in Vault to a Terraform Cloud token type (user, team, or organization) to create new tokens. When users or machines create new token, they are created against this role. Input must be in `role_name = role_config` format.
    
    ```terraform
    terraform_secret_backend_roles = {
      user-role = {
        token_identity  = "user-xxxxxxxxxxxx"
        ttl_seconds     = null
        max_ttl_seconds = null
      }
    }
    ```
    
    - (string) **`token_identity`** _[since v0.0.7]_

        Specifies the Terraform Cloud entity to be used to generate new tokens with. Must follow the following format:

        - Organization - Organization name. For example `FriendsOfTerraform`
        - Team - Team ID. For example `team-1234abcde`
        - User - User ID. For example `user-1234abcde`

        Note that you must use the Terraform Cloud API to get the [Team ID][terraform-api-team] and the [User ID][terraform-api-user].
    
    - (number) **`max_ttl_seconds = null`** _[since v0.0.7]_
    
        Specifies the max TTL (time-to-live) for tokens generated from this role

    - (number) **`ttl_seconds = null`** _[since v0.0.7]_
    
        Specifies the TTL (time-to-live) for tokens generated from this role

## Outputs

- (string) **`mount_path`** _[since v0.0.1]_

    The mount path of the secret engine

[secret-engine]:https://www.vaultproject.io/docs/secrets
[pki-secret-setup]:https://www.vaultproject.io/docs/secrets/pki#setup
[pki-config-urls]:https://www.vaultproject.io/api-docs/secret/pki#set-urls
[terraform-role]:https://www.vaultproject.io/docs/secrets/terraform#organization-team-and-user-roles
[terraform-api-team]:https://www.terraform.io/docs/cloud/api/teams.html#list-teams
[terraform-api-user]:https://www.terraform.io/docs/cloud/api/account.html
[aws-iam-policy]:https://www.vaultproject.io/docs/secrets/aws#example-iam-policy-for-vault
[aws-iam-policy-cross-account]:https://www.vaultproject.io/docs/secrets/aws#sts-assumerole
[aws-role]:https://www.vaultproject.io/docs/secrets/aws#setup