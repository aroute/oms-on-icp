# archive-oms-on-icp

# Phase 1: Deploy ICP and OMS

##### Stage 1

## Module One

### Access the environment

Locate [0] My Computer in your provisioned environment. Click on its monitor to open Web Console in a new browser's tab. 

Log in using the password available in the key icon of the Skytap's toolbar.

Launch putty from the desktop.

There are five putty sessions preconfigured. Launch these sessions accordingly as needed.

### Network Topology

|      | Name         | IP Address | Hostname | Description                                  |
| ---- | ------------ | ---------- | -------- | -------------------------------------------- |
| 1    | My Computer  | 172.16.1.6 | N/A      | Client computer to access the command lines. |
| 2    | ICP Master   | 172.16.1.1 | master   | ICP boot/master/proxy node.                  |
| 3    | ICP Mgmt     | 172.16.1.2 | mgmt     | ICP management node for logging/monitoring.  |
| 4.   | ICP Worker 1 | 172.16.1.3 | worker1  | ICP worker node.                             |
| 5.   | ICP Worker 2 | 172.16.1.4 | worker2  | ICP worker node.                             |
| 6.   | DB Server    | 172.16.1.5 | N/A      | DB2 server                                   |

### Setup networking

You will be setting up host names of the management, worker 1 and worker 2 nodes. The host name for the master node has been preconfigured. Setting up the host name requires a reboot.

#### 1.1 Configure host name

```console
# Putty session: 2.ICPMgmt - IP address: 172.16.1.2
login as: demo
demo@172.16.1.2's password: passw0rd
demo@mgmt:~$ sudo hostnamectl set-hostname mgmt
demo@mgmt:~$ sudo shutdown -r now

# Putty session: 3.Worker1 - IP address: 172.16.1.3
login as: demo
demo@172.16.1.3's password: passw0rd
demo@worker1:~$ sudo hostnamectl set-hostname worker1
demo@worker1:~$ sudo shutdown -r now

# Putty session: 4.Worker2 - IP address: 172.16.1.4
login as: demo
demo@172.16.1.4's password: passw0rd
demo@worker2:~$ sudo hostnamectl set-hostname worker2
demo@worker2:~$ sudo shutdown -r now
```

#### 1.2 Verify connectivity

```console
# Putty session: 1.ICPMaster - IP address: 172.16.1.1
login as: demo
demo@172.16.1.1's password: passw0rd
# Ctrl+c to break out of pings.
demo@master:~$ ping master
demo@master:~$ ping mgmt
demo@master:~$ ping worker1
demo@master:~$ ping worker2
demo@master:~$ ping localhost

# Putty session: 2.ICPMgmt - IP address: 172.16.1.2
login as: demo
demo@172.16.1.2's password: passw0rd
# Ctrl+c to break out of pings.
demo@mgmt:~$ ping master
demo@mgmt:~$ ping mgmt
demo@mgmt:~$ ping worker1
demo@mgmt:~$ ping worker2
demo@mgmt:~$ ping localhost

# Putty session: 3.ICPWorker1 - IP address: 172.16.1.3
login as: demo
demo@172.16.1.3's password: passw0rd
# Ctrl+c to break out of pings.
demo@worker1:~$ ping master
demo@worker1:~$ ping mgmt
demo@worker1:~$ ping worker1
demo@worker1:~$ ping worker2
demo@worker1:~$ ping localhost

# Putty session: 4.ICPWorker2 - IP address: 172.16.1.4
login as: demo
demo@172.16.1.4's password: passw0rd
# Ctrl+c to break out of pings.
demo@worker2:~$ ping master
demo@worker2:~$ ping mgmt
demo@worker2:~$ ping worker1
demo@worker2:~$ ping worker2
demo@worker2:~$ ping localhost
`
```

------



## Module Two

### Setup storage

You will be setting up NFS storage server (on ICP master node). The master node has been pre-configured with a second hard disk of 50GB. The rest of the ICP nodes will act as a NFS client.

#### 2.1 Set up NFS server

Perform the following steps on master node.

```console
# Putty session: 1.ICPMaster - IP address: 172.16.1.1
login as: demo
demo@172.16.1.1's password: passw0rd

demo@master:~$ sudo su -
root@master:~# ls -l /dev/sd*
root@master:~# fdisk /dev/sdb
n
p
1
Enter
Enter
p (list)
w (write)

# Partition:
root@master:~# mkfs.xfs /dev/sdb1

# Reboot
root@master:~# shutdown -r now

login as: demo
demo@172.16.1.1's password: passw0rd

demo@master:~$ sudo su -

# Install NFS server
root@master:~# apt install nfs-kernel-server -y

# Create mount point directory
root@master:~# mkdir -p /mnt/data

# Mount the disk on the mount point directory
root@master:~# mount /dev/sdb1 /mnt/data

# Grant permissions
root@master:~# chmod -R 777 /mnt/data
root@master:~# chown nobody:nogroup /mnt/data

# Add mount to persist the reboot
root@master:~# nano /etc/fstab
/dev/sdb1       /mnt/data       xfs     defaults        0       0

# Make available the NFS share directory for the clients
root@master:~# nano /etc/exports
/mnt/data 172.16.1.0/8(rw,sync,no_root_squash,no_all_squash,no_subtree_check)
/mnt/data 10.0.0.0/8(rw,sync,no_root_squash,no_all_squash,no_subtree_check)
root@master:~# exportfs -a

# Restart NFS server
root@master:~# systemctl restart nfs-kernel-server

# Write files to the mounted directory and verify the availability across the cluster.
login as: demo
demo@172.16.1.1's password: passw0rd
demo@master:~$ cd /mnt/data
demo@master:/mnt/data$ touch file_from_master_demo
# Check/verify
demo@master/mnt/data$ ls -l
```

#### 2.2 Set up NFS clients

Perform and repeat the following steps on mgmt, worker1 and worker2 nodes.

```console
login: demo
password: passw0rd

# Install NFS client
demo@mgmt:~$ sudo apt-get install nfs-common -y

# Create mount point directory
demo@mgmt:~$ sudo mkdir -p /mnt/data

# Mount NFS server's shared directory on the local mount point directory
demo@mgmt:~$ sudo mount master:/mnt/data /mnt/data

# Check/verify
demo@mgmt:~$ cd /mnt/data
demo@mgmt:~$ touch file
demo@mgmt:~$ ls -l (on client)
demo@mgmt:~$ ls -l (on server)

# Add mount to persist the reboot
demo@mgmt:~$ sudo nano /etc/fstab
master:/mnt/data        /mnt/data       nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800    0       0

# Reboot
demo@mgmt:~$ shutdown -r now

