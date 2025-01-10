# -*- mode: ruby -*-
# vi: set ft=ruby :

# コンフィグ関数
def ConfigCentosEasy(config , hostname , ip)
  config.vm.define hostname do |node|
    # ホスト名
    node.vm.hostname = hostname
    # IPアドレス(プライベート接続)
    node.vm.network "private_network", ip: ip
    # IPアドレス(ブリッジ接続)
    # node.vm.network "public_network", ip: ip
    # 起動時実行コマンド
      # - タイムゾーンを日本に設定
    node.vm.provision "shell", inline: <<-SHELL
      timedatectl set-timezone Asia/Tokyo
      SHELL
  end

  config.vm.provider "virtualbox" do |vb|
    # メモリ512MB
    vb.memory = "512"
    # CPU数1台
    vb.cpus = "1"
  end
end

# 本処理
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  # vagrant
  ConfigCentosEasy(config, "vagrant" , "192.168.33.10") 

  # Ansibleによるプロビジョニング
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "provisioning/site.yml"
    ansible.limit = 'all'
  end

  # 再起動(SELinux無効反映のため)
  # ※プラグインのインストールが必要 : vagrant plugin install vagrant-reload
  config.vm.provision "reload"

end
