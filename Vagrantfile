# Campus IL - EdX platform and ecommerce themes
# 
# Copyright (C) 2021 Campus IL
# 
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
# 
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <https://www.gnu.org/licenses/>.

Vagrant.require_version ">= 1.8.7"
unless Vagrant.has_plugin?("vagrant-vbguest")
  raise "Please install the vagrant-vbguest plugin by running `vagrant plugin install vagrant-vbguest`"
end

VAGRANTFILE_API_VERSION = "2"

MEMORY = 4096
CPU_COUNT = 2

# These are versioning variables in the roles. Each can be overridden, first
# with OPENEDX_RELEASE, and then with a specific environment variable of the
# same name but upper-cased.
VERSION_VARS = [
    'edx_platform_version',
    'configuration_version',
    'certs_version',
    'forum_version',
    'xqueue_version',
    'demo_version',
    'NOTIFIER_VERSION',
    'ECOMMERCE_VERSION',
    'ECOMMERCE_WORKER_VERSION',
    'DISCOVERY_VERSION',
]

MOUNT_DIRS = {
  :edx_platform => {:repo => "edx-platform", :local => "/edx/app/edxapp/edx-platform", :owner => "edxapp"},
  :themes => {:repo => "themes", :local => "/edx/app/edxapp/themes", :owner => "edxapp"},
  :forum => {:repo => "cs_comments_service", :local => "/edx/app/forum/cs_comments_service", :owner => "forum"},
  :ecommerce => {:repo => "ecommerce", :local => "/edx/app/ecommerce/ecommerce", :owner => "edxapp"},
  :ecommerce_worker => {:repo => "ecommerce-worker", :local => "/edx/app/ecommerce_worker/ecommerce_worker", :owner => "edxapp"},
  # This src directory won't have useful permissions. You can set them from the
  # vagrant user in the guest OS. "sudo chmod 0777 /edx/src" is useful.
  :src => {:repo => "src", :local => "/edx/src", :owner => "root"},

}
if ENV['VAGRANT_MOUNT_BASE']
  MOUNT_DIRS.each { |k, v| MOUNT_DIRS[k][:repo] = ENV['VAGRANT_MOUNT_BASE'] + "/" + MOUNT_DIRS[k][:repo] }
end

# map the name of the git branch that we use for a release
# to a name and a file path, which are used for retrieving
# a Vagrant box from the internet.
openedx_releases = {
  "opencraft-release/ginkgo.2-campus" => "edx-olive/ginkgo.2-campus-devstack",
}

openedx_releases.default = "edx-olive/ginkgo.2-campus-devstack"
openedx_release = ENV['OPENEDX_RELEASE'] || "opencraft-release/ginkgo.2-campus"

boxname = ENV['OPENEDX_BOXNAME']

# Build -e override lines for each overridable variable.
extra_vars_lines = ""
VERSION_VARS.each do |var|
  rel = ENV[var.upcase] || openedx_release
  if rel
    extra_vars_lines += "-e #{var}=#{rel} \\\n"
  end
end
campus_vars_file = "/tmp/campus-vars.yml"

$script = <<SCRIPT
# Update expired keys on the base ginkgo box
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 69464050

if [ ! -d /edx/app/edx_ansible ]; then
    echo "Error: Base box is missing provisioning scripts." 1>&2
    exit 1
