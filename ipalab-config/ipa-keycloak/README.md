# FreeIPA external IDP integration with Keycloak

In this example an environment with a [FreeIPA](https://freeipa.org)
and a [Keycloak](https://keycloak.org) servers is created so that user
authentication in IPA, using an external IDP is performed.

## Preparing the environment

Create the configuration:

```
python3 -m venv /tmp/ipalab
. /tmp/ipalab/bin/activate
pip install -r requirements.txt
```

Build the container image and instantiate containers:

```
ipalab-config lab_ipa_keycloak.yml
cd ipa-keycloak-idp
podman-compose build
podman-compose up -d
```

At this point, you'll have two containers, one based on the oficial
Keycloak container image, and one that can have IPA deployed to.

Deploy the IPA cluster using
[ansible-freeipa](https://gtihub.com/freeipa/ansible-freeipa):

```
ansible-galaxy collection install \
    freeipa.ansible_freeipa \
    containers.podman
ansible-playbook -i inventory.yml \
    ${HOME}/.ansible/collections/ansible_collections/freeipa/ansible_freeipa/playbooks/install-cluster.yml
```

The provided Keycloak container uses a self-signed certificate that is
unkown to the IPA container. The certificate is found in the container
`keycloak` at the path `/opt/keycloak/conf/cert.pem`. This certificate
must be added to the list of trusted certificates on the `server`
container. This can be achieved by executing:

```
keycloak/trust_keycloak.sh server
```

## Using Keycloak web interface

Since the whole environment runs using rootles containers, in a Podman
virtual network, direct access to the host ports is not possible, but
can be achieved using `podman unshare`. For example, to _ssh_ into the
container (if `sshd` is available) or to access the `httpd` server.

When using [Firefox](https://mozilla.org/firefox) a profile is needed
to access the containers URLs, and to ease access the script
`scripts/open-firefox.sh` is provided. This script will manage the
Firefox profile and call `firefox` with the proper configuration for
`podman unshare`, allowing access to Keycloak and WebUI.

Before starting Firefox, add the entries found in the generated `hosts`
file to your machine `/etc/hosts` so the host names can be resolved. The
file `hosts` has all the containers entries needed, add it with:

```
sudo bash -c "cat hosts >> /etc/hosts"
```

Start the Keycloak web interface with:

```
scripts/open-firefox.sh https://keycloak.example.test:8443
```

In a similar fashion you can access the IPA WebUI with:

```
scripts/open-firefox.sh https://server.ipa.test:443
```


## Setting up Keycloak as an External IDP for IPA

In order to perform OAuth 2.0 device authorization grant flow against
an IdP, an OAuth 2.0 client has to be registered with the IdP and a
capability to allow the device authorization grant has to be given to it.

On Keycloak, this is achieved by setting _OAuth 2.0 Device Authorization Grant_,
in the _Authentication Flow_, to `true`.

The configuration required for the Keycloak client is:

```json
{
  "enabled" : true,
  "clientAuthenticatorType" : "client-secret",
  "redirectUris" : [ "https://${IPASERVER}/ipa/idp/*" ],
  "webOrigins" : [ "https://${IPASERVER}" ],
  "protocol" : "openid-connect",
  "attributes" : {
    "oauth2.device.authorization.grant.enabled" : "true",
    "oauth2.device.polling.interval": "5"
  }
}
```

Note that `oauth2.device.authorization.grant.enabled` is enabled.

To create the OIDC client for Keycloak, you can use the script provided
by `ipalab-config`. The script requires the IPA FQDN, an OIDC client ID
and the OIDC client password. Execute:

```
keycloak/keycloak_add_oidc_client.sh \
    server.ipa.test \
    ipa_oidc_client \
    Secret123
```

Now, we can set the external IDP on IPA, either by using the `idp-add`
CLI command, or with a playbook, using ansible-freeipa:

```
ansible-playbook -i inventory.yml playbooks/idp_keycloak.yml
```

## Testing the setup

To test the setup, create a user on Keycloak using:

```
keycloak/keycloak_add_user.sh jdoe jdoe@example.test userPASS
```

Perform login with user `jdoe` on Keycloak web interface:

```
scripts/open-firefox.sh https://keycloak.example.test:8443/realms/master/account
```

And add a user on IPA, with authorization through IDP:

```
ansible-playbook -i inventory.yml playbooks/add_user_auth_idp.yml
```

Now to authorize the new user, the commands should be execute on the
`server` container:

```
podman exec -it server bash
```

On the `server`, execute:

```
[server]$ kinit -n -c ./fast.ccache
[server]$ kinit -T ./fast.ccache jdoe
Authenticate at https://keycloak.example.test:8443/realms/master/device?user_code=GKTH-BJSS and press ENTER.:
```

Copy the provided link to the same Firefox window as `jdoe` has logged in, and grant the required authorization.

When back to the console, type ENTER, and if everything went fine, user will have a TGT for `jdoe` on IPA side:

```
[server]$ klist -A
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: jdoe@IPA.TEST

Valid starting     Expires            Service principal
05/09/25 16:34:00  05/10/25 16:18:52  krbtgt/IPA.TEST@IPA.TEST
```

## Troubleshooting

If anything goes wrong, you can search `journalctl` for `ipa-otpd`
entries.

To increase the log level, set the `oidc_child` debug level in
`/etc/ipa/default.conf` by setting:

```
[global]
oidc_child_debug_level=10
```

Valid values are between 0 and 10 and any value above 6 includes debug
output from `libcurl` utility.
