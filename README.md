## Задача 1
### Опишите своими словами основные преимущества применения на практике IaaC паттернов.
- увеличивается скорость разворачивания инфраструктуры
- минимизируются отличия между dev, test и prod окружениями
- более быстрая подготовка песочниц для разработчиков, увеличение скорости разработки
### Какой из принципов IaaC является основополагающим?
Идемпотентность - когда при повторном выполнении операции получается идентичный результат

## Задача 2
### Чем Ansible выгодно отличается от других систем управление конфигурациями?
Ansible не требует установки клиента, работает по ssh.
### Какой, на ваш взгляд, метод работы систем конфигурации более надёжный push или pull?
Зависит от ситуации, так например если между клиентом и сервером системы управления конфигурациями есть файерволы и NAT, то в этом случае целесообразнее использовать метод pull.

Если такие сетевые технологии не используются - то тут лучше будет push, т. к. применение конфигурации будет инициироваться с сервера управления конфигурациями, что в итоге происходит более оперативно.

## Задача 3 Установить на личный компьютер: VirtualBox, Vagrant, Ansible
Приложить вывод команд установленных версий каждой из программ, оформленный в markdown.

```
dmitrii@dmitrii-virtual-machine:~$ vagrant -v
Vagrant 2.2.6
dmitrii@dmitrii-virtual-machine:~$ ansible --version
ansible 2.9.6
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/dmitrii/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  executable location = /bin/ansible
  python version = 3.8.10 (default, Nov 26 2021, 20:14:08) [GCC 9.3.0]
dmitrii@dmitrii-virtual-machine:~$ vboxmanage -v
6.1.26_Ubuntur145957
```

