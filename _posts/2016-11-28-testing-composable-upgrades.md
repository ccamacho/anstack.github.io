---
layout: post
title: "Testing composable upgrades"
author: "Carlos Camacho"
categories:
  - blog
tags:
  - tripleo
  - openstack
commentIssueId: 25
---

This is a brief recipe about how I'm testing
composable upgrades N->O.

Based on the original shardy's notes
from [this](http://paste.openstack.org/show/590436/) link.

The following steps are followed to upgrade your Overcloud from Mitaka to latest master (Ocata).

- Deploy latest master TripleO following [this](http://www.anstack.com/blog/2016/07/04/manually-installing-tripleo-recipe.html) post.

- Remove the current Overcloud deployment.

```
  source stackrc
  heat stack-delete overcloud
```

- Remove the Overcloud images and create new ones (for the Newton Overcloud).

```
  cd
  openstack image list
  openstack image delete <image_ID> #Delete all the Overcloud images overcloud-full* 
  rm -rf /home/stack/overcloud-full.*

  export STABLE_RELEASE=newton
  export USE_DELOREAN_TRUNK=1
  export DELOREAN_TRUNK_REPO="https://trunk.rdoproject.org/centos7-newton/current/"
  export DELOREAN_REPO_FILE="delorean.repo"
  /home/stack/tripleo-ci/scripts/tripleo.sh --overcloud-images

  # Or reuse images
  # wget https://buildlogs.centos.org/centos/7/cloud/x86_64/tripleo_images/newton/delorean/overcloud-full.tar
  # tar -xvf overcloud-full.tar
  # openstack overcloud image upload --update-existing
```

- Download Newton tripleo-heat-templates.

```
  cd
  git clone -b stable/newton https://github.com/openstack/tripleo-heat-templates tht-newton
```

- Configure the DNS (needed when upgrading the Overcloud).

```
  neutron subnet-update `neutron subnet-list -f value | awk '{print $1}'` --dns-nameserver 192.168.122.1
```

- Deploy a Newton Overcloud.

```
  openstack overcloud deploy \
  --libvirt-type qemu \
  --templates /home/stack/tht-newton/ \
  -e /home/stack/tht-newton/overcloud-resource-registry-puppet.yaml \
  -e /home/stack/tht-newton/environments/puppet-pacemaker.yaml
```

- Install prerequisites in nodes (if no DNS configured this will fail).

```
cat > upgrade_repos.yaml << EOF
parameter_defaults:
  UpgradeInitCommand: |
    set -e

    #Master repositories
    sudo curl -o /etc/yum.repos.d/delorean.repo https://buildlogs.centos.org/centos/7/cloud/x86_64/rdo-trunk-master-tripleo/delorean.repo
    sudo curl -o /etc/yum.repos.d/delorean-deps.repo https://trunk.rdoproject.org/centos7/delorean-deps.repo


    export HOME=/root
    cd /root/
    if [ ! -d tripleo-ci ]; then
      git clone https://github.com/openstack-infra/tripleo-ci.git
    else
      pushd tripleo-ci
      git checkout master
      git pull
      popd
    fi

    if [ ! -d tripleo-heat-templates ]; then
      git clone https://github.com/openstack/tripleo-heat-templates.git
    else
      pushd tripleo-heat-templates
      git checkout master
      git pull
      popd
    fi

    ./tripleo-ci/scripts/tripleo.sh --repo-setup
    sed -i "s/includepkgs=/includepkgs=python-heat-agent*,/" /etc/yum.repos.d/delorean-current.repo
    #yum -y install python-heat-agent-ansible
    yum install -y python-heat-agent-*

    rm -f /usr/libexec/os-apply-config/templates/etc/puppet/hiera.yaml
    rm -f /usr/libexec/os-refresh-config/configure.d/40-hiera-datafiles
    rm -f /etc/puppet/hieradata/*.yaml
    yum remove -y python-UcsSdk openstack-neutron-bigswitch-agent python-networking-bigswitch openstack-neutron-bigswitch-lldp python-networking-odl
    crudini --set /etc/ansible/ansible.cfg DEFAULT library /usr/share/ansible-modules/
EOF
```

- Download master tripleo-heat-templates.

```
  cd
  git clone https://github.com/openstack/tripleo-heat-templates tht-master
```

- Upgrade Overcloud to master

```
  cd
  openstack overcloud deploy \
  --templates /home/stack/tht-master/ \
  -e /home/stack/tht-master/overcloud-resource-registry-puppet.yaml \
  -e /home/stack/tht-master/environments/puppet-pacemaker.yaml \
  -e /home/stack/tht-master/environments/major-upgrade-composable-steps.yaml \
  -e upgrade_repos.yaml
```

If the last steps manage to finish successfully (WIP), you just have upgraded your Overcloud from Newton to Ocata (latest master).

For more resources related to TripleO deployments, check out the [TripleO YouTube channel](https://www.youtube.com/channel/UCNGDxZGwUELpgaBoLvABsTA).

<div style="font-size:10px">
  <blockquote>
    <p><strong>Updated 2017/01/28:</strong> Working fine.</p>
  </blockquote>
</div>
