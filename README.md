# smallstep-systemd-units

## Install the units

Automatically:
1. Run `sudo mkdir /etc/step`
1. Run `sudo ./install`

Manually:
1. Run `sudo mkdir /etc/step`
1. `sudo cp mv units/* /etc/systemd/system/`
1. run `systemctl daemon-reload`

## Bootstrap your CA

Pick a name for your CA. If you have a smallstep account using the first part of the CA URL is ideal. Next create a systemd drop-in with:

```
systemctl edit step-ca-bootstrap@rad.service
```
and add in between the comments. Replace your CA fingerprint and CA URL:

```
[Service]
Environment=STEP_CA_FINGERPRINT=f9d5890ade55931789f7c75110ab53064520f6e8aab65509ea53dc463e6f4911
Environment=STEP_CA_URL=https://rad.example.ca.smallstep.com
```

Next enable and start `step-ca-bootstrap@rad.service`:

```
systemctl enable --now step-ca-bootstrap@rad.service
```

and check the status to make sure your service started correctly with `systemctl status step-ca-bootstrap@rad.service`.

You can bootstrap as many CAs as you want with this systemd template. Just create a new drop-in for the next CA with a different name:

```bash
systemctl edit step-ca-bootstrap@anotherca.service
```

and set `STEP_CA_FINGERPRINT` and `STEP_CA_URL` as shown above and start the service. This will bootstrap a new CA under a new `step context` in `/etc/step`

## Issue a certificate for Hashicorp Vault

In this example we are going to automate the issuing of TLS certificates for [Hashicorp Vault](https://www.vaultproject.io/)

It's configuration for using TLS certs looks like this in `/etc/vault.d/vault.hcl `:

```hcl
# Full configuration options can be found at https://www.vaultproject.io/docs/configuration

storage "file" {
  path = "/opt/vault/data"
}

# HTTPS listener
listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/opt/vault/tls/vault.crt"
  tls_key_file  = "/opt/vault/tls/vault.key"
  tls_client_ca_file = "/opt/vault/tls/vault-ca.crt"
}
```

Be sure you configure your Vault server with the correct file paths and take note of them. You will need them for a step below.

#### Create JWK provisioner
**Note:** Before you can bootstrap your CA you need to create a [JWK provisioner](https://smallstep.com/docs/step-ca/provisioners/#jwk=) on your CA with a password. You will need that password in the drop-in below.

```bash
pwgen -1 36 |tee rad_jwk_provisioner_password
step beta ca provisioner add rad --type JWK --create --password-file=rad_jwk_provisioner_password
```

#### Create the drop-in for the service:

```
systemctl edit step-issue-cert@vault.service
```

and add in the following overrides:

```
[Unit]
After=step-ca-bootstrap@rad.service
Wants=step-ca-bootstrap@rad.service

[Service]
Environment=STEP_CONTEXT=rad
Environment=STEP_CA_PROVISIONER=rad
Environment=STEP_CA_FILE=/opt/vault/tls/vault-ca.crt
Environment=STEP_CRT_FILE=/opt/vault/tls/vault.crt
Environment=STEP_KEY_FILE=/opt/vault/tls/vault.key
Environment=STEP_FILE_OWNER=vault
Environment=STEP_FILE_GROUP=vault
Environment=STEP_FILE_MODE=0640
Environment='STEP_SANS=--san 127.0.0.1 --san localhost --san vault'
Environment=STEP_CRT_DURATION=24h
Environment=STEP_CRT_SUBJECT=yourhostname
Environment=STEP_PROVISIONER_PASSWORD=rad_jwk_provisioner_password
Environment=STEP_PROVISIONER_PASSWORD_FILE=/etc/step/profiles/rad/rad_jwk_provisioner_password
ExecStartPost=systemctl reload %i.service
```

**Note:** Be sure to set `STEP_CRT_SUBJECT` to your hostname of your server and `STEP_PROVISIONER_PASSWORD` to the password you set on your JWK provisioner. Also `STEP_CONTEXT` should match the name of your `step-ca-bootstrap@.service` so it uses the right `step context`. You will want to set the `After=` and `Wants=` to `step-ca-bootstrap@.service` so your `step-issue-cert@vault.service` starts after the CA is bootstrapped. The bootstrap should only have to happen once but we just use these Unit directives to make sure we have everything setup before we start issuing certs. If you don't want this service to reload your `vault.service` automatically remove `ExecStartPost=systemctl reload %i.service`. Lastly you can adjust how long the cert is valid for by changing `STEP_CRT_DURATION`. Valid time units are "ns", "us" (or "µs"), "ms", "s", "m", "h" (RFC 3339 format).

```
systemctl enable --now step-issue-cert@vault.service
```

and check the status to make sure your service started correctly with `systemctl status step-issue-cert@vault.service`.

You can issue certificates for as many services as you want with this systemd template. Just create a new drop-in for the next service with a different name:

```bash
systemctl edit step-issue-cert@foo.service
```

and fill in the overrides to fit your new service's needs.

## Auto-renew a certificate for Hashicorp Vault

You can now setup certificate auto-renewal. First create the systemd drop-in:

```
systemctl edit step-renew-cert@vault.service
```

and add the following overrides:

```
[Unit]
After=step-ca-bootstrap@rad.service step-issue-cert@vault.service
Wants=step-ca-bootstrap@rad.service step-issue-cert@vault.service

[Service]
Environment=STEP_CONTEXT=rad
Environment=STEP_CA_PROVISIONER=rad
Environment=STEP_CA_FILE=/opt/vault/tls/vault-ca.crt
Environment=STEP_CRT_FILE=/opt/vault/tls/vault.crt
Environment=STEP_KEY_FILE=/opt/vault/tls/vault.key
Environment=STEP_FILE_OWNER=vault
Environment=STEP_FILE_GROUP=vault
ExecStartPost=systemctl reload %i.service
```

You will want to auto reload the vault service after the certificate renews so be sure to leave `ExecStartPost=systemctl reload %i.service` and the end. This will issue a `/bin/kill --signal HUP` to the vault process and it will gracefully reload with the new TLS certificate. If your service doesn't support a SIGHUP, you will want to change this to `ExecStartPost=systemctl restart %i.service`. A restart might cause downtime to your service so please be mindful on how you issue the restart. We don't enable or start this unit. It is called by `step-renew-cert@vault.timer` every 12hr +/- 1 hour by default.

Lastly if you adjusted the `STEP_CRT_DURATION` on your certificate you will want to adjust the systemd timer that triggers `step-renew-cert@vault.service`. Create the drop-in:

```
systemctl edit step-renew-cert@vault.timer
```

```
[Timer]
OnUnitActiveSec=12hr
```

and set `OnUnitActiveSec=12hr` to something below `STEP_CRT_DURATION`. See [systemd.timer](https://www.freedesktop.org/software/systemd/man/systemd.timer.html) for more information.


# License

MIT License

Copyright (c) 2022 QuickVM

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
