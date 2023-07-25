Installation Guide (ELI5 version)
The operating system used is Ubuntu 22.04. The installation should be performed from the root account.

```bash
sudo -i
```

```bash
sudo apt-get update && sudo apt-get upgrade
```

To create a wallet, first download and run geth. (Only used for wallet creation purposes)
```bash
wget https://gethstore.blob.core.windows.net/builds/geth-linux-amd64-1.11.5-a38f4108.tar.gz

tar -xvf geth-linux-amd64-1.11.5-a38f4108.tar.gz

cd geth-linux-amd64-1.11.5-a38f4108

sudo mv geth /usr/local/bin/

geth account new
```
Install git and open .env to edit its content:

```bash
sudo apt install git

git clone https://github.com/JSY3456/ocash-mining-pool.git

cd ocash-mining-pool

cp .env.example .env && cp config/config.example.json config/config.json

sudo nano .env
```

```bash

POOL_ADDRESS="<pool ocash address>"

KEYSTORE_DIR_PATH="/root/.ethereum/keystore/"

KEYSTORE_PASSWORD_FILE_PATH="/root/.ethereum/.pw"

POOL_TLS_CERT_PATH="/var/lib/certs/cert.pfx"
```

Install certbot. If you don't have a domain, skip this step. Replace /<your.domain> with your domain.

```bash

sudo apt install certbot

sudo certbot certonly --standalone --preferred-challenges http -d <your.domain>

sudo mkdir -p /var/lib/certs/

sudo openssl pkcs12 -export -out /var/lib/certs/cert.pfx -inkey /etc/letsencrypt/live/<your.domain>/privkey.pem -in /etc/letsencrypt/live/<your.domain>/fullchain.pem

```
Enter the password.

*If you can't find the path of the file, run ' sudo certbot certificates' to check the location.

```bash
 sudo nano ~/ocash-mining-pool/config/config.json
```

```bash

"pools": [
      {
        "id": "eth1",
        "enabled": true,
        "coin": "ethereum",
        "address": "<pool ocash address>",
        "rewardRecipients": [
          {
            "type": "op",
            "address": "<rewards ocash address>",
            "percentage": 1.0

```

If you have installed certbot and connected it to the domain, change the settings as below. The password is the one you created when you made cert.pfx.

```bash

"ports": {
          "4073": {
            "name": "GPU-SMALL",
            "listenAddress": "*",
            "tls": true,
            "tlsPfxFile": "/var/lib/certs/cert.pfx",
            "tlsPfxPassword": "<password>",
            "difficulty": 0.1,
            "varDiff": {
              "minDiff": 0.1,
              "maxDiff": null,
              "targetTime": 15,
              "retargetTime": 90,
              "variancePercent": 30

```
Create a wallet key file. If you want to change the path or filename of the wallet key file, modify the content of 'docker-compose.yml'

```bash

cd ~/.ethereum/

sudo nano .pw
```

Open the ports:
```bash
sudo apt install ufw
#remote access
sudo ufw allow ssh
#api
sudo ufw allow 4000
#node p2p
sudo ufw allow 30303
#mining
sudo ufw allow 4073
#Website
sudo ufw allow 80

sudo ufw enable

```



Run docker and wait.

```bash

cd ~/ocash-mining-pool/
```

```bash
sudo apt install docker-compose
```
```bash
sudo docker-compose up
```

*I will upload two methods for front-end settings when I have time. If you are in a hurry, please refer to the front-end installation method in this link:

https://medium.com/@uanid/how-to-operate-a-mining-pool-in-a-windows-environment-using-a-virtual-machine-vm-ocash-ver1-0-6f54236602ae 



# ōCash Mining Pool

> **BEWARE: You should understand how this pool setup works! It involves manipulation with
> your pool's private key and password to it so it is crucial to configure it in a secure way.
> Default setup provides a secure configuration but if you change things, you can end up with
> your ōCash node being open to the internet and accepting transactions from you pool's address**

