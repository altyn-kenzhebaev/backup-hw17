# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']

MACHINES = {
    :backup => {
        :box_name => "almalinux/9",
        :cpus => 2,
        :memory => 1024,
        :ip_addr => '192.168.50.10',
        :playbook => "ansible/backup.yml",
        :disks => {
          :sata1 => {
              :dfile => home + '/VirtualBox VMs/backup/sata1.vdi',
              :size => 2048,
              :port => 1
          },
        }
    },
    :client => {
        :box_name => "almalinux/9",
        :cpus => 2,
        :memory => 1024,
        :ip_addr => '192.168.50.15',
        :playbook => "ansible/client.yml",
    },
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
      config.vm.synced_folder ".", "/vagrant", disabled: true
      config.vm.define boxname do |box|
          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s
          box.vm.network "private_network", ip: boxconfig[:ip_addr]
          box.vm.provider :virtualbox do |vb|
            vb.name = boxname.to_s
            vb.memory = boxconfig[:memory]
            vb.cpus = boxconfig[:cpus]
            if vb.name == "backup"
              boxconfig[:disks].each do |dname, dconf|
                unless File.exist?(dconf[:dfile])
                  vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                end
                vb.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
              end
            end
          end
          box.vm.provision "ansible" do |ansible|
            ansible.playbook = boxconfig[:playbook]
          end
      end
  end
end