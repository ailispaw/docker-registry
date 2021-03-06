# A dummy plugin for Barge to set hostname and network correctly at the very first `vagrant up`
module VagrantPlugins
  module GuestLinux
    class Plugin < Vagrant.plugin("2")
      guest_capability("linux", "change_host_name") { Cap::ChangeHostName }
      guest_capability("linux", "configure_networks") { Cap::ConfigureNetworks }
    end
  end
end

REGISTRY_IP = "192.168.33.201"

Vagrant.configure(2) do |config|
  config.vm.define "docker-registry"

  config.vm.box = "ailispaw/barge"

  config.vm.network :forwarded_port, guest: 2375, host: 2375, auto_correct: true, disabled: true

  config.vm.network :private_network, ip: "#{REGISTRY_IP}"

  config.vm.synced_folder ".", "/vagrant", type: "nfs",
    mount_options: ["nolock", "vers=3", "udp", "noatime", "actimeo=1"]

  config.vm.provision :shell do |sh|
    sh.inline = <<-EOT
      echo 'DOCKER_EXTRA_ARGS="--userland-proxy=false \
        --registry-mirror=http://#{REGISTRY_IP}:5000 \
        --insecure-registry=#{REGISTRY_IP}:5000"' >> /etc/default/docker
      /etc/init.d/docker restart
    EOT
  end

  config.vm.provision :shell, run: "always" do |sh|
    sh.inline = <<-EOT
      docker rm -f registry || true
      docker rm -f frontend || true
    EOT
  end

  config.vm.provision "registry", type: "docker", run: "always" do |docker|
    docker.pull_images "registry:2"
    docker.run "registry",
      image: "registry:2",
      args: [
        "-p 5000:5000",
        "-v /vagrant/registry:/var/lib/registry",
        "-v /vagrant/config.yml:/etc/docker/registry/config.yml"
      ].join(" "),
      restart: false
  end

  config.vm.provision "frontend", type: "docker", run: "always" do |docker|
    docker.pull_images "konradkleine/docker-registry-frontend:v2"
    docker.run "frontend",
      image: "konradkleine/docker-registry-frontend:v2",
      args: [
        "-p 80:80",
        "-e ENV_DOCKER_REGISTRY_HOST=#{REGISTRY_IP}",
        "-e ENV_DOCKER_REGISTRY_PORT=5000",
        "-v /usr/bin/dumb-init:/dumb-init:ro --entrypoint=/dumb-init"
      ].join(" "),
      cmd: "/root/start-apache.sh",
      restart: false
  end
end
