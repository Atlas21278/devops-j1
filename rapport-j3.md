# Rapport J3 - Résilience, Observabilité et Déploiement Avancé

## Partie 1 - Infrastructure Redondante et Load-Balancée

---

### Exercice 1 - Ajout d'une VM de production (app-prod2)

**Ajout dans le Vagrantfile :**

```ruby
config.vm.define "app-prod2" do |prod2|
  prod2.vm.hostname = "app-prod2"
  prod2.vm.network "private_network", ip: "192.168.56.13"
  prod2.vm.provider "virtualbox" do |vb|
    vb.memory = 1024
    vb.cpus = 1
  end
  # ... provisioning admin user + SSH key
end
```

**Lancement de la VM :**

```bash
vagrant up app-prod2
```

**Mise à jour de inventory-prod.yaml :**

```yaml
all:
  children:
    my_infra:
      hosts:
        app-dev:
          ansible_host: 192.168.56.10
        app-prod1:
          ansible_host: 192.168.56.11
        app-prod2:
          ansible_host: 192.168.56.13
      vars:
        ansible_user: admin
        ansible_ssh_private_key_file: /var/lib/jenkins/.ssh/id_rsa
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
```

**Test de connexion SSH :**

```bash
vagrant ssh app-prod2 -c "hostname && ip addr show eth1 | grep 'inet '"
```

```
app-prod2
inet 192.168.56.13/24 brd 192.168.56.255 scope global eth1
```

![vagrant status montrant les 4 VMs running](screenshot-ex1-vagrant-status.png)

---

### Exercice 2 - Load Balancer HAProxy

**Ajout de lb-prod dans le Vagrantfile :**

```ruby
config.vm.define "lb-prod" do |lb|
  lb.vm.hostname = "lb-prod"
  lb.vm.network "private_network", ip: "192.168.56.14"
  lb.vm.provider "virtualbox" do |vb|
    vb.memory = 512
    vb.cpus = 1
  end
  # HAProxy installé au provisioning
  lb.vm.provision "shell", inline: "apt-get install -y haproxy"
end
```

**Lancement du load balancer :**

```bash
vagrant up lb-prod
```

**Configuration HAProxy (`haproxy.cfg`) :**

```
frontend todolist_front
    bind *:80
    default_backend todolist_back

backend todolist_back
    balance roundrobin
    option tcp-check
    server app-prod1 192.168.56.11:8000 check
    server app-prod2 192.168.56.13:8000 check

frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
```

**Déploiement de la config HAProxy :**

```bash
vagrant upload haproxy.cfg /tmp/haproxy.cfg lb-prod
vagrant ssh lb-prod -c "sudo cp /tmp/haproxy.cfg /etc/haproxy/haproxy.cfg && sudo systemctl restart haproxy"
```

**Test de résilience — arrêt de app-prod1 :**

```bash
vagrant ssh app-prod1 -c "sudo systemctl stop todolist"
# L'application reste accessible via le load balancer (app-prod2 prend le relais)
```

**Remise en service de app-prod1 :**

```bash
vagrant ssh app-prod1 -c "sudo systemctl start todolist"
```

![Application accessible via lb-prod](screenshot-ex2-haproxy-app.png)

![HAProxy stats — app-prod1 DOWN en rouge, app-prod2 UP](screenshot-ex2-haproxy-stats-prod1-down.png)

---

### Exercice 4 - Rolling Update Ansible

**Principe :** mise à jour des serveurs un par un (`serial: 1`) pour maintenir le service disponible pendant le déploiement. HAProxy continue d'envoyer le trafic vers le serveur non mis à jour pendant que l'autre redémarre.

**Playbook `deploy_rolling.yml` :**

```yaml
- name: Rolling update todolist (1 server at a time)
  hosts: my_infra
  become: true
  gather_facts: false
  serial: 1

  tasks:
    - name: Check app is responding before update
      ansible.builtin.uri:
        url: "http://{{ app_ip }}:8000/"
        status_code: [200, 302]
      delegate_to: localhost
      become: false

    - name: Pull latest code from GitHub
      ansible.builtin.git:
        repo: "{{ repo_url }}"
        dest: "{{ app_dir }}"
        version: "{{ repo_branch }}"
        force: true
      become_user: admin

    - name: Set ALLOWED_HOSTS in settings.py
      ansible.builtin.lineinfile:
        path: "{{ app_dir }}/todo/settings.py"
        regexp: '^ALLOWED_HOSTS\s*='
        line: "ALLOWED_HOSTS = ['localhost','127.0.0.1','{{ app_ip }}','192.168.56.14']"

    - name: Run database migrations
      ansible.builtin.command: "{{ venv_dir }}/bin/python manage.py migrate"
      args:
        chdir: "{{ app_dir }}"

    - name: Restart todolist service
      ansible.builtin.systemd:
        name: todolist
        state: restarted

    - name: Wait for app to be back up after update
      ansible.builtin.uri:
        url: "http://{{ app_ip }}:8000/"
        status_code: [200, 302]
      delegate_to: localhost
      become: false
      retries: 10
      delay: 3
```

**Lancement du rolling update :**

```bash
ansible-playbook -i inventory-prod.yaml deploy_rolling.yml \
  --limit "app-prod1,app-prod2" \
  -e "ansible_ssh_private_key_file=/home/vagrant/.ssh/id_rsa"
```

**Résultat :**

```
PLAY RECAP
app-prod1 : ok=7  changed=5  unreachable=0  failed=0
app-prod2 : ok=7  changed=5  unreachable=0  failed=0
```

app-prod1 est mis à jour en premier (vérifié avant et après), puis app-prod2 — le service reste disponible via HAProxy tout au long.

![Rolling update — app-prod1 puis app-prod2](screenshot-ex4-rolling-update.png)

---
