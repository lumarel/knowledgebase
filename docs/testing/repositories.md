# Repository docs

## Switch repos

```bash
sed -i -e 's/^mirrorlist.*//g' -e 's/^#baseurl/baseurl/g' /etc/yum.repos.d/*
echo "stg/rocky" > /etc/dnf/vars/contentdir
```
