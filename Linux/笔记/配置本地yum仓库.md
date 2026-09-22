```bash
cat > /etc/yum.repos.d/dvd.repo <<EOF
[BaseOS]
name=BaseOS
baseurl=file:///media/BaseOS
gpgcheck=0

[AppStream]
name=AppStream
baseurl=file:///media/AppStream
gpgcheck=0
EOF
```

```bash
yum clean all && yum makecache
```