fi
if [ ! -f #{campus_vars_file} ]; then
    echo "Error: Missing Campus IL vars." 1>&2
    exit 1
fi
if [ -d /edx/app/edx_ansible/venvs/edx_ansible/ ]; then
    # Fix the permission in the configuration venv; this is required for reprovisioning
    sudo chown -R edx-ansible:edx-ansible /edx/app/edx_ansible/venvs/edx_ansible/
fi
export PYTHONUNBUFFERED=1
source /edx/app/edx_ansible/venvs/edx_ansible/bin/activate
cd /edx/app/edx_ansible/edx_ansible/playbooks

EXTRA_VARS="#{extra_vars_lines}"
CONFIG_VER="#{ENV['CONFIGURATION_VERSION'] || openedx_release || 'master'}"

ansible-playbook -i localhost, -c local run_role.yml -e role=edx_ansible -e configuration_version=$CONFIG_VER $EXTRA_VARS -e @#{campus_vars_file}
ansible-playbook -i localhost, -c local vagrant-devstack.yml -e configuration_version=$CONFIG_VER $EXTRA_VARS -e @#{campus_vars_file}

SCRIPT

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  boxfile = ""
  if not boxname
    reldata = openedx_releases[openedx_release]
    if Hash == reldata.class
      boxname = openedx_releases[openedx_release][:name]
      boxfile = openedx_releases[openedx_release].fetch(:file, "")
    else
      boxname = reldata
    end
  end
  if boxfile == ""
    boxfile = "#{boxname}.box"
  end

  # Creates an edX devstack VM from an official release
  config.vm.box = boxname
  config.vm.network :private_network, ip: "192.168.33.10"

  # If you want to run the box but don't need network ports, set VAGRANT_NO_PORTS=1.
  # This is useful if you want to run more than one box at once.
  if not ENV['VAGRANT_NO_PORTS']
    config.vm.network :forwarded_port, guest: 8000, host: 8000  # LMS
    config.vm.network :forwarded_port, guest: 8001, host: 8001  # Studio
    config.vm.network :forwarded_port, guest: 8002, host: 8002  # Ecommerce
    config.vm.network :forwarded_port, guest: 8003, host: 8003  # LMS for Bok Choy
    config.vm.network :forwarded_port, guest: 8031, host: 8031  # Studio for Bok Choy
    config.vm.network :forwarded_port, guest: 8120, host: 8120  # edX Notes Service
    config.vm.network :forwarded_port, guest: 8765, host: 8765
    config.vm.network :forwarded_port, guest: 9200, host: 9200  # Elasticsearch
    config.vm.network :forwarded_port, guest: 18080, host: 18080  # Forums
    config.vm.network :forwarded_port, guest: 8100, host: 8100  # Analytics Data API
    config.vm.network :forwarded_port, guest: 8110, host: 8110  # Insights
    config.vm.network :forwarded_port, guest: 9876, host: 9876  # ORA2 Karma tests
    config.vm.network :forwarded_port, guest: 50070, host: 50070  # HDFS Admin UI
    config.vm.network :forwarded_port, guest: 8088, host: 8088  # Hadoop Resource Manager
    config.vm.network :forwarded_port, guest: 18381, host: 18381    # Course discovery
  end
  
  config.ssh.insert_key = true

  config.vm.synced_folder  ".", "/vagrant", disabled: true

  # Enable X11 forwarding so we can interact with GUI applications
  if ENV['VAGRANT_X11']
    config.ssh.forward_x11 = true
  end

  if ENV['VAGRANT_USE_VBOXFS'] == 'true'
    MOUNT_DIRS.each { |k, v|
      config.vm.synced_folder v[:repo], v[:local], create: true, owner: v[:owner], group: "www-data"
    }
  else
    MOUNT_DIRS.each { |k, v|
      config.vm.synced_folder v[:repo], v[:local], create: true, nfs: true
    }
  end

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", MEMORY.to_s]
    vb.customize ["modifyvm", :id, "--cpus", CPU_COUNT.to_s]
    vb.customize ["modifyvm", :id, "--uart1", "off"]
    vb.customize ['modifyvm', :id, '--nictype1', 'virtio']
  end

  # Disable auto-update as it causes plugin failures
  config.vbguest.auto_update = false

  # Assume that the base box has the edx_ansible role installed
  # We can then tell the Vagrant instance to update itself.
  config.vm.provision "file", source: "campus-vars.yml", destination: campus_vars_file
  config.vm.provision "shell", inline: $script
end
