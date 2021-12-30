1. sudo apt-get update && sudo apt-get upgrade
2. sudo apt-get full-upgrade
3. sudo vim /etc/hostname
4. ssh-copy-id ubuntu@192.168.0.x
5. sudo apt-get install open-iscsi nfs-common exfat-fuse exfat-utils
6. longhorn:
Create the default configuration file /etc/multipath.conf if not existed
Add the following line to blacklist section devnode "^sd[a-z0-9]+"
blacklist {
    devnode "^sd[a-z0-9]+"
}
Restart multipath service
# systemctl restart multipathd.service

Verify that configuration is applied
# multipath -t
