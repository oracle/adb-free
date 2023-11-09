# Oracle Autonomous Database Free Container Image Documentation

Oracle Autonomous Database Free Container image comes with 2 prebuilt Databases - `MY_ATP` and `MY_ADW`. These are similar to Transaction Processing and Data Warehouse workload type databases in Autonomous Database Serverless Cloud service.

Following key features are supported:

- Oracle Rest Data Services (ORDS)
- APEX
- Database Actions
- Mongo API enabled (by default routed to `MY_ATP`)

The storage size is limited to 20 GB for each Database

## Using this image

### Container CPU/memory requirements

Oracle Autonomous Database Free container needs 4 CPUs and 8 GiB memory

### Starting an ADB Free container

To start an Oracle Autonomous Database Free container run the following command 

```bash
podman run -d \
-p 1521:1522 \
-p 1522:1522 \
-p 8443:8443 \
-p 27017:27017 \
-e MY_ADB_WALLET_PASSWORD=*** \
-e MY_ADW_ADMIN_PASSWORD=*** \
-e MY_ATP_ADMIN_PASSWORD=*** \
--cap-add SYS_ADMIN \
--device /dev/fuse \
--name adb-free \
ghcr.io/oracle/adb-free:latest
```
On first startup of the container:

- User mandatorily has to change the admin passwords. Please specify them using the environment variables `MY_ADW_ADMIN_PASSWORD` and `MY_ATP_ADMIN_PASSWORD`

- Wallet is generated using the wallet password `MY_ADB_WALLET_PASSWORD`


#### Note:
- For OFS mount, container should start with `SYS_ADMIN` capability. Also, virtual device `/dev/fuse` should be accessible
- `--hostname` is the Fully Qualified Domain Name (FQDN) of your host.

Note the following ports which are forwarded to the container process

| Port | Description                                     |
|------|-------------------------------------------------|
| 1521 | TLS                                             |
| 1522 | mTLS                                            |
| 8443 | HTTPS port for ORDS / APEX and Database Actions |
| 27017 | Mongo API ( MY_ATP )                            |

If you are behind a corporate proxy, pass the proxy environment variables as shown below:

```bash
podman run -d \
-p 1521:1522 \
-p 1522:1522 \
-p 8443:8443 \
-p 27017:27017 \
-e MY_ADB_WALLET_PASSWORD=*** \
-e MY_ADW_ADMIN_PASSWORD=*** \
-e MY_ATP_ADMIN_PASSWORD=*** \
-e http_proxy=http://my-corp-proxy.com:80/ \
-e https_proxy=http://my-corp-proxy.com:80/ \
-e no_proxy=localhost,127.0.0.1 \
-e HTTP_PROXY=http://my-corp-proxy.com:80/  \
-e HTTPS_PROXY=http://my-corp-proxy.com:80/  \
-e NO_PROXY=localhost,127.0.0.1 \
--cap-add SYS_ADMIN \
--device /dev/fuse \
--name adb-free \
ghcr.io/oracle/adb-free:latest
```
### Migrating data across containers

