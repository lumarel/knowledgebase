# SSSD operations

## Force renewal of Kerberos Computer Account password

- `adcli update --domain=MY.DOMAIN`

## DynDNS update issues with authenticationprovider ad?

Not that easy... could be Kerberos acting up, could be something else..

The easy fix is sometimes to restart sssd, but that doesn't always work.

Rejoining the domain helps sometimes as the Kerberos token gets fully recreated.

But there is also the option to manually update via `nsupdate`:

[https://access.redhat.com/solutions/2601451](https://access.redhat.com/solutions/2601451)
