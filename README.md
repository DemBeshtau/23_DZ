# DNS. Настройка и обслуживание #
1. На предложенном стенде https://github.com/erlong15/vagrant-bind:
   - добавить сервер client2;
   - завести в зоне   dns.lab имена:
     - web1 - смотрит на client1;
     - web2 - смотрит на client2;
   - завести зону newdns.lab;
   - завести в новой зоне запись:
     - www - смотрит на обоих клиентов;
2. Настроить split-dns:
   - client1 - видит обе зоны, но в зоне dns.lab только web1;
   - client2 - видит только dns.lab;
3. Дополнительное задание:
   - произвести настройки без выключения selinux.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.18, Ansible 9.4.0 и образа CentOS 7 версии 1804_2. <br/>
### Ход решения ###
1. Установка необходимого программ
