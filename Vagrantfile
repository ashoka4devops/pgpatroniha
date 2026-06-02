# -*- mode: ruby -*-
# vi: set ft=ruby :
# ---------------------------------------------------------------------------
# Vagrantfile  -  Patroni + PostgreSQL 17 HA cluster, DEDICATED etcd tier.
#
# Multi-machine definition driven entirely by config/vagrant.yml. Each VM is
# provisioned by shared_scripts/setup.sh, which dispatches on the node role
# (etcd | postgres | haproxy). All topology data the guests need is computed
# here and handed to the scripts as environment variables, so the shell
# scripts stay topology-free.
#
# Unlike the stacked variant, etcd runs on its OWN nodes. Bring the etcd tier
# up first so it forms quorum, then the database nodes, then HAProxy:
#
#   vagrant up etcd1 etcd2 etcd3
#   vagrant up pg1 pg2 pg3
#   vagrant up haproxy
#
# `vagrant up` with no args also works - Vagrant follows the order in
# vagrant.yml (etcd nodes are listed first).
# ---------------------------------------------------------------------------

require "yaml"

VAGRANT_ROOT = File.dirname(File.expand_path(__FILE__))
CONFIG       = YAML.load_file(File.join(VAGRANT_ROOT, "config", "vagrant.yml"))

NODES = CONFIG.fetch("nodes")
BOX   = CONFIG.fetch("box", "oraclelinux/9")

ETCD_NODES = NODES.select { |n| n["role"] == "etcd" }
PG_NODES   = NODES.select { |n| n["role"] == "postgres" }

# "etcd1:192.168.56.30 ... pg1:192.168.56.10 ... haproxy:192.168.56.20"
ALL_HOSTS = NODES.map { |n| "#{n['name']}:#{n['ip']}" }.join(" ")

# etcd initial cluster string, from the etcd tier:
#   "etcd1=http://192.168.56.30:2380,etcd2=http://192.168.56.31:2380,..."
ETCD_INITIAL_CLUSTER =
  ETCD_NODES.map { |n| "#{n['name']}=http://#{n['ip']}:2380" }.join(",")

# Patroni etcd3 endpoints, from the etcd tier:
#   "192.168.56.30:2379,192.168.56.31:2379,192.168.56.32:2379"
ETCD_ENDPOINTS = ETCD_NODES.map { |n| "#{n['ip']}:2379" }.join(",")

# HAProxy backends, from the postgres tier: "pg1:192.168.56.10 pg2:... ..."
PG_BACKENDS = PG_NODES.map { |n| "#{n['name']}:#{n['ip']}" }.join(" ")

Vagrant.configure("2") do |config|
  config.vm.box = BOX
  config.vm.box_url = CONFIG["box_url"] if CONFIG["box_url"]
  config.vm.box_check_update = false

  config.vm.synced_folder ".", "/vagrant", disabled: false
  config.vm.synced_folder "shared_scripts", "/vagrant_scripts"

  NODES.each do |node|
    config.vm.define node["name"] do |vm|
      vm.vm.hostname = node["name"]
      vm.vm.network "private_network", ip: node["ip"]

      vm.vm.provider "virtualbox" do |vb|
        vb.name   = "patroni-pg17-#{node['name']}"
        vb.memory = node["memory"]
        vb.cpus   = node["cpus"]
        vb.customize ["modifyvm", :id, "--groups", "/patroni-pg17-etcd-sep"]
      end

      vm.vm.provision "shell",
        path: "shared_scripts/setup.sh",
        env: {
          "NODE_NAME"            => node["name"],
          "NODE_IP"              => node["ip"],
          "NODE_ROLE"            => node["role"],
          "ALL_HOSTS"            => ALL_HOSTS,
          "PG_BACKENDS"          => PG_BACKENDS,
          "ETCD_INITIAL_CLUSTER" => ETCD_INITIAL_CLUSTER,
          "ETCD_ENDPOINTS"       => ETCD_ENDPOINTS
        }
    end
  end
end