## Задача 4 (*)
### Воспроизвести практическую часть лекции самостоятельно.
Расположение файлов
```
$ tree homework
homework
├── ansible.cfg
├── inventory
├── provision.yml
└── Vagrantfile
```
Содержимое файлов
```
$ cat homework/ansible.cfg 
[defaults]
inventory=./inventory
deprecation_warnings=False
command_warnings=False
ansible_port=22
interpreter_python=/usr/bin/python3
```
```
$ cat homework/inventory 
[nodes:children]
manager
[manager]
server1.netology ansible_host=127.0.0.1 ansible_port=20011 ansible_user=vagrant
```
```
$ cat homework/provision.yml 
---
- hosts: nodes
  become: yes
  become_user: root
  remote_user: vagrant
  tasks:
    - name: Create directory for ssh-keys
      file: state=directory mode=0700 dest=/root/.ssh/

    - name: Adding rsa-key in /root/.ssh/authorized_keys
      copy: src=~/.ssh/id_rsa.pub dest=/root/.ssh/authorized_keys owner=root mode=0600
      ignore_errors: yes

    - name: Checking DNS
      command: host -t A google.com

    - name: Installing tools
      apt: >
          package={{ item }}
          state=present
          update_cache=yes
      with_items:
        - git
        - curl

    - name: Installing docker
      shell: curl -fsSL get.docker.com -o get-docker.sh && chmod +x get-docker.sh && ./get-docker.sh

    - name: Add the current user to docker group
      user: name=vagrant append=yes groups=docker
```
```
$ cat homework/Vagrantfile 
ISO = "bento/ubuntu-20.04"
NET = "192.168.192."
DOMAIN = ".netology"
HOST_PREFIX = "server"
INVENTORY_PATH = "./inventory"


servers = [
  {
    :hostname => HOST_PREFIX + "1" + DOMAIN,
    :ip => NET + "11",
    :ssh_host => "20011",
    :ssh_vm => "22",
    :ram => 1024,
    :core => 1
  }
]
Vagrant.configure(2) do |config|
  config.vm.synced_folder ".", "/vagrant", disabled: false
  servers.each do |machine|
    config.vm.define machine[:hostname] do |node|
      node.vm.box = ISO
      node.vm.hostname = machine[:hostname]
      node.vm.network "private_network", ip: machine[:ip]
      node.vm.network :forwarded_port, guest: machine[:ssh_vm], host: machine[:ssh_host]
      node.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--memory", machine[:ram]]
        vb.customize ["modifyvm", :id, "--cpus", machine[:core]]
        vb.name = machine[:hostname]
      end
      node.vm.provision "ansible" do |setup|
       setup.inventory_path = INVENTORY_PATH
       setup.playbook = "./provision.yml"
       setup.become = true
       setup.extra_vars = { ansible_user: 'vagrant' }
      end
    end
  end
end
```
Результат выполнения 'vagrant up'
```
$ vagrant up
Bringing machine 'server1.netology' up with 'virtualbox' provider...
==> server1.netology: Importing base box 'bento/ubuntu-20.04'...
==> server1.netology: Matching MAC address for NAT networking...
==> server1.netology: Checking if box 'bento/ubuntu-20.04' version '202112.19.0' is up to date...
==> server1.netology: Setting the name of the VM: server1.netology
==> server1.netology: Clearing any previously set network interfaces...
==> server1.netology: Preparing network interfaces based on configuration...
    server1.netology: Adapter 1: nat
    server1.netology: Adapter 2: hostonly
==> server1.netology: Forwarding ports...
    server1.netology: 22 (guest) => 20011 (host) (adapter 1)
    server1.netology: 22 (guest) => 2222 (host) (adapter 1)
==> server1.netology: Running 'pre-boot' VM customizations...
==> server1.netology: Booting VM...
==> server1.netology: Waiting for machine to boot. This may take a few minutes...
    server1.netology: SSH address: 127.0.0.1:2222
    server1.netology: SSH username: vagrant
    server1.netology: SSH auth method: private key
    server1.netology: Warning: Connection reset. Retrying...
    server1.netology: 
    server1.netology: Vagrant insecure key detected. Vagrant will automatically replace
    server1.netology: this with a newly generated keypair for better security.
    server1.netology: 
    server1.netology: Inserting generated public key within guest...
    server1.netology: Removing insecure key from the guest if it's present...
    server1.netology: Key inserted! Disconnecting and reconnecting using new SSH key...
==> server1.netology: Machine booted and ready!
==> server1.netology: Checking for guest additions in VM...
==> server1.netology: Setting hostname...
==> server1.netology: Configuring and enabling network interfaces...
==> server1.netology: Mounting shared folders...
    server1.netology: /vagrant => /home/dmitrii/homework
==> server1.netology: Running provisioner: ansible...
Vagrant has automatically selected the compatibility mode '2.0'
according to the Ansible version installed (2.9.6).

Alternatively, the compatibility mode can be specified in your Vagrantfile:
https://www.vagrantup.com/docs/provisioning/ansible_common.html#compatibility_mode

    server1.netology: Running ansible-playbook...

PLAY [nodes] *******************************************************************

TASK [Gathering Facts] *********************************************************
ok: [server1.netology]

TASK [Create directory for ssh-keys] *******************************************
ok: [server1.netology]

TASK [Adding rsa-key in /root/.ssh/authorized_keys] ****************************
An exception occurred during task execution. To see the full traceback, use -vvv. The error was: If you are using a module and expect the file to exist on the remote, see the remote_src option
fatal: [server1.netology]: FAILED! => {"changed": false, "msg": "Could not find or access '~/.ssh/id_rsa.pub' on the Ansible Controller.\nIf you are using a module and expect the file to exist on the remote, see the remote_src option"}
...ignoring

TASK [Checking DNS] ************************************************************
changed: [server1.netology]

TASK [Installing tools] ********************************************************
ok: [server1.netology] => (item=['git', 'curl'])

TASK [Installing docker] *******************************************************
changed: [server1.netology]

TASK [Add the current user to docker group] ************************************
changed: [server1.netology]

PLAY RECAP *********************************************************************
server1.netology           : ok=7    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=1  
```
Результат 'docker ps'
```
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

