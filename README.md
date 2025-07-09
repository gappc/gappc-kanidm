# gappc-iam

This project contains the files and configurations for the
IAM (Identity Access manager) [Kanidm](https://github.com/kanidm/kanidm) and Docker compose.

Follow this README for the most important steps. Take a look at the Kanidm documentation for further information.

## Table of contents

- [Start](#start)
- [Kanidm CLI](#kanidm-cli)
- [OAuth2](#oauth2)

## Start

This chapter describes the steps necessary to start Kanidm.

### Create your configuration

Create [`server.toml`](./data/server.toml) if it does not exist. The important parts you need to review and change are the `domain` and `origin` values.

> See doc at [https://kanidm.github.io/kanidm/stable/evaluation_quickstart.html](https://kanidm.github.io/kanidm/stable/evaluation_quickstart.html) for example `toml` file

### Create certificate chain

You need a certificate chain to run Kanidm. You can use your own certificate or generate a self-signed one. The certificate chain must be in PEM format and should be named `chain.pem`. Place it in the `data` directory.

```bash
openssl req -x509 -newkey rsa:4096 -keyout data/key.pem -out data/chain.pem -days 365 -nodes -subj "/CN=gappc-iam.com"
openssl x509 -in data/chain.pem -text -noout
```

### Start the container

```bash
# cleanup
# rm ./data/kanidm/db/kanidm.db* && unlink data/kanidmd.sock

# start kanidm
docker compose up
```

Open [https://localhost](https://localhost)

### Recover the Admin Role Passwords

The `admin` account is used to configure Kanidm itself.

```bash
docker exec -it kanidm kanidmd recover-account admin
```

The `idm_admin` account is used to manage persons and groups.

```bash
docker exec -i -t kanidm kanidmd recover-account idm_admin
```

## Kanidm CLI

> Note that the Kanidm CLI configuration in this repository disables certificate validation. That means, the CLI accepts also invalid certificates from the kanidm server it tries to connect to.
>
> **THIS IS NOT RECOMMENDED FOR PRODUCTION USE!!!**
>
> Set `verify_ca = true` in the [config](./data/kanidm-cli/etc/kanidm/config) file to enabled certificate validation.

You can use the Kanidm CLI with Docker to manage your Kanidm instance.

Its best to set an alias for the Docker command to make it easier to use:

```bash
# Note: the `chain.pem` below is used to verify the server certificate. It is not the correct one, but since we set `verify_ca = false` in the config, we will be able to access the server anyway.
# THIS IS NOT RECOMMENDED FOR PRODUCTION USAGE!!!

# The `--network host` option allows the Docker container to connect to a Kanidm instance on the same host

alias kanidm='docker run --rm -it \
  --network host \
  -v ./data/chain.pem:/data/chain.pem:ro \
  -v ./data/kanidm-cli/etc/kanidm/config:/data/config:ro \
  -v ./data/kanidm-cli/.config/kanidm:/root/.config/kanidm \
  -v ./data/kanidm-cli/.cache/kanidm_tokens:/root/.cache/kanidm_tokens \
  kanidm/tools:latest /sbin/kanidm'
```

### Server administration

For this chapter we assume, that you have the kanidm command available in your shell (see above).

> Note: the certificate `chain.pem` is used to verify the server certificate. Althouhg it is a valid certificate, it is not the one you need
>
> - it is **STRONGLY** advised to replace it with the correct one
> - as an alternative you can use the `--accept-invalid-certs` on the kanidm commands to skip the certificate verification for a single command (still **not recommended for production usage**)

```bash
# test if you can connect to the server
kanidm version

# list all persons
kanidm person list
```

Add a demo user:

```bash
# create a demo_user with the name "Demonstration User"
kanidm person create chris "Demonstration User" --name idm_admin

# get the demo_user
kanidm person get chris --name idm_admin

# create reset token for the demo_user. Following the created link, the user can set a new password and TOTP secret
kanidm person credential create-reset-token chris --name idm_admin

# update the demo_user credentials via CLI
kanidm person credential update chris --name idm_admin
```

## OAuth2

Use the [Kanidm CLI](#kanidm-cli) to run the following commands.

```bash
# list all groups
kanidm group list

# create new group for OAuth2 clients
kanidm group create mywebapp_users

# add members
kanidm group add-members mywebapp_users alice

# create a new OAuth2 client
kanidm system oauth2 create-public mywebapp "My Web App" https://webapp.example.com

# map groups
kanidm system oauth2 update-scope-map mywebapp mywebapp_users email openid profile groups

# enable and allow localhost redirect
kanidm system oauth2 add-redirect-url mywebapp http://localhost:3000
```

## Links

- [https://github.com/kanidm/kanidm](https://github.com/kanidm/kanidm)
- [https://kanidm.github.io/kanidm/stable/evaluation_quickstart.html](https://kanidm.github.io/kanidm/stable/evaluation_quickstart.html)
