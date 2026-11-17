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
    node.vm.network "forwarded_port", guest: 8080, host: 18080

    node.vm.provider :digital_ocean do |provider, override|
      override.vm.box = "digital_ocean"
      override.vm.box_url = "https://github.com/devopsgroup-io/vagrant-digitalocean/raw/master/box/digital_ocean.box"
      override.ssh.private_key_path = File.expand_path(ENV['SSH_KEY_PATH'])
      override.ssh.username = "root"
      override.nfs.functional = false

      override.vm.synced_folder "/Users/adeboyefrancis/Inno-Labs/DevOps-Intro/app", "/opt/quicknotes",
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