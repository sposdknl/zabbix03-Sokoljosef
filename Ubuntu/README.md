# Install Zabbix Agent2 on Ubuntu
Repositories for teaching purposes at SPOS DK

![Ubuntu and ZabbixAgent2 OSY AI](../Images/osy-Ubuntu-ZabbixAgent2.webp)

Repository pro vyuku na SPOS DK

## Automatická instalace Zabbix Agent2 na OS Linux Ubuntu

- Vagrantfile obsahuje sekci pro aplikaci příkazů pro instalaci monitorovacího
[Zabbix Agent2](https://www.zabbix.com/).

### Instalace Zabbix Agent2

```console
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu22.04_all.deb
dpkg -i zabbix-release_latest+ubuntu22.04_all.deb

apt-get update
apt-get install -y zabbix-agent2 zabbix-agent2-plugin-*

systemctl enable zabbix-agent2
systemctl start zabbix-agent2
```

### Konfigurace Zabbix Agent2

#!/bin/bash

# Vygeneruj unikátní hostname a zkrať ho
UUID=$(uuidgen)
FULL_HOSTNAME="ubuntu-$UUID"
SHORT_HOSTNAME="${FULL_HOSTNAME%%-*}-${FULL_HOSTNAME#*-}"

# Záloha původního konfiguračního souboru
ORIG_CONF="/etc/zabbix/zabbix_agent2.conf-orig"
CONF="/etc/zabbix/zabbix_agent2.conf"
sudo cp -v "$CONF" "$ORIG_CONF"

# Odstraň staré hodnoty
for KEY in Hostname Server ServerActive HostMetadata Timeout; do
    sudo sed -i "/^$KEY=/d" "$CONF"
done

# Přidej nové nastavení
NEW_CONF=$(cat <<EOF

### Custom config for autoreg ###
Hostname=$SHORT_HOSTNAME
Server=192.168.1.2
ServerActive=192.168.1.2
HostMetadata=SPOS
Timeout=30
EOF
)
echo "$NEW_CONF" | sudo tee -a "$CONF" > /dev/null

# Porovnej starý a nový soubor
sudo diff -u "$ORIG_CONF" "$CONF"

# Restartuj agenta
sudo systemctl restart zabbix-agent2

### Upravený vagrantfile

IMAGE_NAME = "ubuntu/jammy64"

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false

    config.vm.provider "virtualbox" do |v|
        v.memory = 2048
        v.cpus = 2
    end

    config.vm.define "ubuntu" do |ubuntu|
        ubuntu.vm.box = IMAGE_NAME
        ubuntu.vm.hostname = "ubuntu"

        # NAT síť s port forwarding pro SSH
        ubuntu.vm.network "forwarded_port", guest: 22, host: 2223, host_ip: "127.0.0.1"

        # Interní síť pro komunikaci se Zabbix Appliance
        ubuntu.vm.network "private_network", ip: "192.168.1.3", virtualbox__intnet: "intnet"

        # Přenos veřejného klíče
        ubuntu.vm.provision "file", source: "id_rsa.pub", destination: "~/.ssh/me.pub"

        # Přidání veřejného klíče do authorized_keys
        ubuntu.vm.provision "shell", inline: <<-SHELL
          cat /home/vagrant/.ssh/me.pub >> /home/vagrant/.ssh/authorized_keys
        SHELL

        # Instalace Zabbix Agent2
        ubuntu.vm.provision "shell", path: "install-zabbix-agent2.sh"

        # Konfigurace Zabbix Agent2 pro auto-registraci
        ubuntu.vm.provision "shell", path: "configure-zabbix-agent2.sh"
    end
end

...
