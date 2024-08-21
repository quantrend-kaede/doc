# Slurm Control node

安裝需要的套件
> sudo apt install munge slurmd slurmctld slurmdbd mariadb-server

設定 MariaDB
> sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation

登入 MariaDB 控制台
> sudo mysql -u root -p

創建 slurm 資料庫
```
CREATE DATABASE slurm_acct_db;
CREATE USER 'slurm'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON slurm_acct_db.* TO 'slurm'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Setup slurm.conf

登入到control node
> sudo vi /mnt/data1/slurm.conf

設定 cluster
```
ClusterName=QUANTREND
SlurmctldHost=quantrend(192.168.10.178)
AuthType=auth/munge
```

設定 select type
```
SelectType=select/cons_tres
SelectTypeParameters=CR_CPU,CR_LLN
```

設定 slurmdb
```
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=localhost
AccountingStoragePort=6819
```

設定 partition
```
NodeName=quantlord NodeAddr=192.168.10.188 CPUs=144 State=UNKNOWN
NodeName=quantrend NodeAddr=192.168.10.178 CPUs=144 State=UNKNOWN
NodeName=xaos NodeAddr=192.168.10.158 CPUs=144 State=UNKNOWN
NodeName=quantland NodeAddr=192.168.10.168 CPUs=144 State=UNKNOWN
NodeName=pixiu NodeAddr=192.168.10.198 CPUs=96 State=UNKNOWN
NodeName=dragon NodeAddr=192.168.10.208 CPUs=96 State=UNKNOWN
NodeName=athena NodeAddr=192.168.10.218 CPUs=96 State=UNKNOWN
NodeName=apex NodeAddr=192.168.10.228 CPUs=96 State=UNKNOWN
NodeName=gaia NodeAddr=192.168.10.238 CPUs=96 State=UNKNOWN
NodeName=titan NodeAddr=192.168.10.248 CPUs=96 State=UNKNOWN
PartitionName=QTX Nodes=ALL Default=YES MaxTime=1440 State=UP
```

設定 munge.key
> sudo cp /etc/munge/munge.key /mnt/data1

## 設定 slurmdb
> sudo vi /etc/slurm/slurmdbd.conf
```
AuthType=auth/munge
DbdHost=localhost
DbdPort=6819
DebugLevel=info
LogFile=/var/log/slurmdbd.log
PidFile=/var/run/slurmdbd.pid
SlurmUser=slurm
StorageType=accounting_storage/mysql
StorageHost=localhost
StoragePort=3306
StoragePass=password
StorageUser=slurm
StorageLoc=slurm_acct_db
```

啟動 slurm
> sudo ln -s /mnt/data1/slurm.conf /etc/slurm
sudo systemctl restart  slurmctld
sudo systemctl enable slurmctld
sudo systemctl restart  slurmd
sudo systemctl enable slurmd
sudo systemctl restart slurmdbd
sudo systemctl enable slurmdbd

確認 slurmdbd
> sudo scontrol show config | grep AccountingStorage

其中 AccountingStorageType 應該是 accounting_storage/slurmdbd

# Slurm node

登入到每一台主機去
> sudo apt install munge slurmd

同步 munge.key 與 slurm.conf
> sudo cp /mnt/data1/munge.key /etc/munge
sudo cp /mnt/data1/slurm.conf /etc/slurm

重啟服務
> sudo systemctl restart munge
sudo systemctl restart slurmd
sudo systemctl enable slurmd

# 測試

查看 partition 狀態是否有所有的 node 都啟動
> sinfo

查看每個節點的狀態
> scontrol show nodes

加入以下測試 script
> vi /mnt/data1/count.sh
```
#!/bin/bash

counter=1
while [ $counter -le 10 ]; do
  echo $counter `hostname`
  ((counter++))
  sleep 1
done
```
> vi /mnt/data1/go.sh
```
#!/bin/bash

counter=1
while [ $counter -le 20 ]; do
  srun /mnt/data1/count.sh &
  ((counter++))
done
```
> chmod 755 /mnt/data1/count.sh
> chmod 755 /mnt/data1/go.sh

測試工作分配，應該會看到所有的 node 都有被平均分配工作
> /mnt/data1/go.sh


