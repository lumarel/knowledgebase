# SSSD operations

## Force renewal of Kerberos Computer Account password

- `adcli update --domain=MY.DOMAIN`

## DynDNS update issues with authenticationprovider ad?

Not that easy... could be Kerberos acting up, could be something else..

The easy fix is sometimes to restart sssd, but that doesn't always work.

Rejoining the domain helps sometimes as the Kerberos token gets fully recreated.

But there is also the option to manually update via `nsupdate`
(this works for all kinds of dns updates):

[https://access.redhat.com/solutions/2601451](https://access.redhat.com/solutions/2601451)

Another solution that seems to work in Active Directory environments is to delete the
dynamic DNS entry and restart sssd, which will just recreate the entry.
