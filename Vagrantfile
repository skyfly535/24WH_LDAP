# -*- mode: ruby -*-
# vim: set ft=ruby :

Vagrant.configure("2") do |config|
  # Указываем ОС, версию, количество ядер и ОЗУ
  config.vm.box = "bento/centos-8.4"
  #config.vm.box_version = "20210210.0"

  config.vm.provider :virtualbox do |v|
    v.memory = 1024
    v.cpus = 1
  end

  # Указываем имена хостов и их IP-адреса
  boxes = [
    { :name => "ipa.otus.lan",
      :ip => "192.168.57.10",
    },
    { :name => "client1.otus.lan",
      :ip => "192.168.57.11",
    },
    { :name => "client2.otus.lan",
      :ip => "192.168.57.12",
    }
  ]
  # Цикл запуска виртуальных машин
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network "private_network", ip: opts[:ip]
      config.vm.provision "shell", run: "always", inline: <<-SHELL
      cd /etc/yum.repos.d/
      sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-* # меняем настройки репозиторий, так состарыми ничего нельзя поставить
      sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
      SHELL
    end
  end
end
