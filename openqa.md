# OpenQA commands

## Test PRs

Make sure you are in the base branch of the PR (mostly develop).
You also aren't allowed to have any not checked in changes.

Then run:

```bash
git fetch origin pull/<PRnumber>/head && git checkout FETCH_HEAD
```

This only works for PRs which don't depend on another PR which hasn't been applied up to now.
For the other case you either need to locally merge the PRs in a temporary one or merge them in a fork on Github.

When you are done with testing the PR just run this to get back to the develop branch:

```bash
git switch develop
```

## Creating multi machine worker cluster

Firewall settings on workers:

```bash
firewall-cmd --add-ports={5991-5994,20000-20100}/tcp --permanent
firewall-cmd --reload
```

Firewall settings on master:

```bash
firewall-cmd --add-ports=5991-5994/tcp --permanent
firewall-cmd --reload
```

SELinux on master:

```bash
setsebool -P domain_can_mmap_files 1
```

NFS shares in fstab:

```bash
<NFShost>:/openqa_shared/hdd /var/lib/openqa/share/factory/hdd nfs nfsvers=3 0 0
<NFShost>:/openqa_shared/iso /var/lib/openqa/share/factory/iso nfs nfsvers=3 0 0
<NFShost>:/openqa_shared/tests /var/lib/openqa/share/tests nfs nfsvers=3 0 0
```
