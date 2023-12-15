---
title: "Securing SSH with the Vault SSH backend and GitHub authentication"
date: "2020-05-30"
categories: 
  - "linux"
  - "security"
  - "ssh"
---

This blog is going to be about using Hashicorp's Vault to issue short-lived certificates to use with SSH. Most guides have you using a username & password to authenticate with Vault, but I've chosen to delegate that to GitHub instead.

I'm assuming you already have a Vault server running - I won't be covering that in the course of this blog. You'll also need a sufficiently-privileged Vault token, and `jq` installed on the machine you wish to SSH from.

### Initial Vault configuration

Firstly, we need to make the SSH secrets engine available - the `path` you set will be used for all calls to the Vault API to issue certificates later on. To save time later, we'll export this as a variable we can reference again, then enable the SSH secrets engine:

```
export VAULT_PATH="ssh-client"
vault secrets enable -path=${VAULT_PATH} ssh
```

Then we need to generate a signing key and store the CA certificate (which will later be placed on all servers you want to SSH to):

```
vault write ${VAULT_PATH}/config/ca generate_signing_key=true
vault read -field=public_key ${VAULT_PATH}/config/ca > ~/trusted-user-ca-keys.pem
```

With that in place, we need to create a policy to define what operations can be carried out against the key signing path, so we'll create that and then apply it:

```
cat ssh-user-policy.hcl
path "${VAULT_PATH}/sign/ssh-user" {
    capabilities = ["create","update"]
}
vault policy write ssh-user ssh-user-policy.hcl
```

Now we need a role which defines what actions the user is allowed to carry out against the `ssh-client` endpoint. We can use these options to tweak how secure we want this setup to be:

```
cat ssh-user-role.hcl
{
    "allow_user_certificates": true,
    "allowed_users": "*",
    "default_user": "local_username",
    "allow_user_key_ids": "true",
    "default_extensions": [
        {
          "permit-pty": "",
          "permit-agent-forwarding": ""
        }
    ],
    "key_type": "ca",
    "ttl": "5m0s",
    "allow_user_key_ids": "false",
    "key_id_format": "{{token_display_name}}"
}
```

- `"allow_user_certificates": true`

- Specifies that this endpoint will generate user certificates (instead of host certificates).

- `"allowed_users": "*"`

- Using a wildcard allows this role to sign keys for any username - you can instead provide a list of usernames and the endpoint won't provide a certificate if the username is not in the list.

- `"default_user": "local_username"`

- If no user is provided then the signed key will default to this username.

- `"default_extensions": [ { "permit-pty": "" } ]`

- Sets SSH options which allow/restrict what a user can do on the remote server. `permit-pty` allows the user to get an interactive session, and `permit-agent-forwarding` allows you to forward your key with `-A`.

- `"ttl": "5m0s"`

- Setting the TTL determines how long the certificate is valid for. In this case we're setting it to 5 minutes, so you'll be able to use it to initiate an SSH session until the TTL expires. Once it expires, you'll need to make a new signing request, but as we're wrapping this in a helper function this will done automatically each time.

Now we apply that role to our Vault path:

```
vault write ${VAULT_PATH}/roles/ssh-user @ssh-user-role.hcl
```

We can check that this action completed successfully by querying the Vault API (there is no CLI command to do this):

```
curl -s --header "X-Vault-Token: ${VAULT_TOKEN}" --request LIST https://${VAULT_ADDR}/v1/ssh-client/roles | jq .
{
  "request_id": "252a2e24-2e62-dc48-35e2-c49a719065f5",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "key_info": {
      "ssh-user": {
        "key_type": "ca"
      }
    },
    "keys": [
      "ssh-user"
    ]
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```

And if you wish to check the role details:

```
curl -s --header "X-Vault-Token: ${VAULT_TOKEN}" https://${VAULT_ADDR}/v1/ssh-client/roles/ssh-user | jq .
```

### GitHub configuration

To enable GitHub to authenticate you, you'll need to first [create an organisation](https://github.com/organizations/plan) (the free plan is sufficient), then you'll need to create a team and add yourself to said team. The team name doesn't matter, but we will need it (and the organisation name) when we configure Vault in a minute. I called my team `ssh`; not exactly original but it'll get the job done.

You'll also need to create a [Personal Access Token](https://github.com/settings/tokens) with just the scope of `read:org`. Don't forget to name it something logical so you remember what the token is used for in six month's time.

### Vault + GitHub

Now we need to configure Vault's GitHub authentication endpoint. First export the org and team names as variables, then still using the privileged Vault token, enable GitHub auth:

```
export ORGNAME=myorg
export TEAM=myteam
vault auth enable github
```

Next, link Vault with your newly-created organisation:

```
vault write auth/github/config organization=${ORGNAME}
```

And then map the ssh-user policy to the GitHub team:

```
vault write auth/github/map/teams/${TEAM} value=ssh-user
```

Finally, in a different terminal, test out your access token to make sure you can log in:

```
vault login -method=github token=${VAULT_GITHUB_TOKEN}
```

### Configuring the remote server(s)

