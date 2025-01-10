# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure("2") do |config|
  config.vm.box = "bento/centos-7.7"
  # NATとプライベートネットワークを併用
  config.vm.network "forwarded_port", guest: 443, host: 8443
  config.vm.network "private_network", ip: "192.168.33.10"
  # 公開ネットワーク（ブリッジアダプタを使用）
  config.vm.network "public_network", bridge: "Realtek USB GbE Family Controller"
  # 仮想マシン設定
  config.vm.boot_timeout = 60000
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--cableconnected1", "on"]
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
  end
  # vbguestプラグインの設定
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end
  # プロビジョニングスクリプト
  config.vm.provision "shell", inline: <<-SHELL
    set -e
    # 必要なパッケージのインストール
    sudo yum clean all
    sudo yum -y update
    sudo yum -y install httpd curl unzip zip git
    # remiリポジトリの追加 (エラーを無視)
    sudo yum install -y https://rpms.remirepo.net/enterprise/remi-release-7.rpm || true
    # PHP 8.2のインストール
    sudo yum -y install php82 php82-php php82-php-bcmath php82-php-json php82-php-mbstring php82-php-pdo php82-php-xml php82-php-mysqlnd --enablerepo=remi-php82
    sudo yum -y install php82-php-fpm php82-php-cli --enablerepo=remi-php82
    # phpコマンドのシンボリックリンク作成
    sudo ln -sf /usr/bin/php82 /usr/bin/php
    # ApacheのPHPモジュールを有効化
    sudo ln -sf /opt/remi/php82/root/usr/lib64/httpd/modules/libphp.so /etc/httpd/modules/libphp8.so

    # Apache設定ファイルを修正してPHP 8.2モジュールをロード
    sudo tee /etc/httpd/conf.modules.d/00-php.conf > /dev/null <<EOF
LoadModule php_module modules/libphp8.so
AddHandler php-script .php
DirectoryIndex index.php
EOF

    # PHP設定変更（short_open_tagを有効化）
    sudo sed -i 's/short_open_tag = Off/short_open_tag = On/' /etc/opt/remi/php82/php.ini

    # MariaDBのリポジトリ追加
    sudo tee /etc/yum.repos.d/MariaDB.repo > /dev/null <<EOF
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.5/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
EOF
    # MariaDBのインストール
    sudo yum -y install MariaDB-server MariaDB-client
    sudo systemctl enable mariadb
    sudo systemctl start mariadb
    # MySQLの設定（存在チェックと作成）
    DB_EXISTS=$(mysql -u root -e "SHOW DATABASES LIKE 'matomo';" | grep matomo || true)
    if [ -z "$DB_EXISTS" ]; then
      mysql -u root -e "CREATE DATABASE matomo;"
      echo "Database 'matomo' created."
    else
      echo "Database 'matomo' already exists."
    fi
    USER_EXISTS=$(mysql -u root -e "SELECT User FROM mysql.user WHERE User = 'user';" | grep user || true)
    if [ -z "$USER_EXISTS" ]; then
      mysql -u root -e "CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';"
      mysql -u root -e "GRANT ALL PRIVILEGES ON matomo.* TO 'user'@'localhost';"
      mysql -u root -e "FLUSH PRIVILEGES;"
      echo "User 'user' created and granted privileges."
    else
      echo "User 'user' already exists."
    fi
    # Apache設定
    sudo mkdir -p /etc/ssl/certs
    sudo mkdir -p /etc/ssl/private
    sudo cp /vagrant/mycert.crt /etc/ssl/certs/
    sudo cp /vagrant/mykey.key /etc/ssl/private/
    sudo systemctl enable httpd
    sudo systemctl start httpd
    
    # 仮想ホスト設定ファイルの作成
    sudo tee /etc/httpd/conf.d/matomo.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName vm.matomo.com
    DocumentRoot /vagrant/matomo
    DirectoryIndex index.php
    <Directory /vagrant/matomo>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog /var/log/httpd/matomo_error.log
    CustomLog /var/log/httpd/matomo_access.log combined
</VirtualHost>
EOF
    # Apacheの再起動
    sudo systemctl restart httpd
  SHELL
end
