# SystemD operations

## Override parameters (i.e. docker engine)

Run `systemctl edit docker.service` and add the content

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// --dns 8.8.8.8
```

To replace a line it has first to be cleared, then it can be set again.

## Restart AuditD

This service is a little speciality.

`systemctl restart auditd` does not work, it's needed to use `service auditd restart`
