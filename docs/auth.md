# Authentication Backends

The authentication method is defined by the HTTP basic auth username, therefore multiple authentication method can be used to protect different Terraform states.

## HTTP Basic Auth

This authentication creates a hash value of provided HTTP basic auth password and state path to get the filename of the state. Therefore only the right combination of state path and password can fetch this exact state again. It's really simple to setup, no user or credential management required. The drawback is that the server can be used by everyone, who has access to the API endpoint, so it should only be used in secure or testing environments.

### Config
| Environment Variable | Type | Example | Description                                                                                     |
|----------------------|------|---------|-------------------------------------------------------------------------------------------------|
| AUTH_BASIC_ENABLED   | bool | `true`  | HTTP basic auth is enabled by default (checkout [docs/auth.md](docs/auth.md) for other options) |

**Example Terraform backend configuration**
```hcl
terraform {
  backend "http" {
    address        = "https://<terraform-state-server>/state/project1/example"
    lock_address   = "https://<terraform-state-server>/state/project1/example"
    unlock_address = "https://<terraform-state-server>/state/project1/example"
    username       = "basic"
    password       = "some-random-secret"
  }
}
```

## JSON Web Tokens

JWT allow granting access to a state for a given time (the token lifetime). The project and ID of the state must be part of the `terraform-backend` token claim.

`terraform-backend` token claim format:
```json
{
    "terraform-backend": {
        "project": "project1",
        "state": "example"
    }
}
```

NOTE: `state` value can be set to `*` to allow accessing all project states

### Config
| Environment Variable     | Type   | Example                                      | Description                                                                       |
|--------------------------|--------|----------------------------------------------|-----------------------------------------------------------------------------------|
| AUTH_JWT_OIDC_ISSUER_URL | bool   | `https://vault.example.com/v1/identity/oidc` | Issuer URL which is used to validate token (if not defined, JWT auth is disabled) |
| AUTH_JWT_OIDC_CLIENT_ID  | string | `terraform-backend` (Default)                | Client ID (string or URI) used for validating token audience claim                |


**Example Terraform backend configuration**
```hcl
terraform {
  backend "http" {
    address        = "https://<terraform-state-server>/state/project1/example"
    lock_address   = "https://<terraform-state-server>/state/project1/example"
    unlock_address = "https://<terraform-state-server>/state/project1/example"
    username       = "jwt"
    password       = "<json-web-token>"
  }
}
```

### Using another claim as project

By default the `terraform-backend` claim is used. For identity providers which cannot emit that claim, such as the OIDC ID tokens of GitHub Actions or Forgejo Actions, the project can instead be read from an existing top-level claim:

| Environment Variable   | Type   | Example         | Description                                                                                                                                                                                              |
|------------------------|--------|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| AUTH_JWT_PROJECT_CLAIM | string | `repository_id` | Name of a token claim whose string value must equal the `project` of the requested state. If defined, it replaces the `terraform-backend` claim (tokens with only that claim are rejected) and grants access to all states of the project. |

Tokens without the claim, or with a non-string or empty value, are rejected. The claim must be a unique and stable identifier which is controlled by the identity provider and not by the token requester (e.g. `repository_id`, not `sub` or a user-chosen value). As `project` is a single URL path segment, claims containing `/` (e.g. `owner/repo`) can never match.

### Example using HashiCorp Vault Identity Tokens

HashiCorp Vault allows creating [Identity Tokens](https://www.vaultproject.io/docs/secrets/identity/identity-token) for third party systems, so that Vault policies can be used to give access to specific Terraform states.

Terraform code for creating a OIDC role for accessing the state at `https://<terraform-state-server>/state/project1/exmaple`:
```hcl
resource "vault_identity_oidc_key" "example" {
  name      = "example"
  algorithm = "RS256"
}

resource "vault_identity_oidc_role" "example" {
  name = "example"
  key  = vault_identity_oidc_key.example.name
  # token is valid for one hour
  ttl  = 3600

  template = <<-EOT
    {
      "terraform-backend": {
        "project": "project1",
        "state": "example"
      }
    }
    EOT
}

resource "vault_identity_oidc_key_allowed_client_id" "example" {
  key_name          = vault_identity_oidc_key.example.name
  allowed_client_id = vault_identity_oidc_role.example.client_id
}
```

Terraform backend configuration
```hcl
terraform {
  backend "http" {
    address        = "https://<terraform-state-server>/state/project1/example"
    lock_address   = "https://<terraform-state-server>/state/project1/example"
    unlock_address = "https://<terraform-state-server>/state/project1/example"
    username       = "jwt"
  }
}
```

Terraform backend initialization (requires initialized HashiCorp Vault CLI client session):
```sh
TOKEN=$(vault read -field token identity/oidc/token/example)
terraform init -backend-config="password=$TOKEN" -reconfigure
```