# Write files to the mounted directory and verify the availability across the cluster.
demo@mgmt:~$ cd /mnt/data
demo@mgmt:~$ touch file_from_mgmt_demo
demo@mgmt:~$ ls -l
```

Repeat the above steps on mgmt, worker 1 and worker2 nodes.

------



## Module Three

### Setup database

#### 3.1 Create new database

```console
# Putty session: 5.DBServer - IP address: 172.16.1.5
login as: demouser
demouser@172.16.1.5's password: green*8end

# From the demouser's command line, switch to db2inst1:
[demouser@dev ~]$ su - db2inst1
password: Yantra#12
[db2inst1@dev ~]$ 

[db2inst1@dev ~]$ db2start
[db2inst1@dev ~]$ db2 create db omdb
[db2inst1@dev ~]$ db2
```

#### 3.2 Create new schema

```db2
db2 => CONNECT TO OMDB

   Database Connection Information

     Database server        = DB2/LINUXX8664 11.1.0
     SQL authorization ID   = DB2INST1
     Local database alias   = OMDB

db2 => CREATE SCHEMA OMDB

	DB20000I  The SQL command completed successfully.

db2 => CREATE BUFFERPOOL SSFSBP32K IMMEDIATE SIZE 250 AUTOMATIC PAGESIZE 32K

    DB20000I  The SQL command completed successfully.

db2 => CREATE SYSTEM TEMPORARY TABLESPACE SSFSSYSTEMP PAGESIZE 32K MANAGED BY AUTOMATIC STORAGE EXTENTSIZE 16 OVERHEAD 10.5 PREFETCHSIZE 16 TRANSFERRATE 0.14 BUFFERPOOL SSFSBP32K

    DB20000I  The SQL command completed successfully.

db2 => CREATE REGULAR TABLESPACE SSFSREGULAR PAGESIZE 32K MANAGED BY AUTOMATIC STORAGE EXTENTSIZE 16 OVERHEAD 10.5 PREFETCHSIZE 16 TRANSFERRATE 0.14 BUFFERPOOL SSFSBP32K DROPPED TABLE RECOVERY ON

    DB20000I  The SQL command completed successfully.

db2 => QUIT

   DB20000I  The QUIT command completed successfully.

[db2inst1@dev ~]$
```

#### 3.3 Grant permissions

```console
[db2inst1@dev ~]$ db2 connect to OMDB

[db2inst1@dev ~]$ db2 GRANT DBADM, CREATETAB, BINDADD, CONNECT, CREATE_NOT_FENCED, IMPLICIT_SCHEMA, LOAD ON DATABASE TO USER demouser

    DB20000I  The SQL command completed successfully.

[db2inst1@dev ~]$ exit
logout
[demouser@dev ~]$
```

Following must be executed as a user other than db2inst1:

```console
[demouser@dev ~]$ . /home/db2inst1/sqllib/db2profile 

[demouser@dev ~]$ db2 connect to OMDB

   Database Connection Information

     Database server        = DB2/LINUXX8664 11.1.0
     SQL authorization ID   = DB2INST1
     Local database alias   = OMDB

[demouser@dev ~]$ db2 set schema OMDB

   DB20000I  The SQL command completed successfully.

[demouser@dev ~]$ db2 GRANT USE OF TABLESPACE USERSPACE1 TO USER DB2INST1

   DB20000I  The SQL command completed successfully.
```

#### 3.4 Configure database

```console
[demouser@dev ~]$ su - db2inst1
password: Yantra#12
[db2inst1@dev ~]$ db2set DB2_MMAP_WRITE=OFF

# Ignore errors

[db2inst1@dev ~]$ db2set DB2_MMAP_READ=OFF

# Ignore errors

[db2inst1@dev ~]$ 
db2set DB2_PINNED_BP=
db2set DB2MEMMAXFREE=
db2set DB2_ENABLE_BUFPD=
db2set DB2_USE_ALTERNATE_PAGE_CLEANING=ON
db2set DB2_EVALUNCOMMITTED=ON
db2set DB2_SKIPDELETED=ON
db2set DB2_SKIPINSERTED=ON
db2set DB2_SELECTIVITY=ALL
db2set DB2LOCK_TO_RB=STATEMENT
db2set DB2_NUM_CKPW_DAEMONS=0
```

After running all of these command, you should have a database configured for the db2inst1 user to access.  

**DB Setup on Windows machine (FYI only - do not apply)**

```dos
db2cmd -i -w db2clpsetcp 
echo %DB2CLP%
db2 create db omdb
db2
CONNECT TO OMDB
CREATE SCHEMA OMDB
CREATE BUFFERPOOL SSFSBP32K IMMEDIATE SIZE 250 AUTOMATIC PAGESIZE 32K
CREATE SYSTEM TEMPORARY TABLESPACE SSFSSYSTEMP PAGESIZE 32K MANAGED BY AUTOMATIC STORAGE EXTENTSIZE 16 OVERHEAD 10.5 PREFETCHSIZE 16 TRANSFERRATE 0.14 BUFFERPOOL SSFSBP32K
CREATE REGULAR TABLESPACE SSFSREGULAR PAGESIZE 32K MANAGED BY AUTOMATIC STORAGE EXTENTSIZE 16 OVERHEAD 10.5 PREFETCHSIZE 16 TRANSFERRATE 0.14 BUFFERPOOL SSFSBP32K DROPPED TABLE RECOVERY ON
db2 set schema OMDB
db2set DB2_MMAP_WRITE=OFF
(Ignore errors)
db2set DB2_MMAP_READ=OFF
(Ignore errors)
db2set DB2_PINNED_BP=
db2set DB2MEMMAXFREE=
db2set DB2_ENABLE_BUFPD=
db2set DB2_USE_ALTERNATE_PAGE_CLEANING=ON
db2set DB2_EVALUNCOMMITTED=ON
db2set DB2_SKIPDELETED=ON
db2set DB2_SKIPINSERTED=ON
db2set DB2_SELECTIVITY=ALL
db2set DB2LOCK_TO_RB=STATEMENT
db2set DB2_NUM_CKPW_DAEMONS=0
```

#### 3.5 Configure DBVisualizer tool

1. Click on DBVisualizer icon from the desktop.
2. Select the default Windows theme and click Continue.
3. Name your connection in the New Wizard Screen. Name it OMSLocal or anything else you prefer.
4. Select DB2 as a Database Driver from the list.
5. Double-click Database Server field and type `172.16.1.5`which is your DB server's IP address.
6. Double-click the Database and type `omdb` (which is the database you created following the above steps 3.1 to 3.5).
7. Double-click Database Userid and type `db2inst1`
8. Double-click Database Password and type `Yantra#12`
9. Click Ping Server button to test the connectivity. Click OK at the successful connect message.
10. Click Finish to finish the New Connection Wizard.
11. From the left-hand pane, under your OMSLocal connection, expand the `OMDB` database.
12. Check that the tables are not filled yet.

