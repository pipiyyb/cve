---

```
#启动前提
cd squashfs-root/
mount --bind /proc/ proc/
mount --bind /sys/ sys/
mount --bind /dev/ dev/
mknod -m 666 /dev/null c 1 3


# 强制删除原来的 /var 和 /tmp（无论它是文件还是死链）
rm -rf /var
rm -rf /tmp

# 重新把它们创建为真实的文件夹，并建立需要的子目录
mkdir -p /var/run
mkdir -p /var/log/lighttpd
mkdir -p /tmp/run
mkdir -p /tmp/log/lighttpd

# 再次尝试启动 lighttpd

/usr/sbin/lighttpd -f /etc/lighttpd/lighttpd.conf
```







