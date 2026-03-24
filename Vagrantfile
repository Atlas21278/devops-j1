Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"

  # --- VM DEV ---
  config.vm.define "app-dev" do |dev|
    dev.vm.hostname = "app-dev"
    dev.vm.network "private_network", ip: "192.168.56.10", virtualbox__hostonly: "VirtualBox Host-Only Ethernet Adapter"
    dev.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
    end

    dev.vm.provision "shell", inline: <<-SHELL
      set -e
      apt-get update -y
      apt-get install -y sudo

      id -u admin >/dev/null 2>&1 || useradd -m -s /bin/bash admin
      echo "admin ALL=(ALL) NOPASSWD:ALL" >/etc/sudoers.d/admin

      mkdir -p /home/admin/.ssh
      chmod 700 /home/admin/.ssh

      cat <<'EOF' >/home/admin/.ssh/authorized_keys
#{File.read("id-rda-infra.key.pub")}
EOF

      chmod 600 /home/admin/.ssh/authorized_keys
      chown -R admin:admin /home/admin/.ssh
    SHELL
  end

  # --- VM PROD ---
  config.vm.define "app-prod1" do |prod|
    prod.vm.hostname = "app-prod1"
    prod.vm.network "private_network", ip: "192.168.56.11", virtualbox__hostonly: "VirtualBox Host-Only Ethernet Adapter"
    prod.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
    end

    prod.vm.provision "shell", inline: <<-SHELL
      set -e
      apt-get update -y
      apt-get install -y sudo

      id -u admin >/dev/null 2>&1 || useradd -m -s /bin/bash admin
      echo "admin ALL=(ALL) NOPASSWD:ALL" >/etc/sudoers.d/admin

      mkdir -p /home/admin/.ssh
      chmod 700 /home/admin/.ssh

      cat <<'EOF' >/home/admin/.ssh/authorized_keys
#{File.read("id-rda-infra.key.pub")}
EOF

      chmod 600 /home/admin/.ssh/authorized_keys
      chown -R admin:admin /home/admin/.ssh
    SHELL
  end

  # --- VM Jenkins ---
  config.vm.define "jenkins" do |jk|
    jk.vm.hostname = "jenkins"
    jk.vm.network "private_network", ip: "192.168.56.12", virtualbox__hostonly: "VirtualBox Host-Only Ethernet Adapter"
    jk.vm.network "forwarded_port", guest: 8080, host: 8080
    jk.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
      vb.cpus = 2
    end

    # Copie de la clé SSH privée vers la VM
    jk.vm.provision "file", source: "id-rda-infra.key", destination: "/tmp/id-rda-infra.key"

    jk.vm.provision "shell", inline: <<-SHELL
      set -e
      apt-get update -y
      apt-get install -y sudo curl gnupg2 git python3 python3-pip ansible

      # Java (requis par Jenkins)
      apt-get install -y default-jdk

      # Repo Jenkins stable
      curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key \
        | gpg --dearmor -o /usr/share/keyrings/jenkins-keyring.gpg
      echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.gpg] https://pkg.jenkins.io/debian-stable binary/" \
        > /etc/apt/sources.list.d/jenkins.list
      apt-get update -y
      apt-get install -y jenkins

      # pipenv pour le security check
      pip3 install pipenv --break-system-packages

      # Clé SSH pour que Jenkins puisse se connecter aux VMs app
      mkdir -p /var/lib/jenkins/.ssh
      cp /tmp/id-rda-infra.key /var/lib/jenkins/.ssh/id_rsa
      chmod 600 /var/lib/jenkins/.ssh/id_rsa
      echo "StrictHostKeyChecking no" > /var/lib/jenkins/.ssh/config
      chown -R jenkins:jenkins /var/lib/jenkins/.ssh

      # User admin pour SSH depuis le host
      id -u admin >/dev/null 2>&1 || useradd -m -s /bin/bash admin
      echo "admin ALL=(ALL) NOPASSWD:ALL" >/etc/sudoers.d/admin
      mkdir -p /home/admin/.ssh
      chmod 700 /home/admin/.ssh

      cat <<'EOF' >/home/admin/.ssh/authorized_keys
#{File.read("id-rda-infra.key.pub")}
EOF

      chmod 600 /home/admin/.ssh/authorized_keys
      chown -R admin:admin /home/admin/.ssh

      systemctl enable jenkins
      systemctl start jenkins

      echo "=== Jenkins initial admin password ==="
      sleep 10
      cat /var/lib/jenkins/secrets/initialAdminPassword 2>/dev/null || echo "Not ready yet - check: sudo cat /var/lib/jenkins/secrets/initialAdminPassword"
    SHELL
  end
end
