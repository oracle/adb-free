# Oracle Autonomous Database Free Container Image Documentation

Oracle Autonomous Database Free Container image supports 2 types of database workload types; `ADW` and `ATP`. These are similar to Transaction Processing and Data Warehouse workload type databases in Autonomous Database Serverless Cloud service.

Following key features are supported:

- Oracle Rest Data Services (ORDS)
- APEX
- Database Actions
- Mongo API
- Oracle Estate Explorer (OEE)

The storage size is limited to 20 GB for each Database

## Using this image

### Database versions

From the [released images](https://github.com/oracle/adb-free/pkgs/container/adb-free), choose the database version and corresponding image to work with.

We use the following naming convention:

| Database version | Latest image tag | Specific release image tag |
|------------------|------------------|----------------------------|
| 23ai | latest-23ai      | 24.11.4.2-23ai              |
| 19c | latest           | 24.12.1.2                   |

### Container CPU/memory requirements

Oracle Autonomous Database Free container needs 4 CPUs and 8 GiB memory

### Install podman

Please refer the official documentation to install podman on [Linux](https://podman.io/docs/installation#installing-on-linux), [Windows](https://podman.io/docs/installation#windows) or [Mac](https://podman.io/docs/installation#macos)


### Start podman machine on MacOS (x86_64) or Windows (x86_64)

Containers need the Linux kernel. Run following commands to start a podman virtual machine

```bash
podman machine init
podman machine set --cpus 4 --memory 8192
podman machine start
```

Refer the [FAQ](#faq) to configure virtual machine on ARM machine (M1/M2 chips)

### Starting an ADB Free container

> Note: Although the instructions use `podman`, the image format is compliant with both Open Container Initiative (OCI) and Docker.
> ADB container works seamlessly with both OCI and Docker container runtimes. You can also use `docker` to start the container.

To start an Oracle Autonomous Database Free container for **ATP** workload, run the following command

```bash
podman run -d \
-p 1521:1522 \
-p 1522:1522 \
-p 8443:8443 \
-p 27017:27017 \
-e WORKLOAD_TYPE=ATP \
-e WALLET_PASSWORD=*** \
-e ADMIN_PASSWORD=*** \
--cap-add SYS_ADMIN \
--device /dev/fuse \
--name adb-free \
ghcr.io/oracle/adb-free:latest-23ai
```

> Note: Use `ghcr.io/oracle/adb-free:latest` for 19c

#### On first startup of the container:

- User mandatorily has to change the admin passwords. Please specify the password using the environment variable
`ADMIN_PASSWORD`

- Wallet is generated using the wallet password `WALLET_PASSWORD`


Following table explains the environment variables passed to the container

| Environment variable | Description                                                                                                                                                                               |
|----------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| WORKLOAD_TYPE       | Can be either `ATP` or `ADW`. Default value is `ATP`                                                                                                                                      |
| DATABASE_NAME       | Database name should contain only alphanumeric characters. if not provided, the Database will be called either `MYATP` or `MYADW` depending on the passed workload type                   |
| ADMIN_PASSWORD       | Admin user password must be between 12 and 30 characters long and must include at least one uppercase letter, one lowercase letter, and one numeric. The password cannot contain username |
| WALLET_PASSWORD      | Wallet password must have a minimum length of eight characters and contain alphabetic characters combined with numbers or special characters.                                             |
| ENABLE_ARCHIVE_LOG   | To enable archive logging in the database. Default value is True. To turn off archive logging set the value to False                                                                                                                         |


> **_Note_**: For OFS mount, container should start with `SYS_ADMIN` capability. Also, virtual device `/dev/fuse` should be accessible

#### Ports

Note the following ports which are forwarded to the container process

| Port | Description                          |
|------|--------------------------------------|
| 1521 | TLS                                  |
| 1522 | mTLS                                 |
| 8443 | HTTPS port for ORDS / APEX and Database Actions |
| 27017 | Mongo API                            |

#### HTTP proxy
If you are behind a corporate proxy, there are 2 ways to configure the database
to use the proxy settings

1. Set the `HTTP_PROXY` database property. This is used by packages like `DBMS_CLOUD`

```sql
exec DBMS_CLOUD_CONTAINER_ADMIN.set_database_property('HTTP_PROXY', 'www-my-corp-proxy.com:80/');
```

2. Use `UTL_HTTP.set_proxy` to set proxy for HTTP requests sent using UTL_HTTP

```sql
exec UTL_HTTP.SET_PROXY('www-my-corp-proxy.com');
```


### adb-cli

`adb-cli` can be used to perform database operations after container is up and running

To use adb-cli, you can define the following alias for convenience
```bash
alias adb-cli="podman exec <container_name> adb-cli"
```

#### Available commands

```bash
>> adb-cli --help

Usage: adb-cli [OPTIONS] COMMAND [ARGS]...

  ADB-S Command Line Interface (CLI) to perform container-runtime database
  operations

Options:
  -v, --version  Show the version and exit.
  --help         Show this message and exit.

Commands:
  add-database
  change-password
```

#### Add Database

You can add a database using the `add-database` command

```bash
adb-cli add-database --workload-type "ADW" --admin-password "Welcome_1234"
```

#### Change Password

To change password for Admin user, use the following command

```bash
adb-cli change-password --database-name "MYADW" --old-password "Welcome_1234" --new-password "Welcome_12345"
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
-e WORKLOAD_TYPE=ATP \
-e WALLET_PASSWORD=*** \
-e ADMIN_PASSWORD=*** \
--cap-add SYS_ADMIN \
--device /dev/fuse \
--name adb-free \
--volume adb_container_volume:/u01/data \
ghcr.io/oracle/adb-free:latest-23ai
```


### Connecting to Oracle Autonomous Database Free container

#### ORDS/APEX/Database Actions

Container hostname is used to generate self-signed SSL certs to serve HTTPS traffic on port 8443. APEX and Database Actions can be accessed using the container host (or simply localhost)

| Application      | URL                                                                           |
|------------------|------------------------------------------------------------------------------|
| APEX             | https://localhost:8443/ords/apex                           |
| Database Actions | https://localhost:8443/ords/sql-developer  |

> **_Note:_**  For additional databases plugged in using `adb-cli add-database` command, please use the URL formats `https://localhost:8443/ords/{database_name}/apex` and `https://localhost:8443/ords/{database_name}/sql-developer` to access APEX and Database Actions respectively.

#### Wallet Setup

In the container, TLS wallet is generated at location `/u01/app/oracle/wallets/tls_wallet`

Copy wallet to your host.

```bash
podman cp adb-free:/u01/app/oracle/wallets/tls_wallet /scratch/tls_wallet
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

##### MYATP TNS aliases

For mTLS use the following
- myatp_medium
- myatp_high
- myatp_low
- myatp_tp
- myatp_tpurgent

For TLS use the following

- myatp_medium_tls
- myatp_high_tls
- myatp_low_tls
- myatp_tp_tls
- myatp_tpurgent_tls

##### MYADW TNS aliases

For mTLS use the following
- myadw_medium
- myadw_high
- myadw_low

For TLS use the following
- myadw_medium_tls
- myadw_high_tls
- myadw_low_tls

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

#### SQL Developer Desktop application

Copy the wallet from the container and zip it

```bash
podman cp adb-free:/u01/app/oracle/wallets/tls_wallet /scratch/tls_wallet

zip -j /scratch/tls_wallet.zip /scratch/tls_wallet/*

```

Once you zip the Wallet, open SQLDeveloper and follow the below steps:

1. Click on File -> New -> Database Connection

2. Enter username / password

3. From "Connection Type" dropdown choose "Cloud Wallet"

4. Under "Configuration file" browse path to your wallet.zip

5. Select Service from the dropdown

6. Click on Connect



#### SQL*Plus

In this example, we connect using the alias `myatp_low`
```text
sqlplus admin/<myatp_admin_password>@myatp_low

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
conn = oracledb.connect(user="admin", password="<myadw_admin_password>", dsn="myadw_medium", config_dir="/scratch/tls_wallet", wallet_location="/scratch/tls_wallet", wallet_password="***")
cr = conn.cursor()
r = cr.execute("SELECT 1 FROM DUAL")
print(r.fetchall())

>> [(1,)]

```


### Create an app user

Connect as Admin
```bash
sqlplus admin/<myatp_admin_password>@myatp_medium
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

### Oracle Estate Explorer (OEE)

[Oracle Estate Explorer](https://www.oracle.com/database/cloud-migration/estate-explorer/) is a tool that enables customers to
programmatically evaluate groups of Oracle databases for migration readiness to Oracle’s Autonomous Database (ADB). The output from OEE
provides a high-level estate overview of the tested group of databases, ranks them according to their alignment with ADB pre-requisites and
delivers a graded relative effort of any remediation required.

The OEE APEX app is installed in the adb-free images and is available to use out-of-the-box

Following steps are required to launch the OEE app:

1. Login as Database admin and set a password for the `MPACK_OEE` user

```sql
ALTER USER MPACK_OEE IDENTIFIED BY <PASSWORD>
```

2. Visit the APEX URL using https://localhost:8443/ords/apex

3. Login to the `MPACK_OEE` APEX workspace using the password set in Step 1

4. Change the MPACK_OEE’s APEX account password. A warning page will be displayed after changing the MPACK_OEE’s APEX account password, please ignore it and click on Application Builder to launch the OEE application.

5. On the application home page, click "Run Application" to open the OEE app in a new browser tab.

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

> Note: Running x86_64 arch containers can have issues translating instructions for ARM. We give higher memory to the VM to avoid such issues

```bash
colima start --cpu 4 --memory 10 --arch x86_64
```

### How can I start Colima x86_64 Virtual Machine using Apple's new virtualization framework - Rosetta ?

> Note: Running x86_64 arch containers can have issues translating instructions for ARM. We give higher memory to the VM to avoid such issues


```bash
softwareupdate --install-rosetta
colima stop
colima delete
colima start --cpu 4 --memory 10 --arch x86_64 --vm-type vz --vz-rosetta

# Verify if Colima is using the new profile
docker context ls
colima status
```

### How can I start podman VM on x86_64 Mac with minimum memory/cpu requirements ?
```bash
podman machine init
podman machine set --cpus 4 --memory 8192
podman machine start
```

## Contributing

This project welcomes contributions from the community. Before submitting a pull request, please [review our contribution guide](./CONTRIBUTING.md)

## Security

Please consult the [security guide](./SECURITY.md) for our responsible security vulnerability disclosure process

## License

Copyright (c) 2024 Oracle and/or its affiliates.

Released under the Universal Permissive License v1.0 as shown at
<https://oss.oracle.com/licenses/upl/>.