So far everything we've configured is for the client's benefit - this will all have been for nothing if the destination servers don't know to trust your Vault-signed key. To achieve that, you need to distribute the CA certificate you extracted from Vault earlier, and configure SSH to trust it.

Assuming the CA certificate is on the destination server(s), you need to copy it to `/etc/ssh` and then edit your SSHD config file to reference it:

```
sudo cp trusted-user-ca-keys.pem /etc/ssh/
echo "TrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem" | sudo tee -a /etc/ssh/sshd_config
```

Finally, restart the SSHD service to pick up the certificate.

### Tying everything together

To ensure you don't have to provide the GitHub token each time you SSH somewhere, add it to a dotfile in your home directory (and optionally ensure only you can read it):

```
echo "export VAULT_GITHUB_TOKEN=YOUR_GITHUB_TOKEN" > ~/.vault_github_token
chmod 0400 ~/.vault_github_token
```

The following functions need to be made available to your shell - they can be added to the end of your `.bashrc`.

First up is a generic function which will obtain a signed key from Vault. You'll need to adjust the `VAULT_ADDR` & `VAULT_PUBLIC_SSH_KEY` variables to suit your setup:

```
vault_sign_key () {
  VAULT_ADDR="https://vault.mydomain.com"
  VAULT_PUBLIC_SSH_KEY=~/.ssh/my_public_ssh_key.pub
  VAULT_SIGNED_KEY=$(echo ".ssh/vault_signed_key.pub")
  VAULT_GITHUB_TOKEN_PATH=~/.vault_github_token

  if [[ ! -f "${VAULT_GITHUB_TOKEN_PATH}" ]]; then
    echo "[ERR] No GitHub access token found at ${VAULT_GITHUB_TOKEN_PATH}"
    return
  fi

  source "${VAULT_GITHUB_TOKEN_PATH}"
  export TMP_DIR=$(mktemp -d)

  VAULT_PAYLOAD="{\"token\":\"${VAULT_GITHUB_TOKEN}\"}"
  VAULT_TOKEN=$(curl -s -X POST -d ${VAULT_PAYLOAD} ${VAULT_ADDR}/v1/auth/github/login | jq -r .auth.client_token)

  if ! grep -q "s." <<< $VAULT_TOKEN ; then
    echo "[ERR] Vault token not retrieved."
    return
  fi

  cat > "$(echo ${TMP_DIR}/ssh-ca.json)" << EOF
{
    "public_key": "$(cat ${VAULT_PUBLIC_SSH_KEY})",
    "valid_principals": "${SSH_USER}"
}
EOF

  if ! curl -s --fail -H "X-Vault-Token: ${VAULT_TOKEN}" -X POST -d @${TMP_DIR}/ssh-ca.json \
      ${VAULT_ADDR}/v1/ssh-client/sign/ssh-user | jq -r .data.signed_key > ${VAULT_SIGNED_KEY} ; then
    echo "[ERR] Failed to sign public key."
    exit 3
  fi
}
```

And to initiate the SSH session we use another function. In this case, you'll need to adjust the `VAULT_PRIVATE_SSH_KEY` variable to suit your setup:

```
vault_ssh () {
  if [[ -z "${1}" ]]; then
    echo "[INFO] Usage: vssh user@host [-p 2222]"
    return
  fi

  if [[ "${1}" =~ ^-+ ]]; then
    echo "[ERR] Additional SSH flags must be passed after the hostname. e.g. 'vssh user@host -p 2222'"
    return
  elif [[ "${1}" =~ ^[a-zA-Z]+@[a-zA-Z]+ ]]; then
    SSH_USER=$(echo $1 | cut -d'@' -f1)
    SSH_HOST=$(echo $1 | cut -d'@' -f2)
  else
    SSH_USER=$(whoami)
    SSH_HOST=${1}
  fi

  VAULT_PRIVATE_SSH_KEY=~/.ssh/my_private_ssh_key
  VAULT_SIGNED_KEY=$(echo ".ssh/vault_signed_key.pub")

  # sign the public key
  vault_sign_key

  # shift arguments one to the left to remove target address
  shift 1

  # construct an SSH command with the credentials, and append any extra args
  ssh -i ${VAULT_SIGNED_KEY} -i ${VAULT_PRIVATE_SSH_KEY} ${SSH_USER}@${SSH_HOST} $@
}
```

Finally, in the `.bashrc` we alias the command `vssh` to call the `vault_ssh` function above:

```
alias vssh='vault_ssh'
```

With this all in place, you can finally test it out:

```
source .bashrc
vssh user@host
```

Provided everything is properly configured you should connect to the destination server using your shiny new Vault-authed credentials.

### Notes

The `vssh` function tries to work out the format of the destination address, and will accept either `vssh user@host` or `vssh host`. If the latter option is used, then your local username will be used as the remote user (as would happen if you just ran `ssh host`).

You can also provide any extra SSH flags to `vssh` and they will be passed through to the SSH command it constructs. The only caveat here is that the host server address must come _before_ any additional flags. `vssh user@host -p 2222` will work, whereas `vssh -p 2222 user@host` will generate an error message.
