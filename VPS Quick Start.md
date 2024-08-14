So, you obtained a new Ubuntu server. Let's configure it a bit.
# Create a new user

- Update and upgrade:

```bash
apt update && apt upgrade -y
```

- Change the hostname: 

```bash
hostname <new-hostname>
```

- Create a new user and add them to sudo:

```bash
adduser <new-user>
usermod -aG sudo <newuser>
```

- Disconnect from root:

```bash
exit
```


# Configure ssh

On your local machine:

- Copy your ssh key to the new user:

```bash
ssh-copy-id -i <ssh-identity> <new-user>@<remote-ip>
```

- Connect to the remote with the new user:

```bash
ssh <new-user>@<remote-ip>
```

- Open the ssh config to change some security options:

```bash
sudo vim /etc/ssh/sshd_config
```

- Change this:
	- Change port: `Port 6022`
	- Disable root login: `PermitRootLogin no`
	- Disable password authentication: `PasswordAuthentication no`

- Restart sshd and disconnect:

```bash
sudo systemctl restart sshd
exit
```

- On your local machine, add alias for easy access. Open `.zshrc` and add:

```bash
export <SERVER_ENV_VAR>="<new-user>@<server-ip>"
alias ssh-server-name="ssh -p 6022 -i <ssh-identity> $<SERVER_ENV_VAR>"
```

- Source the config:

```bash
. ./.zshrc
```

# Some useful configuration (not required)

## Install Oh-My-Zsh:

```bash
sudo apt install zsh curl
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

Switch theme. Open `.zshrc` and change the `ZSH_THEME` var:

```bash
ZSH_THEME="gallifrey"
```

Source the `.zshrc`:

```bash
. ./.zshrc
```

## Use specific ssh key for git

Open `~/.ssh/config` and add the following:

```bash
Host github.com
 HostName github.com
 IdentityFile ~/.ssh/id_rsa_github
```

Set the right permissions:

```bash
chmod 600 ~/.ssh/config
```

# Configure firewall with iptables

More info [here](https://youtu.be/Q0EC8kJlB64).

## Explore current state

First, check the current state:

```bash
sudo iptables -L -nv
```

Should be no rules yet. 

Now let's check what sockets are being used at the moment:

```bash
sudo ss -ntulp
```

Also we need to know what network interfaces are on our server:

```bash
ip a
```

## Minimal working configuration

First, let's allow the ssh connections. Earlier, we configured ssh to listen the port 6022. Let's now open this port:

```bash
sudo iptables -A INPUT -p tcp --dport=6022 -j ACCEPT
```

Then, allow localhost connections. We allow any traffic on the local interface, so that our local applications can communicate with each other:

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
```

Allow the `icmp` protocol, so the ping packet will be accepted:

```bash
sudo iptables -A INPUT -p icmp -j ACCEPT
```

If you are going to host a web server, it's worth opening ports 80 and 443:

```bash
sudo iptables -A INPUT -p tcp --dport=80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport=443 -j ACCEPT
```

And let's allow connections established by our own server:

```bash
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Now it's time to **double check the ssh rules**. If everything is okay, that means we are all set for changing the default policy:

```bash
sudo iptables -P INPUT DROP
```

In case something went wrong after this step, just reboot the server. iptables configuration is not persistent at this time yet.

## Maintain persistence

We want our rules to be persistent after reboots. So we need to install some more packets:

```bash
sudo apt install iptables-persistent netfilter-persistent -y
```