#### Mount Volume
To persist data across container restarts and removals, you should mount a volume at `/u01/data` and follow the steps mentioned in the [documentation to migrate PDB data across containers](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/autonomous-docker-container.html#GUID-03B5601E-E15B-4ECC-9929-D06ACF576857) 

```bash
podman run -d \
-p 1521:1522 \
-p 1522:1522 \
-p 8443:8443 \
-p 27017:27017 \
-e MY_ADB_WALLET_PASSWORD=*** \
-e MY_ADW_ADMIN_PASSWORD=*** \
-e MY_ATP_ADMIN_PASSWORD=*** \
--cap-add SYS_ADMIN \
--device /dev/fuse \
--name adb-free \
--volume adb_container_volume:/u01/data \
ghcr.io/oracle/adb-free:latest
```

### Connecting to Oracle Autonomous Database Free container


#### ORDS/APEX/Database Actions

During container initialization `--hostname` argument is used to generate self-signed SSL certs to serve HTTPS traffic on port 8443.

For example, if you run the ADB free container on a host whose FQDN is `my.host.com` you can pass `--hostname my.host.com`

APEX and Database Actions can be accessed using the passed hostname (or simply localhost)


| Application      | MY_ATP                                          | MY_ADW                                                              | Example                                            |
|------------------|-------------------------------------------------|---------------------------------------------------------------------|----------------------------------------------------|
| APEX             | https://localhost:8443/ords/my_atp/             | https://localhost:8443/ords/my_adw/                                 | https://my.host.com:8443/ords/my_atp               |
| Database Actions | https://localhost:8443/ords/my_atp/sql-developer | https://localhost:8443/ords/my_adw/sql-developer | https://my.host.com:8443/ords/my_adw/sql-developer |

#### Wallet Setup

In the container, TLS wallet is generated at location `/u01/app/oracle/wallets/tls_wallet`

Copy wallet to your host. 

```bash
podman cp adb_container:/u01/app/oracle/wallets/tls_wallet /scratch/tls_wallet
```
In this example, wallet is copied to `/scratch/tls_wallet` folder

Point `TNS_ADMIN` environment variable to the wallet directory
```bash
export TNS_ADMIN=/scratch/tls_wallet
```

If you want to connect to a remote host where the ADB free container is running, replace `localhost` in `$TNS_ADMIN/tnsnames.ora` with the remote host FQDN

```bash
sed -i 's/localhost/my.host.com/g' $TNS_ADMIN/tnsnames.ora
```

#### Available TNS aliases

Similar to Autonomous Database Serverless Cloud service, use any one of the following aliases to connect to ADB free container.

##### MY_ATP TNS aliases

For mTLS use the following 
- my_atp_medium
- my_atp_high
- my_atp_low
- my_atp_tp
- my_atp_tpurgent

For TLS use the following

- my_atp_medium_tls
- my_atp_high_tls
- my_atp_low_tls
- my_atp_tp_tls
- my_atp_tpurgent_tls

##### MY_ADW TNS aliases

For mTLS use the following 
- my_adw_medium
- my_adw_high
- my_adw_low

For TLS use the following
- my_adw_medium_tls
- my_adw_high_tls
- my_adw_low_tls

TNS alias mappings to their connect string can be found in` $TNS_ADMIN/tnsnames.ora` file.

#### TLS walletless connection

To connect without a wallet, you need to update your client's truststore with the self-signed certificate generated at container start

##### Linux system truststore

Copy `/u01/app/oracle/wallets/tls_wallet/adb_container.cert` from container and update your system truststore

```bash
podman cp adb-free:/u01/app/oracle/wallets/tls_wallet/adb_container.cert adb_container.cert
sudo cp adb_container.cert /etc/pki/ca-trust/source/anchors
sudo update-ca-trust
```

##### MacOS system trustsore

For MacOS, please refer the [support guide](https://support.apple.com/guide/keychain-access/add-certificates-to-a-keychain-kyca2431/mac) to add certificate to keychain

##### JDK truststore

For JDK truststore update, you can use `keytool`

Linux example:

```bash
sudo keytool -import -alias adb_container_certificate -keystore $JAVA_HOME/lib/security/cacerts -file adb_container.cert 
```

MacOS example:
```bash
sudo keytool -import -alias adb_container_certificate -file adb_container.cert -keystore  /Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home/lib/security/cacerts
```


#### SQL*Plus

In this example, we connect using the alias `my_atp_low`
```text
sqlplus admin/<my_atp_admin_password>@my_atp_low

SQL*Plus: Release 21.0.0.0.0 - Production on Wed Jul 26 22:38:27 2023
Version 21.9.0.0.0

Copyright (c) 1982, 2022, Oracle.  All rights reserved.

Last Successful login time: Wed Jul 26 2023 16:36:16 +00:00

Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.20.0.1.0

SQL> 
```

#### Python

Install the python-oracledb driver for Oracle Database
```bash
python3 -m pip install oracledb
```

```python
import oracledb
conn = oracledb.connect(user="admin", password="<my_adw_admin_password>", dsn="my_adw_medium", config_dir="/scratch/tls_wallet", wallet_location="/scratch/tls_wallet", wallet_password="***")
cr = conn.cursor()
r = cr.execute("SELECT 1 FROM DUAL")
print(r.fetchall())

>> [(1,)]

```


### Create an app user

Connect as Admin
```bash
sqlplus admin/<my_atp_admin_password>@my_atp_medium
```
Create user as shown below:
```sql
CREATE USER APP_USER IDENTIFIED BY "<my_app_user_password>" QUOTA UNLIMITED ON DATA;
 
-- ADD ROLES
GRANT CONNECT TO APP_USER;
GRANT CONSOLE_DEVELOPER TO APP_USER;
GRANT DWROLE TO APP_USER;
GRANT RESOURCE TO APP_USER;  
 
 
-- ENABLE REST
BEGIN
    ORDS.ENABLE_SCHEMA(
        p_enabled => TRUE,
        p_schema => 'APP_USER',
        p_url_mapping_type => 'BASE_PATH',
        p_url_mapping_pattern => 'app_user',
        p_auto_rest_auth=> TRUE
    );
    commit;
END;
/
 
-- QUOTA
ALTER USER APP_USER QUOTA UNLIMITED ON DATA;

```

## F.A.Q

### How can I run Oracle Autonomous Database Free container on ARM64 arch i.e. machines with M1/M2 chips ?
Use colima + docker to emulate x86_64 arch. Replace podman with docker in all commands. This is only until we have a native ARM 64 image.

### How can I install colima and docker on machines with M1/M2 chips ?
```bash
brew install docker
brew install docker-compose
brew install colima
brew reinstall qemu
```

### How can I start Colima x86_64 Virtual Machine with minimum memory/cpu requirements ?
```bash
colima start --cpu 4 --memory 8 --arch x86_64
```

### How can I start podman VM on x86_64 Mac with minimum memory/cpu requirements ?
```bash
podman machine init
podman machine set --cpus 4 --memory 8192
podman machine start
```
