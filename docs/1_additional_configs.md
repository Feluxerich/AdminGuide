# Additional Configs (Optional)

## Mount Samba File Storage

!!! note "" Here: `Hetzner Storage Box`

### Install Dependencies

You have to install the `cifs-utils`-Package.

```shell
sudo apt install cifs-utils
```

### Create Storage Destination Folder

Now you create the folder where your files and your data should be saved to.

```shell
sudo mkdir /media/share
# OR
sudo mkdir /srv/minio
# To use it directly as storage for MinIO
```

### Set Credentials

The credentials used to connect to the samba file share have to be written in to a file.

```shell
sudo cat <<_EOF >> /root/.sbcredentials
username=STORAGE-BOX-USERNAME
password=STORAGE-BOX-PASSWORD
_EOF

sudo chmod 400 /root/.sbcredentials
```

### Mount

```shell
sudo mount -t cifs -o rw,vers=3.0,credentials=/root/.examplecredentials //username.your-storagebox.de/(backup|sub-account-name) (/media/share|/srv/minio)
```

If there are no errors you may add this line to your `/etc/fstab`:

```shell
//username.your-storagebox.de/(backup|sub-account-name) (/media/share|/srv/minio) cifs vers=3.0,credentials=/root/.examplecredentials
```

And finally you're done.
