# Создаем ВМ с помощью vagrant, на которую учтанавливаем ansible
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.boot_timeout = 600

  config.vm.define "ansible01" do |ansible01|
    ansible01.vm.hostname = "ansible01"
    ansible01.vm.network "private_network", ip: "192.168.56.11"
    
    # Стандартные настройки SSH
    ansible01.ssh.insert_key = true
    ansible01.ssh.forward_agent = false
    
    ansible01.vm.synced_folder ".", "/vagrant", disabled: true
  end

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = 2024
    vb.cpus = 2
    vb.name = "ansible01"
  end
end

# На развернутую ВМ устанавливаем ainsible
sudo apt update
sudo apt install software-properties-common
# Добавляем Ansible PPA в source list
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
vagrant@ansible01:~$ ansible --version
ansible [core 2.17.14]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/vagrant/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/vagrant/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.10.12 (main, Mar  3 2026, 11:56:32) [GCC 11.4.0] (/usr/bin/python3)
  jinja version = 3.0.3
  libyaml = True
  # Создаем ВМ с помощью vagrant, которая будет управляться ansible
  Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.boot_timeout = 600

  config.vm.define "ansible01" do |ansible01|
    ansible01.vm.hostname = "ansible-client"
    ansible01.vm.network "private_network", ip: "192.168.56.12"
    
    # Стандартные настройки SSH
    ansible01.ssh.insert_key = true
    ansible01.ssh.forward_agent = false
    
    ansible01.vm.synced_folder ".", "/vagrant", disabled: true
  end

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = 2024
    vb.cpus = 2
    vb.name = "ansible-client"
  end
end
# Для подключения на хосте ansible создадим SSH ключ и скопируем на клиента
vagrant@ansible01:~$ ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
Generating public/private rsa key pair.
Your identification has been saved in /home/vagrant/.ssh/id_rsa
Your public key has been saved in /home/vagrant/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:FCrqH4Lqnvgpj5eCNmxKvR8bghD3yJhkemOa1dlLv9s vagrant@ansible01
The key's randomart image is:
+---[RSA 4096]----+
|        .        |
|       . .       |
|.o. . . .        |
|+* = + .         |
|= O + o S        |
|.Oo. . o         |
|*oo+.o. .        |
|B*++o.+  o       |
|@X=.oo  o.E      |
+----[SHA256]-----+
vagrant@ansible01:~$ ssh-copy-id vagrant@192.168.56.12
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/vagrant/.ssh/id_rsa.pub"
The authenticity of host '192.168.56.12 (192.168.56.12)' can't be established.
ED25519 key fingerprint is SHA256:IA5iFwYOgZ9zxLBstc6M2IDkm/bFIGhJfrPcrpzbOgc.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
(vagrant@192.168.56.12) Password:
(vagrant@192.168.56.12) Password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'vagrant@192.168.56.12'"
and check to make sure that only the key(s) you wanted were added.
# Проверим подключение к клиенту
vagrant@ansible01:~$ ssh vagrant@192.168.56.12 'echo "Connected!"'
Connected!
# Создадим inventory.ini
vagrant@ansible01:~$ mkdir inventory
vagrant@ansible01:~/inventory$ nano inventory.ini

[webservers]
web1 ansible_host=192.168.56.12 ansible_user=vagrant

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=~/.ssh/id_rsa

# Проверим подключение
vagrant@ansible01:~/inventory$ ansible -i inventory.ini all -m ping
web1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
# В текущем каталоге создадим файл ansible.cfg
vagrant@ansible01:~/inventory$ nano ansible.cfg
vagrant@ansible01:~/inventory$ cat ansible.cfg
[defaults]
inventory = inventory.ini
host_key_checking = False
timeout = 30
remote_user = vagrant
private_key_file = ~/.ssh/id_rsa
# Изменим файл inventory.ini
vagrant@ansible01:~/inventory$ cat inventory.ini
[webservers]
192.168.56.12
# Проверим, что работает
 ansible 192.168.56.12 -m command -a uptime
[WARNING]: Platform linux on host 192.168.56.12 is using the discovered Python
interpreter at /usr/bin/python3.10, but future installation of another Python
interpreter could change the meaning of that path. See
https://docs.ansible.com/ansible-
core/2.17/reference_appendices/interpreter_discovery.html for more information.
192.168.56.12 | CHANGED | rc=0 >>
 20:00:26 up  5:02,  3 users,  load average: 0.00, 0.00, 0.00