------



##### Stage 2

## Module Four

### Setup IBM Cloud Private (ICP)

**Setup docker on mgmt, worker1 and worker2 (FYI only - do not apply)**

```console
# Putty session: 1.ICPMaster - IP address: 172.16.1.1
login as: demo
demo@172.16.1.1's password: passw0rd
demo@master:~$ sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
demo@master:~$ sudo tee setupdocker.sh <<EOF
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install docker-ce=18.03.1~ce-0~ubuntu -y
sudo docker run hello-world
sudo usermod -aG docker demo
EOF
demo@master:~$ sudo chmod +x setupdocker.sh
demo@master:~$ ./setupdocker.sh
demo@master:~$ exit
login as: demo
demo@172.16.1.1's password: passw0rd
demo@master:~$ docker run hello-world
demo@master:~$ docker -v
```

#### 4.1 Load install file

```console
login as: demo
demo@172.16.1.1's password: passw0rd
demo@master:~$ cd /opt/images/icp
demo@master:~$ sudo tee load.sh <<EOF
tar xf ibm-cloud-private-x86_64-3.1.2.tar.gz -O | sudo docker load
EOF
demo@master:~$ sudo chmod +x load.sh

# launch load.sh (45 minutes wait).

demo@master:~$ sudo mkdir /opt/ibm-cloud-private-3.1.2 && cd /opt/ibm-cloud-private-3.1.2
demo@master:~$ sudo docker run -v $(pwd):/data -e LICENSE=accept \
ibmcom/icp-inception-amd64:3.1.2-ee \
cp -r cluster /data
```

#### 4.2 Configure install (config file)

```console
demo@master:/opt/ibm-cloud-private-3.1.2/cluster$ sudo cp config.yaml config.bak
demo@master:/opt/ibm-cloud-private-3.1.2/cluster$ sudo cp hosts hosts.bak
demo@master:/opt/ibm-cloud-private-3.1.2/cluster$ vi config.yaml
```

```yaml
## Advanced Settings
default_admin_user: admin
default_admin_password: admin
password_rules:
 - '(.*)'
ansible_user: demo
ansible_ssh_pass: passw0rd
ansible_ssh_common_args: "-oPubkeyAuthentication=no"
ansible_become: true
ansible_become_password: passw0rd
```

#### 4.3 Configure install (inventory file)

```console
demo@master:/opt/ibm-cloud-private-3.1.2/cluster$ sudo vi hosts
```

```shell
[master]
172.16.1.1

[worker]
172.16.1.3
172.16.1.4

[proxy]
172.16.1.1

[management]
172.16.1.2

#[va]
#5.5.5.5
```

#### 4.4 Move install file

```console
demo@master:/opt/ibm-cloud-private-3.1.2/cluster$ sudo mkdir images
demo@master:/opt/ibm-cloud-private-3.1.2/cluster$ sudo mv /opt/images/icp/ibm-cloud-private-x86_64-3.1.2.tar.gz images/
```

------



##### Stage 3

## Module Five

### Install ICP

#### 5.1 Setup installation command

```console
demo@master:/opt/ibm-cloud-private-3.1.2/cluster$ sudo tee install.sh <<EOF
sudo docker run --net=host -t -e LICENSE=accept \
-v "$(pwd)":/installer/cluster ibmcom/icp-inception-amd64:3.1.2-ee install
EOF
demo@master:/opt/ibm-cloud-private-3.1.2/cluster$ sudo chmod +x install.sh
```

#### 5.2 Install ICP (1.5 hours wait)

```console
demo@master:/opt/ibm-cloud-private-3.1.2/cluster$ sudo ./install.sh
```

Wait 1.5 hours. After the installation is finished, you will be shown your ICP dashboard's URL.

```console
demo@master:/opt/ibm-cloud-private-3.1.2/cluster$ cd ~/
```

#### 5.3 Log in to ICP

```console
demo@master:~$ cloudctl login -a https://172.16.1.1:8443 --skip-ssl-validation -u admin -p admin -c id-mycluster-account -n default
```

#### 5.4 Helm Initiatization (one-time verification)

```console
demo@master:~$ helm init --client-only

demo@master:~$ helm version --tls
```

#### 5.5 Create Log-in script for daily use

```console
demo@master:~$ tee connecticp.sh <<EOF
cloudctl login -a https://172.16.1.1:8443 --skip-ssl-validation -u admin -p admin -c id-mycluster-account -n default
EOF

demo@master:~$ sudo chmod +x connecticp.sh

demo@master:~$ ./connecticp.sh
```

#### 5.6 Test Helm - Deploy & Delete Jenkins on ICP

```console
demo@master:~$ helm repo add ibm-charts https://raw.githubusercontent.com/IBM/charts/master/repo/stable/
demo@master:~$ helm repo update
```

```console
demo@master:~$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-release-jenkins
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /mnt/data/my-release-jenkins
EOF
```

```console
demo@master:~$ helm install --name my-jenkins-release ibm-charts/ibm-jenkins-dev --tls
```

```console
demo@master:~$ kubectl get pods
verify pods running 1/1
demo@master:~$ export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services my-jenkins-release-ibm-j)
demo@master:~$ echo http://172.16.1.1:$NODE_PORT/login
```

1. Access the resulting URL via the browser. You should be able to see Jenkins UI (log in if you want: admin/admin).

2. Verify persistent data by browsing `cd /srv/share/data`
3. This confirms that your ICP and Helm are setup correctly. Delete Jenkins and move on with OMS setup.

#### 5.7 Delete Jenkins

```console
demo@master:~$ helm delete --purge my-jenkins-release --tls
demo@master:~$ kubectl delete pv my-release-jenkins
demo@master:~$ cd /mnt/data
demo@master:~$ sudo rm -r my-release-jenkins
```

------



##### Stage 4

## Module Six

### Setup Order Management (OMS)

#### 6.1 Load OMS ppa archive

```console
demo@master:~$ cd /opt/images/ppa/L2
demo@master:/opt/images/ppa/L2$ tee dockerload.sh <<EOF
docker login -u=admin -p=admin mycluster.icp:8500
cloudctl catalog load-archive --archive IBM_OMS_ENT_ICP_V10_ML_MP.tgz
EOF

demo@master:/opt/images/ppa/L2$ chmod +x dockerload.sh
demo@master:/opt/images/ppa/L2$ ./dockerload.sh
```

Wait 20+ minutes for the upload to finish.

#### 6.2 Create persistent volume

(multi-node NFS - use this)

```
cat <<EOF | kubectl create -f -
kind: PersistentVolume
apiVersion: v1
metadata:
  name: oms-common
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  nfs:
    path: "/mnt/data"
    server: "172.16.1.1"
EOF
```

