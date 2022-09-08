# Setup

## Basic

```shell
apt update -y
apt upgrade -y
apt install -y vim curl wget
```

## Create User

```shell
adduser <username>
usermod -aG sudo,admin <username>
```

## SSH-Configuration

Insert your `ssh-public-keys` in ~/.ssh/authorized_keys.

Then change your `ssh`-Port to 2222:

```shell
sudo vim /etc/ssh/sshd_config
# Search for the line with `Port 22`
# Uncomment it and change it to `Port 2222`
service ssh restart
```

Now you have to relog.

## Install Docker and Docker-Compose

Execute the `docker`-Install Script:

```shell
curl -fsSL https://get.docker.com | sudo bash
```

After this installation you will see a comment which suggests you to install
`docker-rootless` but I do not recommend to do this.

Now you have to install `docker-compose`:

```shell
sudo curl -L -o /usr/local/bin/docker-compose \
  "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)"
sudo chmod +x /usr/local/bin/docker-compose
```

Then create some aliases:

```shell
cat <<_EOF >> .bashrc
alias dc='sudo docker-compose'
alias docker='sudo docker'
alias dcr='sudo docker-compose down && sudo docker-compose up -d'
_EOF
```

Last but not least create a network for `docker`:

```shell
sudo docker network create --subnet 172.20.255.0/24 database
sudo docker network create --subnet 172.30.255.0/24 matrix
```