# Setup guide

Everything below runs on the Ubuntu VM that already hosts the Wazuh all-in-one stack, unless stated otherwise.
Only use this in a lab you own.

## 0. Check resources first

Shuffle needs at least 4 GB of RAM of its own, and the Wazuh indexer is memory-hungry too. Check before you start:

```bash
free -h
df -h /
```

In this lab the VM was raised to 8 GB RAM. If `df` shows far less space than the virtual disk you assigned,
the LVM volume still needs extending (`growpart`, `pvresize`, `lvextend -l +100%FREE`, `resize2fs`).

## 1. Install Docker

Docker's own repository is used because Ubuntu's default package is older.

```bash
sudo apt update && sudo apt install -y ca-certificates curl git
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

## 2. Install Shuffle

```bash
git clone https://github.com/Shuffle/Shuffle
cd Shuffle
mkdir shuffle-database
sudo chown -R 1000:1000 shuffle-database
```

Before the first start, apply the two edits in [`../configs/docker-compose-changes.md`](../configs/docker-compose-changes.md)
(published port 9201, smaller JVM heap). Then:

```bash
docker compose up -d
docker compose ps
```

All five containers (backend, frontend, opensearch, orborus, security) should show `Up`.
Shuffle starts several app containers after this; they are managed by orborus.

## 3. Create the workflow

1. Open `http://<ubuntu-ip>:3001` and create your organisation and admin account (there is no default login).
2. Create a workflow and add a **Webhook** trigger. Start it and copy its URI.
3. Add a **Discord** action ("Send json data as webhook"), paste your Discord webhook URL, and use the body in
   [`../configs/discord-message-body.json`](../configs/discord-message-body.json).
4. Connect the webhook to the Discord action. Optionally add a condition on the connection.
5. Save.

## 4. Point Wazuh at Shuffle

Back up the config, then add the block from [`../configs/ossec-integration.xml`](../configs/ossec-integration.xml):

```bash
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak
sudo nano /var/ossec/etc/ossec.conf
```

Validate before restarting. A broken `ossec.conf` stops the manager from starting:

```bash
(echo "<root>"; sudo cat /var/ossec/etc/ossec.conf; echo "</root>") | python3 -c "import sys, xml.dom.minidom as m; m.parseString(sys.stdin.read()); print('XML OK')"
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```

The Python check reports the line of the problem (add 1 for the wrapper line); Wazuh's own error is just "line 0".

## 4b. Confirm the integrator is sending

```bash
sudo tail -n 15 /var/ossec/logs/integrations.log
```

Each line is an alert file passed to the Shuffle integration. Alerts below `<level>` are not forwarded.

## 5. Test

On the attacker VM, prepare a short wordlist and run Hydra against the SSH service:

```bash
sudo gunzip /usr/share/wordlists/rockyou.txt.gz     # first use only
head -n 30 /usr/share/wordlists/rockyou.txt > /tmp/short-list-30.txt
hydra -l <user> -P /tmp/short-list-30.txt ssh://<ubuntu-ip> -t 4
```

Then check, in order: the Wazuh Security Events dashboard, `integrations.log`, the workflow's run list in Shuffle,
and the Discord channel. The failures are logged on the Wazuh manager's own host, so they appear under agent 000.