(single-node hostpath - on-hold)

```
cat <<EOF | kubectl create -f -
kind: PersistentVolume
apiVersion: v1
metadata:
  name: oms-common
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  hostPath:
    path: "/srv/share"
EOF
```

#### 6.3 Create Datasource Secret

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Secret
metadata:
  name: "omsprod-oms-secret"
type: Opaque
stringData:
  consoleadminpassword: "admin"
  consolenonadminpassword: "noadmin"
  dbname: "omdb"
  dbServerName: "172.16.1.5"
  dbport: "50000"
  dbuser: "db2inst1"
  dbpassword: "Yantra#12"
  datasourceName: "jdbc/OMDS"
  system_overrides.properties: |-
    jdbcService.db2Pool.dbvendor=DB2
    jdbcService.db2Pool.systemPool=true
    jdbcService.db2Pool.url=jdbc:db2://172.16.1.5:50000/omdb
    jdbcService.db2Pool.dbname=omdb
    jdbcService.db2Pool.user=db2inst1
    jdbcService.db2Pool.password=Yantra#12
    jdbcService.db2Pool.schema=omdb
    jdbcService.db2Pool.datasource=jdbc/OMDS
EOF
```

(on hold) Create Datasource Secret

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Secret
metadata:
  name: "omsprod-oms-secret"
type: Opaque
stringData:
  consoleadminpassword: "admin"
  consolenonadminpassword: "noadmin"
  dbpassword: "Yantra#12"
EOF
```

#### 6.4 Setup DNS on ICP, DB and IE client machines

Edit /etc/hosts file and insert or uncomment myoms.icp next to 172.16.1.

(on-hold) Create TLS certificate

```shell
$ openssl req -x509 -nodes -sha256 -subj "/CN=myoms.icp" -days 365 -newkey rsa:2048 -keyout cert.key -out cert.crt(on-hold) Create Ingress secret
```

```shell
$ kubectl create secret tls omsprod-ingress-secret --key cert.key --cert cert.crt -n default
```



##### Stage 5

## Module Seven

### Install OMS

#### 7.1 Deploy Helm Chart from ICP Catalog UI

```
Release name: omsprod
Namespace: default
Image repo: mycluster.icp:8500/default
Application Secret: omsprod-oms-secret
Database Server IP: 172.16.1.5
Database port: 50000
Database name: OMDB
Database user: db2inst1
Database schema name: OMDB
Ingress host: myoms.icp
Secret name for ingress host: omsprod-ingress-secret
Load data: yes
```

#### 7.2 Deploy Helm Chart from command line

Download and extract Helm chart from ICP Catalog.

Edit `values.yaml`

```shell
$ helm install --name omsprod ./ibm-oms-ent-prod/ --tls
```

#### 7.3 Watch and Monitor the Logs

```
kubectl get pods
kubectl logs -f <data setup pod name>
```
1. Wait until the log stops scrolling.
2. Check cd mnt/data and verify datasetup.complete file
```
kubectl get pods
kubectl logs -f <app server pod name>
```

#### 7.4 Delete Helm Chart from ICP Catalog UI (if and when needed)

```
Helm Release - Delete
Delete recently created ConfigMaps
Delete relevant Persistent Volume
```

#### 7.5 Delete Helm Chart from command line (if and when needed)

```shell
$ helm delete omsprod --purge --tls
$ kubectl delete cm omsprod-ibm-oms-ent-prod-config
$ kubectl delete cm omsprod-ibm-oms-ent-prod-def-server-xml-conf
$ kubectl delete pv oms-common
```

# Upgrade & Scale Order Management on Cloud Private

#### Phase 2

# Exercise 1 - Preparation

## 1.1 Setup Terminal

1.1.1 Click on Visual Studio Code icon from the Taskbar.

1.1.2 Click on View - Terminal if you do not see the terminal pane.

1.1.3 Type 'bash' if the prompt is not bash.

1.1.4 From the Bash terminal, change directory.

```sh
cd Downloads/
```

1.1.5 From the Taskbar, launch Git Bash

1.1.6 Connect (SSH) to master node.

```console
$ ssh demo@172.16.1.1
yes
passw0rd
```

1.1.7 Log in to IBM Cloud Private (ICP) via command line.

```console
$ ./connecticp.sh
```

## 1.2 Setup CLIs


1.2.1 From the Taskbar, launch Command Prompt.

1.2.2 Download `cloudctl`, `kubectl` and `helm` command line interface utilities.

```cmd
C:\Users\demo\Downloads> curl -kLo cloudctl-win-amd64-3.1.2-1203.exe https://172.16.1.1:8443/api/cli/cloudctl-win-amd64.exe
C:\Users\demo\Downloads> curl -kLo kubectl-win-amd64-v1.12.4.exe https://172.16.1.1:8443/api/cli/kubectl-win-amd64.exe
C:\Users\demo\Downloads> curl -kLo helm-win-amd64-v2.9.1.tar.gz https://172.16.1.1:8443/api/cli/helm-win-amd64.tar.gz
```

1.2.3 Move `cloudctl` and `kubectl` utilities to system's path/directory. Log in to the cluster and verify.

```cmd
C:\Users\demo\Downloads> move cloudctl-win-amd64-3.1.2-1203.exe c:\Windows\System32\cloudctl.exe
C:\Users\demo\Downloads> move kubectl-win-amd64-v1.12.4.exe c:\Windows\System32\kubectl.exe
C:\Users\demo\Downloads> cloudctl --help
C:\Users\demo\Downloads> kubectl --help
C:\Users\demo\Downloads> cloudctl login -a https://172.16.1.1:8443 --skip-ssl-validation -u admin -p admin -c id-mycluster-account -n default
```

1.2.4 Unzip `Helm` utility, move system's path/directory and activate.

```cmd
C:\Users\demo\Downloads> 7z e helm-win-amd64-v2.9.1.tar.gz
C:\Users\demo\Downloads> 7z x helm-win-amd64-v2.9.1.tar
C:\Users\demo\Downloads> move C:\Users\demo\Downloads\windows-amd64\helm.exe C:\Windows\System32\helm.exe

C:\Users\demo\Downloads> helm init --client-only
C:\Users\demo\Downloads> helm version --tls

When Windows Firewall prompts, click to grant access.
```

## 1.3 Download OMS PPA


### 1.3.1 Download OMS Helm chart

1.3.1.1 Download OMS Helm Chart via Curl.

```cmd
C:\Users\demo\Downloads> curl -kLo ibm-oms-ent-prod-2.0.0.tgz https://172.16.1.1:8443/helm-repo/requiredAssets/ibm-oms-ent-prod-2.0.0.tgz
```

1.3.1.2 Unzip the downloaded *.tgz file to extract the *.tar file.

