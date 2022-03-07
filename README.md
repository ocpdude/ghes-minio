## Installing & Configuring MinIO with GitHub Enterprise Server
---

Demo YouTube video can be found [here](https://youtu.be/zJcrVjvyxaI) \
Find the documentation for MinIO [here](https://docs.min.io)

### GitHub Enterprise server supports 3 storage platforms _only_, Amazon Buckets, Azure Blobs and MinIO.
For an on-prem lab installation, I've chosen MinIO to provide S3 backend storage for GHES Actions and Packages. Follow along as we set this up.

Prereq's
1. DNS entry `minio.redcloud.land`
2. Set GHES to 64Gb of memory to support 'Actions`
3. Install VM to host minio, with a secondary drive for storage provisioning.
    ##### Note: The formatted filesystem could be either ext4 or xfs


Install (I used Fedora 35 Server)
1. Patch system
    * dnf update -y
2. Partition the second drive (if not done already)
    * fdisk /dev/sdb
    * mkfs.xfs /dev/sdb1
    * mkdir /datastore
    * vi /etc/fstab \
        add storage entry: \
        /dev/sdb1   /datastore   xfs   defaults  0 0
    * mount /datastore
3. Add the minio user
    * useradd minio -s /bin/false
4. Add minio permissions to /datastore 
    * chow minio:minio /datastore
5. Create and upload the TLS certificates for redcloud.land
    * mkdir -p /etc/minio/certs/CAs
    * scp minio.*.crt(key) minio:/tmp
    * scp ca.crt minio/tmp
    * mv /tmp/minio.*.crt /etc/minio/certs/public.crt
    * mv /tmp/minio.*.key /etc/minio/certs/private.key
    * mv /tmp/ca.crt /etc/minio/certs/CAs/ca.crt
6. Download the minio binary (106MB)
    * wget https://dl.min.io/server/minio/release/linux-amd64/minio
    * chmod -x minio
    * mv minio /usr/local/bin/
7. Set the firewall rules (Fedora)
    * firewall-cmd --add-port=443/tcp --add-port=9000/tcp --permanent
    * firewall-cmd --reload

Since I want to use systemd to manage the service, we'll configure that. You can also just run `minio server... ` to start the service manually. The official minio GitHub pages provide more details to setting up systemd. My edit include the options for my specific configuration.  https://github.com/minio/minio-service/tree/master/linux-systemd

8. Create an environment file
    * vi /etc/default/minio
    ```
    # Volume to be used for MinIO server.
    MINIO_VOLUMES="/datastore/disk{1...6}"
    # Use if you want to run MinIO on a custom port.
    MINIO_OPTS="--address minio.redcloud.land:443 --console-address :9000 --certs-dir /etc/minio/certs/"
    # Root user for the server.
    MINIO_ROOT_USER=minioadmin
    # Root secret for the server.
    MINIO_ROOT_PASSWORD=minioadmin
    ```
9. Create the systemd service
    * vi /etc/systemd/system/minio.service
    ```
    [Unit]
    Description=MinIO
    Documentation=https://docs.min.io
    Wants=network-online.target
    After=network-online.target
    AssertFileIsExecutable=/usr/local/bin/minio

    [Service]
    AmbientCapabilities=CAP_NET_BIND_SERVICE
    WorkingDirectory=/usr/local/

    User=minio
    Group=minio

    EnvironmentFile=/etc/default/minio
    ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
    ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

    # Let systemd restart this service always
    Restart=always

    # Specifies the maximum file descriptor number that can be opened by this process
    LimitNOFILE=1048576

    # Specifies the maximum number of threads this process can create
    TasksMax=infinity

    # Disable timeout logic and wait until process is stopped
    TimeoutStopSec=infinity
    SendSIGKILL=no

    [Install]
    WantedBy=multi-user.target

    # Built for ${project.name}-${project.version} (${project.name})

    ```
10. Allow non root users to start a service with a port below 1024
    * setcap cap_net_bind_service=+ep /usr/local/bin/minio
11. Enable our new systemd service so it'll start on boot
    * systemctl enable minio
12. Disable SELINUX (Fedora)
    * setenforce 0
13. Start the service...
    * systemctl start minio

#### NOTE: If you get an error while starting, make sure your user permissions are set properly on your mounted volume (/datastore)

