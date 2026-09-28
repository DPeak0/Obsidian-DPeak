```bash
mkdir /etc/yum.repos.d/bak
mv /etc/yum.repos.d/*repo /etc/yum.repos.d/bak

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

mount /dev/sr0 /media/
yum clean all ; yum makecache
```