```cmd
C:\Users\demo\Downloads> 7z e ibm-oms-ent-prod-2.0.0.tgz
```

1.3.1.3 Extract the *.tar file as a folder in Documents folder.

```cmd
C:\Users\demo\Downloads> 7z x -oC:\Users\demo\Documents ibm-oms-ent-prod-2.0.0.tar
```

[//]: # (```sh C:\Users\demo\Downloads> 7z x -o/c/Users/Administrator/Documents ibm-oms-ent-prod-2.0.0.tar```)

### 1.3.2 Download MQ

1.3.2.1 Download MQ Helm Chart via Curl.

```cmd
C:\Users\demo\Downloads> curl -kLo ibm-mqadvanced-server-dev-3.0.1.tgz https://raw.githubusercontent.com/IBM/charts/master/repo/stable/ibm-mqadvanced-server-dev-3.0.1.tgz
```

1.3.2.2 Unzip the downloaded *.tgz file to extract the *.tar file.

```cmd
C:\Users\demo\Downloads> 7z e ibm-mqadvanced-server-dev-3.0.1.tgz
```

1.3.2.3 Exract the *.tar file as a folder in Documents folder.

```cmd
C:\Users\demo\Downloads> 7z x -oC:\Users\demo\Documents ibm-mqadvanced-server-dev-3.0.1.tar
```

[//]: # (```cmd C:\Users\demo\Downloads> 7z x -o/c/Users/Administrator/Documents ibm-mqadvanced-server-dev-3.0.1.tar```)


## 1.4 Setup Log-in routine for IBM Cloud Private (ICP)


### 1.4.1 Create Log in script file

1.4.1.1 Open VS Code and open a new file.

1.4.1.2 Copy/paste the log in string. Save file as `connecticp` Save As 'Batch' in \Downloads folder.

```cmd
cloudctl login -a https://172.16.1.1:8443 --skip-ssl-validation -u admin -p admin -c id-mycluster-account -n default
```

### 1.4.2 Log in to ICP

1.4.2.1 From the VSCode Terminal, type `cmd` to ensure you are in command prompt.

1.4.2.2 Type `connecticp` and hit enter to log in.

# Exercise 2 - Setup Domain name & Certificates

## 2.1 Setup Certificate Authority (CA)

```console
ssh demo@172.16.1.1
passw0rd

$ mkdir certs
$ cd certs

$ openssl genrsa -out ca.key 4096
$ openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=US/ST=CA/L=LA/O=WCE/OU=Demo/CN=myca.com" -key ca.key -out ca.crt
```


## 2.2 Setup My Certificate

### 2.2.1 Generate My Private Key

```console
$ openssl genrsa -out myoms.key 4096
```


### 2.2.2 Create My Certificate Signing Request (CSR)

```console
$ openssl req -sha512 -new -subj "/C=US/ST=CA/L=LA/O=WCE/OU=Demo/CN=myoms.icp" -key myoms.key -out myoms.csr

$ cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1=myoms.icp
DNS.2=myoms
EOF
```

### 2.2.3 Generate My Certificate

```console
$ openssl x509 -req -sha512 -days 3650 -extfile v3.ext -CA ca.crt -CAkey ca.key -CAcreateserial -in myoms.csr -out myoms.crt
```

### 2.2.4 Convert My Certificate (optional)

```console
$ openssl x509 -inform PEM -in myoms.crt -out myoms.cert
```


### 2.2.5 Export My Certificate


```sh
$ scp demo@172.16.1.1:~/certs/\{myoms.crt,myoms.key,ca.crt\} /c/Users/demo/Documents/ibm-oms-ent-prod/
passw0rd
```
[//]: # (```sh $ scp demo@172.16.1.1:~/certs/\{myoms.crt,myoms.key,ca.crt\} ../Documents/ibm-oms-ent-prod/ passw0rd```)


## 2.3 Setup DNS to access the OMS websites

2.3.1 Click on the Windows Start menu and type 'Notepad' to search. Right-click Notepad from the Start menu and select Run as Administrator. Click Yes.

2.3.2 Click File - Open. From the bottom-right, drop-down the Text Documents(\*.txt) and select All files. Click hosts file to open (the first host file you see at the top).

2.3.3 Uncomment the myoms.icp entry. Remove the "#".

2.3.4 Save the host file. Ctrl+s.

2.3.5 From the Command prompt terminal, ping `myoms.icp` to ensure you can ping the domain name.



# Exercise 3 - Prepare and deploy Order Management Helm Chart


## 3.1 Open Helm Chart in VSCode

3.1.1 From VSCode, open folder: \Documents\ibm-oms-ent-prod\

3.1.2 If not open already, open Terminal (View - Terminal). Type `cmd` to switch to command prompt terminal.

## 3.2 Create Kubernetes Resources

Follow the steps to create persistent volume and secret resources.

### 3.2.1 Create Persistent Storage Volume

3.2.1.1 Open new file and copy/paste the content. 

```YAML
kind: PersistentVolume
apiVersion: v1
metadata:
  name: oms-common
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  nfs:
    path: "/mnt/data"
    server: "172.16.1.1"
```

3.2.1.2 Ctrl+s to save the file. Name the file `oms-common`. Select YAML as Save-as type.

3.2.1.3 Deploy the YML file.

```cmd
C:\Users\demo\Documents\ibm-oms-ent-prod> kubectl create -f oms-common.yml
```


### 3.2.2 Create Secret (Database properties)

3.2.2.1 Open new file and copy/paste the content.

```YAML
apiVersion: v1
kind: Secret
metadata:
  name: "omsprod-db-secret"
type: Opaque
stringData:
  consoleadminpassword: "admin"
  consolenonadminpassword: "noadmin"
  dbpassword: "Yantra#12"
```

3.2.2.2 Ctrl+s to save the file. Name the file `omsprod-db-secret`. Select YAML as Save-as type.

3.2.2.3 Deploy the YML file to create the Secret.

```cmd
C:\Users\demo\Documents\ibm-oms-ent-prod> kubectl create -f omsprod-db-secret.yml
```


### 3.2.3 Create Secret (Certificate/Ingress properties)

3.2.3.1 Create a secret for the certificate.

```cmd
C:\Users\demo\Documents\ibm-oms-ent-prod> kubectl create secret tls omsprod-ingress-secret --key myoms.key --cert myoms.crt -n default
```


## 3.3 Edit Values.yaml in VSCode


3.3.1 Open values.yaml file.

3.3.2 From line 3, under the keys `global:` `image:` `repository:` remove the last "/" from the end of the value.

```YAML
    repository: mycluster.icp:8500/default
```

3.3.3 From line 4, key `appSecret:` type `omsprod-db-secret` as its value.

```YAML
  appSecret: omsprod-db-secret
```

3.3.4 From line 5 to 9, type the following as the values.

```YAML
  database:
    serverName: 172.16.1.5
    port: 50000
    dbname: OMDB
    user: db2inst1
    dbvendor: DB2
    datasourceName: jdbc/OMDS
    systemPool: true
    schema: OMDB
 ```

3.3.5 From line 69 to 74, type the following as the values.

```YAML
  ingress:
    enabled: true
    host: myoms.icp
    ssl:
      enabled: true
      secretname: omsprod-ingress-secret
 ```
 
 3.3.6 Line 132, type `install` as value of the key `loadFactoryData`.
 
 ```YAML
 datasetup:
  loadFactoryData: install
  mode: create
```

3.3.7 Ctrl+s to save the values.yaml file

## 3.4 Deploy the chart using the Helm command


```cmd
C:\Users\demo\Documents\ibm-oms-ent-prod> helm install --name omsprod -f values.yaml . --tls
```

## 3.5 Watch and Monitor the Logs


```console
$ kubectl get pods
$ kubectl logs -f <data setup pod name>
```

## 3.6 Delete the first deployment


### 3.6.1 Delete Kubernetes resources

3.6.1.1 Delete Helm release, Configmaps and Persistent Volume.

```console
$ helm delete omsprod --purge --tls
$ kubectl delete cm omsprod-ibm-oms-ent-prod-config
$ kubectl delete cm omsprod-ibm-oms-ent-prod-def-server-xml-conf
$ kubectl delete pv oms-common
$ cd /mnt/data
$ rm datasetup.complete
```

## 3.7 Re-deploy Order Management Helm Chart


### 3.7.1 Re-create Persistent Volume

```cmd
C:\Users\demo\Documents\ibm-oms-ent-prod> kubectl create -f oms-common.yml
```

## 3.8 Re-deploy the chart using the Helm command

```cmd
C:\Users\demo\Documents\ibm-oms-ent-prod> helm install --name omsprod -f values.yaml . --tls
```

## 3.9 Watch and Monitor the Logs


```console
kubectl get pods
kubectl logs -f <data setup pod name>

    Wait until the log stops scrolling. 
```


```console
demo@master:/mnt/data$ kubectl get pods
NAME                                                      READY   STATUS      RESTARTS   AGE
omsprod-ibm-oms-ent-prod-appserver-59bd86497d-xsxcs       1/1     Running     0          14m
omsprod-ibm-oms-ent-prod-datasetup-4bp7q                  0/1     Completed   0          14m
omsprod-ibm-oms-ent-prod-healthmonitor-668f9d5c7d-v5gr9   1/1     Running     0          14m
```



## 3.10 Import certificates


#### 3.10.1.1 Firefox

3.10.1.1.1 Go to Firefox Options - Search for Certificates, click View Certificates. Go to Authorities, click Import.

3.10.1.1.2 Browse to `ibm-oms-end-prod` directory and select `ca` certificate file.

3.10.1.1.3 Check the two Trust this ... boxes. Click OK twice to close the properties.


#### 3.10.1.2 Internet Explorer

3.10.1.2.1 Launch IE. When prompted, click Enable on Java security.

3.10.1.2.2 Go to Internet Options - Content - Certificates - Trusted Root Certification Authority. Click Import. Click Next. Browse to `ibm-oms-end-prod` directory and select `ca` certificate file. Accept default dialog (Next - Place all ... Next - Finish - Yes - OK - Close).

3.10.1.2.3 Go to Security tab - Local Intranet - Sites - Advanced. Insert `https://myoms.icp`. Cick Close. Click OK.

#### 3.10.1.3 Java

3.10.1.3.1 From the Start Menu, type `Control Panel` to search and open the Control Panel. Search for `Java` to open Java properties. Click on Security tab. Click on `Edit Site list`. Click `Add`. Type `https://myoms.icp/*`. Click OK to close.

[//]: # (4.2.4.3.3 Advanced - Select DO NOT CHECK - not recommended - on Perform signed code certificate revocation checks on)

[//]: # (4.2.4.3.4 Advanced - Select DO NOT CHECK - not recommended - on Perform TLS certificate revocation checks on)


## 3.11 Access OMS websites.

- Launch **smcfs** in IE: https://myoms.icp/smcfs (admin/password).
- Launch **sbc** in Firefox: https://myoms.icp/sbc (admin/password).
- FYI. **sma**: https://myoms.icp/sma
- FYI. **isccs**: https://myoms.icp/isccs
- FYI. **wsc**: https://myoms.icp/wsc
- FYI. **adminCenter**: https://myoms.icp/adminCenter


# Exercise 4 - Prepare and deploy MQ Helm Chart


## 4.1 Setup mount folder.

```console
ssh demo@172.16.1.1
passw0rd
mkdir -p /mnt/data/mq
chmod 777 -R /mnt/data/mq
```


## 4.2 Open Helm Chart in VSCode

4.2.1 From VSCode, open a new window, open folder: \Documents\ibm-mqadvanced-server-dev\

4.2.2 Open Terminal (View - Terminal). Switch to `cmd`.

## 4.3 Create Persistent Storage Volume

4.3.1 Open new file and copy/paste the content. 

```YAML
kind: PersistentVolume
apiVersion: v1
metadata:
  name: mq-common
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  nfs:
    path: "/mnt/data/mq"
    server: "172.16.1.1"
```

4.3.2 Ctrl+s to save the file. Name the file `mq-common`. Select YAML as Save-as type.

4.3.3 Deploy the YML file.

```cmd
C:\Users\demo\Documents\ibm-mqadvanced-server-dev> kubectl create -f mq-common.yml
```

## 4.4 Edit Values.yaml in VSCode

4.4.1 Open values.yaml file.

4.4.2 From line 16, change `not accepted` to 'accept`.

```YAML
license: "not accepted"
```

4.4.3 From line 54 - 55, key `type:` change `ClusterIP` to `NodePort` as its value.

```YAML
service:
  type: NodePort
  ```
  
4.4.4 From line 76 - 78, key `name:` type `qmgr` as its value.

```YAML
queueManager:
  name: qmgr
```

4.4.5 From line 80 - 82, key `adminPassword` type `password` as its value.

```YAML
  dev:
    adminPassword: password
```

4.4.6 Ctrl+s to save the values.yaml file

4.4.7 Deploy the MQ chart using the Helm command

```cmd
C:\Users\demo\Documents\ibm-mqadvanced-server-dev> helm install --name mqprod -f values.yaml . --tls
```

## 4.5 Watch and Monitor the Logs

```console
$ kubectl get all -l release=mqprod

    Wait until the status of the pod is READY 1/1
    
NAME                  READY   STATUS    RESTARTS   AGE
pod/mqprod-ibm-mq-0   1/1     Running   0          6m41s

NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                         AGE
service/mqprod-ibm-mq           NodePort    10.0.171.77   <none>        9443:31046/TCP,1414:31608/TCP   6m41s
service/mqprod-ibm-mq-metrics   ClusterIP   10.0.70.237   <none>        9157/TCP                        6m41s
```


## 4.6 Expose Ports


### 4.6.1 Expose MQ Channel port 1414

```console
$ kubectl expose pod mqprod-ibm-mq-0 --port 31608 --name mqchannel --type NodePort
```
### 4.6.2 Expose MQ Web Console port 9443

```console
$ kubectl expose pod mqprod-ibm-mq-0 --port 31046 --name mqwebconsole --type NodePort
```

### 4.7 Identify the exposed ports

```console
$ kubectl get service -l release=mqprod
```
```console
RESULTS

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
mqchannel               NodePort    10.0.48.183    <none>        31608:30931/TCP                 3m11s
mqprod-ibm-mq           NodePort    10.0.171.77    <none>        9443:31046/TCP,1414:31608/TCP   15m
mqprod-ibm-mq-metrics   ClusterIP   10.0.70.237    <none>        9157/TCP                        15m
mqwebconsole            NodePort    10.0.246.213   <none>        31046:30639/TCP                 3m3s
```


# Exercise 5 - Setup MQ/JMS Queue for OMS

## 5.1 Access MQ Command line

```console
kubectl exec -it mqprod-ibm-mq-0 /bin/bash
```

## 5.2 Setup MQ & Create Queue

```console
(mq:9.1.2.0)mqm@mqprod-ibm-mq-0:/$ cd /mnt/mqm/data

(mq:9.1.2.0)mqm@mqprod-ibm-mq-0:/mnt/mqm/data$ tee queues.in <<EOF
DEFINE CHANNEL(SYSTEM.ADMIN.SVRCONN) CHLTYPE(SVRCONN)
SET CHLAUTH(*) TYPE(BLOCKUSER) USERLIST('nobody','*MQADMIN')
SET CHLAUTH(SYSTEM.ADMIN.SVRCONN) TYPE(BLOCKUSER) USERLIST('nobody') 
ALTER QMGR CHLAUTH(DISABLED)
DEFINE AUTHINFO(SYSTEM.DEFAULT.AUTHINFO.IDPWOS) AUTHTYPE(IDPWOS)
ALTER QMGR CONNAUTH(SYSTEM.DEFAULT.AUTHINFO.IDPWOS)
ALTER AUTHINFO(SYSTEM.DEFAULT.AUTHINFO.IDPWOS) AUTHTYPE(IDPWOS) CHCKCLNT(NONE)
REFRESH SECURITY TYPE(CONNAUTH)
define qlocal(DEFAULTAGENTQUEUE) DEFPSIST(YES) DEFBIND(NOTFIXED) 
EOF

(mq:9.1.2.0)mqm@mqprod-ibm-mq-0:/mnt/mqm/data$ runmqsc qmgr < queues.in
```


## 5.3 Setup Java Message Service (JMS)


```console
(mq:9.1.2.0)mqm@mqprod-ibm-mq-0:/mnt/mqm/data$ chmod 777 /opt/mqm/java/bin
(mq:9.1.2.0)mqm@mqprod-ibm-mq-0:/mnt/mqm/data$ sed -i 's#file:///home/user/JNDI-Directory#file:///mnt/mqm/data/jndi#g' /opt/mqm/java/bin/JMSAdmin.config
(mq:9.1.2.0)mqm@mqprod-ibm-mq-0:/mnt/mqm/data$ cat /opt/mqm/java/bin/JMSAdmin.config
(mq:9.1.2.0)mqm@mqprod-ibm-mq-0:/mnt/mqm/data$ mkdir -p /mnt/mqm/data/jndi
```


## 5.4 Setup JMS Queue

```console
(mq:9.1.2.0)mqm@mqprod-ibm-mq-0:/mnt/mqm/data$ tee inst.scp <<EOF
define ctx(qmgr)
change ctx(qmgr)
def qcf(qcf) qmgr(qmgr) tran(CLIENT) chan(SYSTEM.ADMIN.SVRCONN) host(172.16.1.1) port(31608)
def q(DEFAULTAGENTQUEUE) qu(DEFAULTAGENTQUEUE) qmgr(qmgr)
end
EOF

(mq:9.1.2.0)mqm@mqprod-ibm-mq-0:/mnt/mqm/data$ /opt/mqm/java/bin/JMSAdmin < inst.scp
```


## 5.5 Create OMS Agent


5.5.1.   You may already have OMS's Application Manager from earlier. Go back to the Application Manager. Click on Application - Application Platform. Type the password `password` if prompted. 

5.5.2.   From the left-hand pane, double-click on Process Modeling. 

5.5.3.   From the right-hand pane, double-click on Order Fulfillment. 

5.5.4.   From the left-hand pane, look at the bottom and hoover your mouse over to the second tab called Transactions. While still in the left-hand pane, scroll down, find and double-click Schedule Order.

5.5.5.   From the right-hand pane, click on Time-Triggered tab.

5.5.6.   Under Agent Criteria Definition, click on the green "+" icon.

5.5.7.   From the Runtime Properties - Agent Server, click on the green "+" icon. This will launch a new window. From the new window's Server Properties tab, write `server` in the Server Name field. Click the Save icon to save.

5.5.8.   Back to the previous window, drop-down the Alert Queue Name, select DEFAULT(DEFAULT).

5.5.9.   Replace the `DefaultQueueManager` from the JMS Queue Manager field to `DEFAULTQUEUEMANAGER`.

5.5.10.  Drop-down the Initial Context Factory field and select `File`.

5.5.11.  Type `qcf` in the Connection Factory field.

5.5.12.  Type `file:///opt/ssfs`.

5.5.13.  At the top of the window, type AgentTrigger as its Criteria ID name. Click the Save icon to save.


## 5.6 Create OMS binding Configmap


```console
kubectl create configmap omsprod-binding --from-file=/mnt/data/mq/data/jndi/qmgr/.bindings -n default
```

# Exercise 6 - Upgrade OMS

## 6.1 Edit values.yaml


```YAML
  customerOverrides:
  - yfs.oms_provider_url=file:///opt/ssfs
  - yfs.oms_qcf=qcf
  - yfs.yfs.agent.override.icf=com.sun.jndi.fscontext.RefFSContextFactory
  - yfs.yfs.agent.override.providerurl=file:///opt/ssfs
  - yfs.yfs.agent.override.qcf=qcf
  - yfs.yfs.flow.override.icf=com.sun.jndi.fscontext.RefFSContextFactory
  - yfs.yfs.flow.override.providerurl=file:///opt/ssfs
  - yfs.yfs.flow.override.qcf=qcf
  - yfs.sci.queuebasedsecurity.userid=mqm
  - yfs.sci.queuebasedsecurity.password=mqm
  envs: []
```

On line 37, provide the name `omsprod-binding` as a value.

```YAML
  mq:
    bindingConfigName: omsprod-binding
    bindingMountPath: /opt/ssfs/.bindings
```

On line 138, provide the name of `server1` delete the additional blank line.

```YAML
  servers:
  - group: Default Server
    name:
    - server1
```

On line 141, type `donotinstall` as a value for the key `loadFactoryData`

```YAML
datasetup:
  loadFactoryData: donotinstall
  mode: create
```

Ctrl+s to save the file.

## 6.2 Helm Upgrade


```console
C:\Users\demo\Documents\ibm-oms-ent-prod> helm upgrade omsprod -f values.yaml . --tls
```


## 6.3 Watch and Monitor the Logs

```cmd
C:\Users\demo\Documents\ibm-oms-ent-prod> kubectl get pods
```

```console
C:\Users\demo\Documents\ibm-oms-ent-prod> kubectl get pods
NAME                                                      READY   STATUS    RESTARTS   AGE
mqprod-ibm-mq-0                                           1/1     Running   0          4h18m
omsprod-ibm-oms-ent-prod-appserver-bdc7b59d8-vhcpp        1/1     Running   0          4m29s
omsprod-ibm-oms-ent-prod-healthmonitor-78c56dcfc8-l9lrf   1/1     Running   0          4m29s
omsprod-ibm-oms-ent-prod-server1-8c85f4cdb-m2jmr          1/1     Running   0          4m29s
```


## 6.4 Scale pods down


6.4.1 Go to ICP dashboard (https://172.16.1.1:8443) and log in with admin/admin.

6.4.2 From the Hamburger menu on the left, go to Workloads - Deployments.

6.4.3 First, hoover your mouse over the `health-monitor` pod and click the three ... located at the right side of the screen. Select Scale.

6.4.4 Decrease the number 1 to 0. Click Scale Deployment. Repeat the same for `server1` and `appserver` pods.


## 6.5 Fix logs

With all three pods scaled down to 0. Launch DBVisualizer from the Desktop.

Expand OMSServer. Expand OMDB. Expand TABLE.

Scroll-down and double click on YFS_HEARTBEAT. Click OK. Note the populated data.

Click the Data tab. From the menus at the top, click SQL Commander. This opens up a new tab to run SQL commands.

Type the following SQL command and press the green run button to execute.

```sql
DELETE FROM YFS_HEARTBEAT;
```

Click the Data tab to note all the populated data has been deleted. Close the DB Visualizer.

## 6.6 Scale pods up

6.6.1 Go back to the ICP dashboard - Workloads - Deployment.

6.6.2 Click the three ... from the `appserver` pod. Select Scale.

6.4.4 Increase the number 0 to 1. Click Scale Deployment. Repeat the same for `health-monitor` and `server1` pods.

6.4.5 Wait a minute or two to ensure all pods are in 1/1 Ready state.

## 6.7 Verify Agent Server logs for MQ connection

6.7.1 List the pods.

```cmd
C:\Users\demo\Documents\ibm-oms-ent-prod> kubectl get pods
NAME                                                      READY   STATUS    RESTARTS   AGE
mqprod-ibm-mq-0                                           1/1     Running   0          4h40m
omsprod-ibm-oms-ent-prod-appserver-bdc7b59d8-df2jk        1/1     Running   0          2m56s
omsprod-ibm-oms-ent-prod-healthmonitor-78c56dcfc8-2626s   1/1     Running   0          2m49s
omsprod-ibm-oms-ent-prod-server1-8c85f4cdb-jdc8h          1/1     Running   0          2m41s
```

6.7.2 List the logs of the `server` pod.

```cmd
C:\Users\demo\Documents\ibm-oms-ent-prod>kubectl logs -f omsprod-ibm-oms-ent-prod-server1-8c85f4cdb-jdc8h
evaluating env variables from file /opt/ssfs/system_overrides.properties.init into /opt/ssfs/runtime/properties/system_overrides.properties
done!
JDK_VERSION: 1.8.0_191
JDK_VENDOR: IBM Corporation
JDK_VM_NAME: IBM J9 VM
+ /opt/ssfs/runtime/bin/java_wrapper.sh -Xms512m -Xmx1024m -Djava.io.tmpdir=/opt/ssfs/runtime/tmp -Dfile.encoding=UTF-8 -Dnet.sf.ehcache.skipUpdateCheck=true -DINSTALL_DIR=/opt/ssfs/runtime -DINSTALL_LOCALE=en -XX:MetaspaceSize=1024m -Xms512m -Xmx1024m -Dserver.state.dir=/opt/ssfs/runtime/serverstate -classpath /opt/ssfs/runtime/jar/bootstrapper.jar -Dvendor=shell -DvendorFile=/opt/ssfs/runtime/properties/servers.properties -DCACHE_PS=true -DDISABLE_DS_EXTENSIONS=Y com.sterlingcommerce.woodstock.noapp.NoAppLoader -class com.yantra.integration.adapter.IntegrationAdapter -f /opt/ssfs/runtime/properties/AGENTDynamicclasspath.cfg -invokeargs server1
2019-05-14 21:35:24,984:INFO   :main: Initializing data cache...                                   [system]: [ ]: YCPSystem
2019-05-14 21:35:33,727:INFO   :main: PricingRule and PriceListHeader Cache Loading has been skipped [system]: [ ]: YPMExtensionPointInitializer
2019-05-14 21:35:38,651:INFO   :main: Configuring Error Context                                    [system]: [ ]: ErrorContext
2019-05-14 21:35:39,112:INFO   :main: Server server1_0 registered.                                 [system]: [ ]: YFCRemoteManager
2019-05-14 21:35:39,118:INFO   :main: Initializing ehcache...                                      [system]: [ ]: SCExternalCacheSetup
2019-05-14 21:35:39,136:WARN   :main: Catalog Search Index not enabled                             [system]: [ ]: YCMPostEhcacheInitEP
2019-05-14 21:35:39,195:INFO   :main: server state file /opt/ssfs/runtime/serverstate/server1--server1_0 created. [system]: [ ]: ServerStateExporter

2019-05-14 21:35:39,203:INFO   :main: Starting Service[ 0] For RuntimeId: AgentTrigger             [system]: [ ]: ServiceContextMediatorBase
2019-05-14 21:35:39,204:INFO   :main: Successfully started all services for Server: server1        [system]: [ ]: IntegrationAdapter
2019-05-14 21:35:39,926:INFO   :pool-1-thread-1: Info:JMX MBeans loaded                                       [system]: [ ]: PLTJMXUtils
```



