# -*- mode: ruby -*-
# vi: set ft=ruby :

# Para aprovechar este Vagrantfile necesita Vagrant y Virtualbox instalados:
#
#   * Virtualbox
#
#   * Vagrant
#
#   * Plugins de Vagrant:
#       + vagrant-proxyconf y su configuracion si requiere de un Proxy para salir a Internet
#       + vagrant-cachier
#       + vagrant-disksize
#       + vagrant-hostmanager
#       + vagrant-share
#       + vagrant-vbguest

VAGRANTFILE_API_VERSION = "2"

HOSTNAME = "devopsws"
IP_ADDRESS="192.168.56.11"
IP_NETMASK_BITS="21"
IP_NETWORK="192.168.56.0"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = true

    # uso cachier con NFS solamente si el hostmanager gestiona los nombres en /etc/hosts del host
    if Vagrant.has_plugin?("vagrant-cachier")

      # W: Download is performed unsandboxed as root as file '/var/cache/apt/archives/partial/xyz' couldn't be accessed by user '_apt'. - pkgAcquire::Run (13: Permission denied)

      config.cache.synced_folder_opts = {
        owner: "_apt"
      }
      # Configure cached packages to be shared between instances of the same base box.
      # More info on http://fgrehm.viewdocs.io/vagrant-cachier/usage
      config.cache.scope = :box

      # OPTIONAL: If you are using VirtualBox, you might want to use that to enable
      # NFS for shared folders. This is also very useful for vagrant-libvirt if you
      # want bi-directional sync

      # NOTA: con encrypted HOME, no funciona el NFS, si no es tu caso, descomenta los parametros siguientes:
      #config.cache.synced_folder_opts = {
      #  type: :nfs,
      #  # The nolock option can be useful for an NFSv3 client that wants to avoid the
      #  # NLM sideband protocol. Without this option, apt-get might hang if it tries
      #  # to lock files needed for /var/cache/* operations. All of this can be avoided
      #  # by using NFSv4 everywhere. Please note that the tcp option is not the default.
      #  mount_options: ['rw', 'vers=3', 'tcp', 'nolock', 'fsc' , 'actimeo=2']
      #}

      # For more information please check http://docs.vagrantup.com/v2/synced-folders/basic_usage.html
   end

  end

 config.vm.define HOSTNAME do |srv|


    srv.vm.box = "ubuntu/bionic64"
    #srv.vm.box = "ubuntu/focal64"
    #srv.vm.box = "ubuntu/jammy64"
    #srv.vm.box = "debian/bullseye64" # requiere restructuracion de fuentes APT, nombres de paquetes
    srv.disksize.size = '20GB'


    srv.vm.network "private_network", ip: IP_ADDRESS
    srv.vm.boot_timeout = 3600
    srv.vm.box_check_update = false
    srv.ssh.forward_agent = true
    srv.ssh.forward_x11 = true
    srv.vm.hostname = HOSTNAME

    #srv.vm.synced_folder "www/", "/var/www/html"

    if Vagrant.has_plugin?("vagrant-hostmanager")
      srv.hostmanager.aliases = %W(#{HOSTNAME+".dominio.local.tld'"} #{HOSTNAME} )
    end

    srv.vm.provider :virtualbox do |vb|
      vb.gui = false
      vb.cpus = 2
      vb.memory = "1536"

      # si no existe nested virt, instala virtualbox-6.0 en guest, y no instala VMware Workstation
      vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]


      # https://www.virtualbox.org/manual/ch08.html#vboxmanage-modifyvm mas parametros para personalizar en VB
    end
  end

    ##
    # Aprovisionamiento
    #
    config.vm.provision "fix-no-tty", type: "shell" do |s|
        s.privileged = false
        s.inline = "sudo sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile"
    end
    config.vm.provision "actualiza", type: "shell" do |s|  # http://foo-o-rama.com/vagrant--stdin-is-not-a-tty--fix.html
        s.privileged = false
        s.inline = <<-SHELL
          export DEBIAN_FRONTEND=noninteractive
          export APT_LISTCHANGES_FRONTEND=none
          export APT_OPTIONS=' -y --allow-downgrades --allow-remove-essential --allow-change-held-packages -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold '

          sudo apt-get update -y -qq
          echo 'libc6 libraries/restart-without-asking boolean true' | sudo debconf-set-selections
          sudo dpkg-reconfigure libc6
          sudo -E apt-get -q --option "Dpkg::Options::=--force-confold" --assume-yes install libssl1.1 # https://bugs.launchpad.net/ubuntu/+source/openssl/+bug/1832919a

          sudo apt-get upgrade ${APT_OPTIONS}
          sudo apt-get full-upgrade ${APT_OPTIONS}
          sudo apt-get autoremove -y

        SHELL
    end
    config.vm.provision "ssh_pub_key", type: :shell do |s|
      begin
          ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
          s.inline = <<-SHELL
            mkdir -p /root/.ssh/
            touch /root/.ssh/authorized_keys
            echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
            echo #{ssh_pub_key} >> /root/.ssh/authorized_keys
          SHELL
      rescue
          puts "No hay claves publicas en el HOME de su pc"
          s.inline = "echo OK sin claves publicas"
      end
    end

    proxy_host_port = ENV['all_proxy'] || ENV['http_proxy']  || ""
    proxy_host_port = if proxy_host_port.empty? then "" else proxy_host_port.scan(/\/\/([0-9\.]*):/)[0][0]+':'+proxy_host_port.scan(/:([0-9]*)$/)[0][0] end

    config.vm.provision "instala-pip-ansible", type: "shell" do |s| 
        s.privileged = false
        s.inline = <<-SHELL
          export DEBIAN_FRONTEND=noninteractive
          export APT_LISTCHANGES_FRONTEND=none
          export APT_OPTIONS=' -y --allow-downgrades --allow-remove-essential --allow-change-held-packages -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold '

          sudo apt-get install python3-pip ${APT_OPTIONS}
          sudo -H python3 -m pip install pip setuptools wheel
          sudo -H python3 -m pip install ansible

        SHELL
    end
    config.vm.provision "ansible-provision", type: :ansible do |ansible|
      ansible.playbook = "site.yml"
      ansible.config_file = "./vagrant-inventory/ansible.cfg"
      ansible.inventory_path = "./vagrant-inventory/"
      ansible.verbose= "-vv"
      #ansible.tags= [ "tinyproxy" ]
      #ansible.skip_tags= [ "ansible" ]
      ansible.become = false
      # heredo la configuracion de Proxy del entorno del host Vagrant:
      ansible.extra_vars = {
        organizacion: "Mi organizacion",
        all_proxy:   ENV['all_proxy']   || ENV['http_proxy']  || "",
        http_proxy:  ENV['http_proxy']  || "",
        https_proxy: ENV['https_proxy'] || "",
        ftp_proxy:   ENV['ftp_proxy']   || "",
        no_proxy:    ENV['no_proxy']    || "",
        tinyproxy_listen_ip: IP_ADDRESS,
        tinyproxy_default_upstream: "#{proxy_host_port}",
        tinyproxy_allow: [ 
                           IP_ADDRESS + '/' + IP_NETMASK_BITS, 
                           IP_NETWORK + '/' + IP_NETMASK_BITS, 
                           '192.168.20.0/22' 
                         ]
      }
    end

end
