# Ceph Cinder 整合
首先在```Controller```或```Cinder Node```上安裝 Ceph，並將 conf 檔推給串接節點：
```sh
$ ceph-deploy install <node>
```
> ```ceph-deploy``` 可以用```pip install ceph-deploy```安裝。:

在```Controller```節點新增一個 Pool 用來給 Cinder 使用：
```sh
$ ceph osd pool create volumes 128
$ ceph osd pool create backups 128
```

在```Controller```建立 Ceph 驗證與權限：
```sh
$ ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rx pool=images'
$ ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups'
```

在```Controller```複製認證金鑰給```cinder-volume```使用：
```sh
$ ceph auth get-or-create client.cinder | ssh <cinder-volume> sudo tee /etc/ceph/client.cinder.keyring
$ ssh <cinder-volume> sudo chown cinder:cinder /etc/ceph/client.cinder.keyring
```

在```Controller```複製認證金鑰給```cinder-backup```使用：
```sh
$ ceph auth get-or-create client.cinder-backup | ssh <cinder-backup> sudo tee /etc/ceph/client.cinder-backup.keyring
$ ssh <cinder-backup> sudo chown cinder:cinder /etc/ceph/client.cinder-backup.keyring
```

## Block Node
首先安裝套件：
```sh
sudo apt-get install cinder-volume python-mysqldb qemu
```

然後在編輯```/etc/ceph/ceph.conf```，加入以下內容：
```sh
...
[client.cinder]
keyring = /etc/ceph/client.cinder.keyring

[client.cinder-backup]
keyring = /etc/ceph/client.cinder-backup.keyring
```

之後編輯```/etc/cinder/cinder.conf```，並在```[DEFAULT]```部分加入以下：
```sh
[DEFAULT]
...
# volume_group = cinder-volumes
enabled_backends = rbd

[rbd]
volume_backend_name = rbd-backend
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
glance_api_version = 2

rbd_user = cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337
```
> 這邊```Cinder Server```為 ```block1```

若有使用```cinder-backup```的話，則接著加入以下內容：
```sh
[DEFAULT]
...
backup_driver = cinder.backup.drivers.ceph
backup_ceph_conf = /etc/ceph/ceph.conf
backup_ceph_user = cinder-backup
backup_ceph_chunk_size = 134217728
backup_ceph_pool = backups
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
```

完成後，重新啟動服務：
```sh
$ sudo service cinder-volume restart
$ sudo service cinder-backup restart
```