This template is for setting up a simple ōCash mining pool. It uses a docker-compose to run all
needed components, stores all persistent data in docker volumes (ōCash chain, pool database, DAGs...),
so containers can be removed, changed, rebuilt and rerun and until you delete these volumes, no data
is lost.

## Requirements

- linux machine
- docker CE, with docker compose plugin / standalone docker-compose - for installing these, refer
to [official Docker documentation](https://docs.docker.com/engine/install/)
- recent git
- enough disk space for storage of chain data and pool software database
- machine has to have port `30303` open for ōCash node P2P to works, for accessing the pools REST api
from internet also port `4000` needs to be open (port numbers can be changed, these are for default setup)

### Components

- ōCash node
- miningcore pool
- postgresql database

## Running

- prepare your Geth keystore, keep it in a secure space
- clone the pool git repository (`git clone https://github.com/blockcollider/ocash-mining-pool.git`)
- move to the repository clone (`cd ocash-mining-pool`)
- copy template configuration files to actual ones (`cp .env.example .env && cp config/config.example.json config/config.json`)
- fill in the values in `.env` to match your setup
- fill in the needed values in `config/config.json`
  - `pools.[0].address` is the only needed one for basic setup, must match `POOL_ADDRESS` in `.env`
- put your keystore in the directory you filled in `.env` file
- put your keystore password in the file you filled in `.env` file
- double check permissions on these two
- run the containers using `docker compose up`, refer to [Docker Compose documentation](https://docs.docker.com/compose/) for usage

## TLS configuration

It is strongly recommended to run the pool only with TLS encryption (pool will
serve `stratum2+ssl://` endpoint in that case).

### Testing setup

This setup will allow you to test the TLS setup, although it creates a testing
CA which is not trusted by clients by default (it's root key is not in common
CA stores)

For simpliest setup with self-signed certificate, you can use for example
[`mkcert` utility](https://github.com/FiloSottile/mkcert) (refer to you host
system package manager how to install this tool, e.g. on Debian it would be
`apt install mkcert`) for generating the certificate. Mind that miningcore
needs `pkcs12` key format, so the steps would be:

- *install the mkcert utility*
- `mkcert -install`
- `mkcert -cert-file pool-cert.pem -key-file pool-cert.key <POOL_PUBLIC_IP_ADDRESS>`
- `openssl pkcs12 -export -out pool-cert-self-signed.pfx -inkey pool-cert.key -in pool-cert.pem` (choose a password or leave blank for no password)
- put the path from previous step (option `-out`) to you `.env` file to key `POOL_TLS_CERT_PATH`
- change `pool[0].ports[0].tls` to `true`
- change `pool[0].ports[0].tlsPfxPassword` to the certifcate password

Mind that this setup is solely for testing purposes - you have to advise your clients to ignore certificate validation (e.g. `SSL_NOVERIFY=1` when using `ethminer`)

### Production setup

Simpliest way for using proper valid certificate is having a domain you own pointed to the server IP running the pool setup. Then you can use
Let's Encrypt for generating your keys, the common tool for this is [certbot](https://certbot.eff.org/).

Steps (on Debian 10):

- install certbot (`apt install certbot`)
- generate the keys with `certbot certonly --manual --preferred-challenges dns --email <your email> --domains <your domain>`
- as instructed, create the required TXT DNS record for you domain to pass the validation
- convert the key to PKCS12 format using `openssl pkcs12 -export -out pool-cert.pfx -inkey /etc/letsencrypt/live/<your domain>/privkey.pem -in /etc/letsencrypt/live/<your domain>/fullchain.pem`
the PKCS12 key is in file `pool-cert.pfx`.
- mind that the conversion step needs to be done after each key renewal, so it's better to use [certbot hooks](https://eff-certbot.readthedocs.io/en/stable/using.html#pre-and-post-validation-hooks) to this for you
- put path to converted `pool-cert.pfx` to your `.env`s `POOL_TLS_CERT_PATH`
- change `pool[0].ports[0].tls` to `true`
- change `pool[0].ports[0].tlsPfxPassword` to the certifcate password


## TODO

- [ ] Pool web UI
