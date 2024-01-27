# Systemd User PAM SSH script

This script has a rather specific use case. If you fit the following demographic
then this script might just be for you!

* You use systemd
* You login at the linux VT using a getty
* You have a `systemd --user` service called `ssh-agent.service` that starts
   your ssh agent.
* You have to type your passphrase after logging in in order to
   decrypt your SSH key.

This script allows you to only type your password once. When logging in, your
SSH key will be decrypted and added to your ssh-agent for you.

## Installation

(1) Set up your ssh-agent systemd user service with the proper
    environment using lingering to start it at boot

    # setup module
    echo '[Unit]
    Description=SSH key agent

    [Service]
    Type=simple
    Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
    ExecStart=/usr/bin/ssh-agent -D -a $SSH_AUTH_SOCK

    [Install]
    WantedBy=default.target' \
      > ~/.config/systemd/user/ssh-agent.service

    # setup environment
    echo 'SSH_AUTH_SOCK DEFAULT="${XDG_RUNTIME_DIR}/ssh-agent.socket"' \
      > ~/.pam_environment

    # enable now and at boot
    systemctl --user enable --now ssh-agent

    # enable lingering
    loginctl enable-linger $(whoami)

(2) Install the script to a well-known location (You can modify `/usr/lib`)

    sudo curl -o /usr/lib/systemd/systemd-user-pam-ssh \
    https://raw.githubusercontent.com/boydaihungst/systemd-user-pam-ssh/master/systemd-user-pam-ssh

    chmod 664 /usr/lib/systemd/systemd-user-pam-ssh

(3) Configure pam

    Add this to `/etc/pam.d/login`

    Add this to `/etc/pam.d/lightdm`
    auth        optional    pam_exec.so seteuid expose_authtok /usr/lib/systemd/systemd-user-pam-ssh

(4a) Use your system password as your private key passphrase (not recommended)

    ssh-keygen -p -f ~/.ssh/id_rsa
    # type your system password

(4b) Encrypt a passphrase with your system password and a heavy derivation function (recommended)

```bash
## Change your passphrase (optional)

ssh-keygen -p -f ~/.ssh/id_rsa
# type your passphrase

## Save your passphrase encrypted with your system password

read -s PASSWORD
# type your system password

read -s PASSPHRASE
# type your passphrase

echo $PASSPHRASE | openssl enc -pbkdf2 -in - -out ~/.ssh/passphrase -e -aes256 -k "$PASSWORD"

unset PASSWORD
unset PASSPHRASE
```

(4c) Use tpm2-pkcs11

```bash
read -s PASSWORD
# type your user login password
read -s PASSPHRASE
# type your ssh key passphrase, it's can be empty

#Set up tpm2 
https://www.barbhack.fr/2021/slides/barbhack2021_tpm_auth.pdf
https://github.com/tpm2-software/tpm2-pkcs11/blob/master/docs/SSH.md

tpm2_ptool init

# MyRecoveryPassword is also called admin password (use to reset or initialize
# the userpin/PASSPHRASE)
# You only need to init and addtoken once time, if you have more than 1 ssh key, repeat addkey cmd.
tpm2_ptool addtoken --pid=1 --label=ssh --userpin=$PASSPHRASE --sopin=MyRecoveryPassword

# Create new key and add it to tpm2 (Note: only support rsa2048 and ecc256 algorithms)
# Check does your system support which algorithm with: 
# tpm2_testparms rsa4096 -> show unsupported
tpm2_ptool addkey --label=ssh --userpin=$PASSPHRASE --algorithm=rsa2048 # rsa key
tpm2_ptool addkey --label=ssh --userpin=$PASSPHRASE --algorithm=ecc256 # ed25519 key


# Or import key from key file. (Note: only support rsa2048 and ecc256 algorithms)
tpm2_ptool import --label=ssh --privkey=$HOME/.ssh/id_rsa --userpin=$PASSPHRASE #id_rsa
tpm2_ptool import --label=ssh --privkey=$HOME/.ssh/id_ed25519 --userpin=$PASSPHRASE #id_ed25519

# Remember, backup won't work with tpm2. So keep your id_rsa/id_ed25519
# in a safe place in case you reinstall os.
# ~/.ssh/config
Host *
  PKCS11Provider /usr/lib/pkcs11/libtpm2_pkcs11.so
  ForwardAgent yes
  AddKeysToAgent yes

unset PASSWORD
unset PASSPHRASE
```

---
This is an extended version of the [original script by EvanPurkhiser](https://github.com/EvanPurkhiser/systemd-user-pam-ssh)
