# Backup and Restore etcd

# To backup etcd (please check if etcd is running in the current server/node), if not the update the endpoints with correct IP.
```
ETCDCTL_API=3 etcdctl snapshot save --endpoints=https://127.0.0.1:2379 --key=<key_file> --cert=<cert_file> --cacert=<cert_authority_cert> <backup location>.db
```

# Verify the etcd snapshot status
```
ETCDCTL_API=3 etcdctl snapshot status --endpoints=https://127.0.0.1:2379  --key=<key_file> --cert=<cert_file> --cacert=<cert_authority_cert> <backup location>.db
```

# Restore the backup
Create a data directory and make sure it has the same permission as the older data directory. If it has different permissions, please update the permissions with chmod/chown commands.
```
mkdir /var/lib/backup-etcd/
ETCDCTL_API=3 etcdctl snapshot restore --data-dir=/var/lib/backup-etcd/ <backup location>.db
```

Next, Update the running /etc/kubernetes/manifests/etcd.yaml file's hostPath to the new restored location.
So, as you can understand, this is chnaging the static manifest file. After updating this it could take few minutes for the services to return to normalcy.

Please note that, the etcdctl snapshot status is bit of a risky task since there is slight possibility it may alter the content of the snapshot. So, better to avoid it.
Also, have noticed multiple times, restore might following the restore process. Not exactly sure why though. In case it didn't restore, try again ;)
