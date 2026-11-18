## Lab 5 - Virtualization: Hashicorp Vagrant (Idempotent) VM
The goal of this Lab is to have an a standardized idempotent development environment by utilising a consistent Vagrantfile that allows other team members to build and test application using the same configuration without an drift mishaps.

### Workflow - Digial Ocean 1-click App baked Vagrant VM + Required Plugin + Token
I'm using this approach as a POC to abstract heavylifting such as Hypversor and Resource constraint by leveraging Digital Ocean Infrastruture
- Install Vagrant using Homebrew
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/hashicorp-vagrant
```
- Generate API Token (with a 90 days rotation) `Login to Digital Ocean -> Settings -> API -> Generate Token`
- Install digital Ocean Plugin to manage Lifecycle of Droplet created by Vagrant `vagrant plugin install vagrant-digitalocean`
- Create `.env` file to manage secrets or Install vagrant plugin to load environment variables
- Create Vagrantfile at project root
```ruby
GO_VERSION = "1.24.5"

# Load .env without a plugin
env_file = File.join(__dir__, '.env')
if File.exist?(env_file)
  File.readlines(env_file).each do |line|
    line = line.strip
    next if line.empty? || line.start_with?('#')
    key, value = line.split('=', 2)
    ENV[key] ||= value.to_s.gsub(/\A["']|["']\z/, '')
  end
end

# Vagrant Config using Digital Ocean Provider Plugin
Vagrant.configure("2") do |config|
  config.vm.define "quicknotes" do |node|
    node.vm.hostname = "quicknotes"
    node.vm.synced_folder ".", "/vagrant", disabled: true
    node.vm.network "forwarded_port", guest: 8080, host: 18080 # Since using DO machine, Need IPV4 for vagrant droplet 

    node.vm.provider :digital_ocean do |provider, override|
      override.vm.box = "digital_ocean"
      override.vm.box_url = "https://github.com/devopsgroup-io/vagrant-digitalocean/raw/master/box/digital_ocean.box"
      override.ssh.private_key_path = File.expand_path(ENV['SSH_KEY_PATH'])
      override.ssh.username = "root"
      override.nfs.functional = false

      override.vm.synced_folder "./app", "/opt/quicknotes", # rsync to automatically sync changes in the absence of NFS & Virtual box, rysnc-auto is a better option
        type: "rsync", rsync__exclude: [".git/"]

      provider.token        = ENV['DIGITAL_OCEAN_TOKEN']
      provider.ssh_key_name = ENV['DO_SSH_KEY_NAME']
      provider.image        = "ubuntu-24-04-x64"
      provider.region       = "sfo2"
      provider.size         = "s-1vcpu-1gb"
    end

    node.vm.provision "shell", env: { "GO_VERSION" => GO_VERSION }, inline: <<-SHELL
      set -euo pipefail
      if /usr/local/go/bin/go version 2>/dev/null | grep -q "go${GO_VERSION}"; then
        echo "Go ${GO_VERSION} already installed"; exit 0
      fi
      ARCH=$(dpkg --print-architecture)
      curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-${ARCH}.tar.gz" -o /tmp/go.tgz
      rm -rf /usr/local/go
      tar -C /usr/local -xzf /tmp/go.tgz
      rm /tmp/go.tgz
      echo 'export PATH=$PATH:/usr/local/go/bin' > /etc/profile.d/go.sh
      /usr/local/go/bin/go version
    SHELL
  end
end
```
- Start Machine to provision VM `vagrant up --provider=digital_ocean`

### Design Outcome Questions:
- Synced folders: Using `rsync` option for this lab as it enables a seamless integration of working files from Host machine to Vagrant VM.
- NAT vs Bridged vs Host-only:This lab use Vagrant`nat` default tto provide VM with its own private IP and share host ip outbound to allow internet connectivity.
- Go Version 1.24.5: This approach ensures that quicknotes has a consistent environment for dependency alignment.
