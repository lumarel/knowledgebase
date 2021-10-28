# Secureboot docs

## How to make sure that it works?

[https://access.redhat.com/articles/5337691](https://access.redhat.com/articles/5337691)

Run the following on your system to check if secureboot is enabled:

```bash
mokutil --sb-state
```

The expected response is `Secureboot enabled`

When it is disabled or not configured it will show `Failed to read Secureboot` or `Secureboot disabled`

## What about the underlying UEFI?

Most systems force Secureboot, if it is enabled in the settings, so you are unable to boot as long as the key is not in the SB enclave.
