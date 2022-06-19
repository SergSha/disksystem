<h3>### Дисковая подсистема ###</h3>

<h4>Описание домашнего задания</h4>

<ul>
<li>добавить в Vagrantfile еще дисков;</li>
<li>сломать/починить raid;</li>
<li>собрать R0/R5/R10 на выбор;</li>
<li>прописать собранный рейд в конф, чтобы рейд собирался при загрузке;</li>
<li>создать GPT раздел и 5 партиций. В качестве проверки принимаются - измененный Vagrantfile, скрипт для создания рейда, конф для автосборки рейда при загрузке.</li>
<li>Доп. задание - Vagrantfile, который сразу собирает систему с подключенным рейдом и смонтированными разделами. После перезагрузки стенда разделы должны автоматически примонтироваться.</li>
</ul>
<br />



<h4># Добавить в Vagrantfile еще дисков</h4>

<p>В домашней директории создадим директорию disksystem, в которой будут храниться настройки виртуальной машины:</p>

<pre>[user@localhost otus]$ mkdir ./disksystem
[user@localhost otus]$</pre>

<p>Перейдём в директорию disksystem:</p>

<pre>[user@localhost otus]$ cd ./disksystem/
[user@localhost disksystem]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost disksystem]$ vi ./Vagrantfile</pre>

<p>Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
# # vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = 'almalinux/8'
  config.vm.network :forwarded_port, guest: 22, host: 4000

  config.vm.define "loadsystem" do |loadsystem|

    loadsystem.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    end

    loadsystem.vm.disk :disk, size: "1GB", name: "disk1"
    loadsystem.vm.host_name = 'loadsystem'
    loadsystem.vm.network :private_network, ip: "192.168.56.141"

  end
end
</pre>

<p>Запустим виртуальную машину:</p>

<pre>[user@localhost loadsystem]$ vagrant up</pre>
