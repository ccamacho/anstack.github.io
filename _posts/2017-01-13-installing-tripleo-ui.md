---
layout: post
title: "Installing the TripleO UI"
author: "Carlos Camacho"
categories:
  - blog
tags:
  - tripleo
  - openstack
commentIssueId: 27
---

This is a brief recipe to use or install TripleO UI
in the Undercloud.

First, once installed the Undercloud, the TripleO UI
is already available in the 3000 port.

Let's assume you have both root password for your
development environment and the Undercloud node.

TripleO-UI queries directly the endpoints (i.e. keystone)
from your browser, so we need the traffic for the net
192.168.24.0/24 forwarded from your workstation to the
Undercloud node in order to reach all required ports
(6385, 5000, 8004, 8080, 9000, 8989, 3000, 13385, 13000, 13004, 13808, 9000, 13989 and 443).

Let's install sshuttle in your workstation.

```
sudo yum install -y sshuttle
```

Now, let's get the Undercloud IP and configure SSH with a ProxyCommand.

```
instack_mac=`ssh root@labserver "tripleo get-vm-mac instack"`
undercloudIp=`ssh root@labserver "sudo virsh domifaddr instack" | grep $instack_mac | awk '{print $4}' | sed 's/\/.*$//'`

cat << EOF >> ~/.ssh/config
Host lab
  Hostname labserver
  User root
Host uc
  Hostname $undercloudIp
  User root
  ProxyCommand ssh -vvvv -W %h:%p root@lab
EOF
```

sshuttle will ask you for your hypervisor and Undercloud root
password.

To start forwarding the traffic execute:

```
sshuttle -e "ssh -vvv" -r root@uc -vvvv 192.168.24.0/24
```

Once you have done this, open from your browser http://192.168.24.1:3000/
and the TripleO UI should be shown correctly.



If you need a TripleO UI development environment follow:

The first step will be to install the TripleO UI and
all the dependencies.

```bash
  cd
  sudo yum install -y nodejs npm tmux
  git clone https://github.com/openstack/tripleo-ui.git
  cd tripleo-ui
  npm install
```

Now, we need to update all the TripleO UI config files

```bash
  cd
  cp ~/tripleo-ui/dist/tripleo_ui_config.js.sample ~/tripleo-ui/dist/tripleo_ui_config.js
  echo "Changing the default IP"
  export ENDPOINT_ADDR=$(cat stackrc | grep OS_AUTH_URL= | awk -F':' '{print $2}'| tr -d /)
  sed -i "s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/$ENDPOINT_ADDR/g" ~/tripleo-ui/dist/tripleo_ui_config.js

  echo "Removing comments for the keystone URI"
  sed -i '/^  \/\/ \"keystone\"\:/s/^  \/\///' ~/tripleo-ui/dist/tripleo_ui_config.js

  echo "Removing comments for the zaqar_default_queue"
  sed -i '/^  \/\/ \"zaqar_default_queue\"\:/s/^  \/\///' ~/tripleo-ui/dist/tripleo_ui_config.js

  # Uncomment all the parameters
  # sed -i '/^  \/\/ \".*\"\:/s/^  \/\///' ~/tripleo-ui/dist/tripleo_ui_config.js

  echo "Changing listening port for the dev server, 3000 already used"
  sed -i '/port: 3000/s/3000/33000/' ~/tripleo-ui/webpack.config.js
```

In the following step we will use tmux to persist the service running
for debugging purposes.

```bash
  cd
  tmux new -s tripleo-ui
  cd ~/tripleo-ui/
  npm start
```

At this stage you should have up and running your node server (33000 port).

If you followed the first step to see the default TripleO UI installation
go to log in the TripleO UI:  http://localhost:33000/

Happy TripleOing!

<div style="font-size:10px">
  <blockquote>
    <p><strong>Updated 2017/01/13:</strong> First version.</p>
    <p><strong>Updated 2017/01/14:</strong> Add default TripleO UI info. Still getting 'Connection to Keystone is not available'
    the config params are correct, checking it...</p>
    <p><strong>Updated 2017/01/17:</strong> Forwarded all the required ports using sshuttle.</p>
  </blockquote>
</div>
