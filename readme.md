-----------------

## Install and Configure Kubernetes Master Node using Ansible playbook


"In the digital era, Kubernetes via Ansible playbooks streamlines containerized application management, enhancing scalability and reliability." - ChatGPT<br>

The Prerequisites that we will be using are:
1. OS used: Ubuntu 20.04 LTS
2. Recommended RAM: 4 GB

---

Specifications we will be working on
1. Kubeadm version: 1.2
2. Calico Version: 3.27.1
3. Calico plugin link: https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
4. Runtime: Containerd
5. Remote User of the remote host: devuser

---

Before we begin let us go through the per-requisite of installing ansible in our system. Remember we need to have ssh-keys configured on the managed nodes [our server which will serve as master node for kubernetes].
```
#In case of Ubuntu etc.
sudo apt-get update && sudo apt-get upgrade
sudo apt install ansible

#In case of Fedora/RHEL/CentOS
sudo dnf install ansible
```
Also you will need python install in the managed hosts, which I assume will already be present in any basic Linux installation. Just in case you need the ssh-keys configuration command, here are the commands.
```
#To generate the ssh keys
ssh-keygen

#To copy the ssh id's to remote machine
ssh-copy-id username@192.168.1.1
```
After this, we are good to start with ansible playbook.

---

Open a file named master.yml inside any directory, let's say Ansible
We will declare the variables that we will be using in the playbook at various places.
```
vars:
  - cni_plugin_version: https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
  - regular_user: devuser
  - short_version: v1.27
  - version: 1.27.6-00 #not used yet, maybe used in later revisions
  - calico_version: v3.27.1
```
Next we move on to create the tasks.<br>

First things first, we need to turn on the basic firewall ports that are required for the kubelet service and kuberenetes API server to interact with other nodes. Now mostly we have ufw and firewalld being used, so we will check the firewall service present and using when condition, configure the appropriate service.
Note: To manage firewalld we could have used firewalld module as well, and that's perfectly fine. Go ahead and tweak accordingly.
```
- name: Firewall configurations
  block:
    - name: Check if firewalld is installed
      stat:
        path: /usr/bin/firewall-cmd
      register: firewalld_installed
      ignore_errors: yes

    - name: Check if ufw is installed
      stat:
        path: /usr/sbin/ufw
      register: ufw_installed
      ignore_errors: yes

    - name: Open ports for firewalld
      when: firewalld_installed.stat.exists
      block:
        - name: Open port 6443 for API Server
          shell: "firewall-cmd --add-port=6443/tcp --permanent"
          ignore_errors: yes

        - name: Open port 2379 for Etcd client communication
          shell: "firewall-cmd --add-port=2379/tcp --permanent"
          ignore_errors: yes

        - name: Open port 2380 for Etcd server-to-server communication
          shell: "firewall-cmd --add-port=2380/tcp --permanent"
          ignore_errors: yes

        - name: Open port 10250 for Kubelet
          shell: "firewall-cmd --add-port=10250/tcp --permanent"
          ignore_errors: yes
        
        - name: Reloading the firewalld
          shell: "firewall-cmd --reload"     
          ignore_errors: yes           

    - name: Open ports for ufw
      when: ufw_installed.stat.exists
      block:
        - name: Open port 6443 for API Server
          shell: "ufw allow 6443/tcp"
          ignore_errors: yes

        - name: Open port 2379 for Etcd client communication
          shell: "ufw allow 2379/tcp"
          ignore_errors: yes

        - name: Open port 2380 for Etcd server-to-server communication
          shell: "ufw allow 2380/tcp"
          ignore_errors: yes

        - name: Open port 10250 for Kubelet
          shell: "ufw allow 10250/tcp"
          ignore_errors: yes
```
Just in case we want no hassles in later stage while we do the repo configs, we wanted to do a small apt update

```
- name: Update and Upgrade the apt packages before proceeding
  block:
    - name: performing the Update
      ansible.builtin.apt:
        update_cache: true
```
Then we will turn off the swap spaces. Now swap spaces can be persistently mounted as well in the /etc/fstab file, so only turning them off using swapoff -a is not going to work in case the node reboots. So we will comment out the entry in the /etc/fstab file as well.
```
- name: Block to manage Swap space
# I have used a block here, just for my ease of navigating through the file, you can have these tasks out of the block also
  block:
    - name: Check for swap to be on 
      shell: "swapon -s | wc -l"
      register: number_of_swap
      changed_when: false
    
    - name: Turning off swaps
      shell: "swapoff -a"
      when: number_of_swap.stdout != 0
    
    - name: Check for swap entry in /etc/fstab
      shell: "sudo cat /etc/fstab | awk '{print $3}' | grep swap | wc -l"
      register: number_in_fstab
      changed_when: false
    
    - name: Commenting entries in fstab
      lineinfile:
        path: /etc/fstab
        regexp: '.*swap.*'
        line: '# \g<0>'
      when: number_in_fstab.stdout != '0'
```
Next we will rename our host as master and for that we need to change the hostname as well as add an entry in /etc/hosts file to resolve localhost.
```
- name: Change the hostname 
  hostname:
    name: master
    
- name: Making neccesary changes in the etc hosts file
  lineinfile:
    path: /etc/hosts
    line: "127.0.0.1 master"
    state: present
```
Now we will configure the bridge networking and ip forwarding that will be required for kubelet and kubeadm.
```
- name: Configure the sysctl for kubernetes
  shell: "{{ item }}"
  loop:
    - "echo 'net.bridge.bridge-nf-call-ip6tables = 1' | tee /etc/sysctl.d/k8s.conf"
    - "echo 'net.bridge.bridge-nf-call-iptables = 1' | tee -a /etc/sysctl.d/k8s.conf"
    - "sysctl --system"
  ignore_errors: yes

- name: Check if br_netfilter is set
  shell: "lsmod | grep br_net | wc -l"
  register: lsmod_results

- name: Load br_netfilter module
  shell: "{{ item }}"
  loop:
    - "echo 'br_netfilter' | tee /etc/modules-load.d/k8s.conf"
    - "modprobe br_netfilter"
  ignore_errors: yes
  when: lsmod_results.stdout != '2'

- name: Checking for ip_forward configurations
  shell: "cat /proc/sys/net/ipv4/ip_forward"
  register: ip_forward_result
  changed_when: false

- name: Forward IP persistently if required
# the below blocks has all tasks for ip forwarding both runtime and persistent
  block:
    - name: Forwarding the IP in sysctl
      lineinfile:
        path: /etc/sysctl.conf
        line: "net.ipv4.ip_forward=1"
        state: present
      register: forwarded 
    
    - name: Change the ip_forward for runtime
      shell: "sysctl -w net.ipv4.ip_forward=1"
      
    - name: Reloading the sysctl config
      shell: "sysctl -p /etc/sysctl.conf"
      when: forwarded is changed 
  when: ip_forward_result.stdout != '1'
```
To make sure our services don't conflict with docker [ Note: In my personal experience, docker was installed in few machines while I was running the playbook and conflict arose with containerd ], we will remove docker completely.
```
- name: Check and Remove docker if previous versions exist 
  apt:
    name:
      - docker.io
    state: absent
```
Once we are through with these many steps, let us now install containerd runtime and also configure it. We need to add the apt repository and the key.
```
- name: Add the apt repository of containerd and Install containerd runtime
  block:
    - name: Check if keyring is already present
      stat:
        path: /usr/share/keyrings/docker.gpg
      register: containerd_keyring_present

#install the new key only if key-file is missing
    - name: Adding the Containerd repo key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        keyring: /usr/share/keyrings/docker.gpg 
      when: containerd_keyring_present.stat.exists != 'true'

#we will fetch the dpkg architecture required for the repo listing
    - name: Retreive dpkg architecture 
      shell: "echo $(dpkg --print-architecture)"
      register: dpkg_architecture
      changed_when: false
      ignore_errors: true

    - name: The containerd repo
      shell: "echo 'deb [arch={{ dpkg_architecture.stdout }} signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable' | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null"
      changed_when: false
      ignore_errors: true

    - name: Installing containerd runtime
      apt:
        name: containerd.io
        state: present
```
Now containerd after installation does not generate the containerd.toml file on every machine and to avoid that, we will generate the default config and then make the SystemdCgroup = true. We will also download and install the CNI plugin in the /opt/cni/bin folder.
```
- name: Setting up the containerd runtime configurations
  block:
    - name: Making sure the unarchive directory exists 
      file: 
        path: "/opt/cni/bin/"
        state: directory

    - name: Download and unarchive the cni plugin tar file  
      unarchive:
        src: "{{ cni_plugin_version }}"
        dest: "/opt/cni/bin/"
        remote_src: yes
    
    - name: Make the directory
      file:
        path: /etc/containerd
        state: directory

    - name: Generate CNI configurations
      shell: "containerd config default > /etc/containerd/config.toml"
    
    - name: Change the systemd driver of containerd
      replace:
        path: /etc/containerd/config.toml
        regexp: 'SystemdCgroup = false'
        replace: 'SystemdCgroup = true'
```
Once this is done, we need to make sure that our user has the permission to run containerd commands. For that reason we need to add a group, let's name it containerd and add our user to that group. Then in the config.toml we need to make sure that the sock file that the service generated has containerd as the group.
```
- name: Changing the user permissions to access container socket
  block:
    - name: Setting up a group
      group:
        name: containerd
        state: present
    
    - name: Retrieve the group id
      shell: "cat /etc/group | grep containerd | awk -F ':' '{print $3}'"
      register: containerd_group_id

    - name: Adding the user to the group
      user: 
        name: devuser
        groups: containerd
        append: yes
    
    - name: Add the group id in the config file 
      replace:
        path: /etc/containerd/config.toml
        after: '[grpc]\n*address = "/run/containerd/containerd.sock"'
        regexp: gid = 0
        replace: 'gid = {{ containerd_group_id.stdout }}'

    - name: Changing permission of the socket to the group
      file: 
        path: "/run/containerd/containerd.sock"
        group: containerd
```
Now let's reload the containerd service and also install the dependent packages for kubernetes
```
- name: Reloading the containerd service
  service:
    name: containerd
    state: restarted
    enabled: true

- name: Install dependent packages
  apt:
    name: 
      - apt-transport-https 
      - ca-certificates 
      - curl 
      - conntrack
    state: present
```
Now coming down to our main part, that is configuring apt repo for kubernetes and also installing the kubeadm, kubelet and kubectl.
```
- name: Add the Apt repository /usr/share/keyrings/kubernetes-archive-keyring.gpg
  block:
    - name: Check if the keyring file is already present
      stat:
        path: /usr/share/keyrings/kubernetes-archive-keyring.gpg
      register: gpg_key_present
    

    #the below task execute only if keyring file is absent
    - name: Add apt key
      apt_key:
        url: https://pkgs.k8s.io/core:/stable:/{{ short_version }}/deb/Release.key
        keyring: /usr/share/keyrings/kubernetes-archive-keyring.gpg
      when: gpg_key_present.stat.exists != 'true'

- name: Add the Kubernetes Repository
  apt_repository:
    repo: deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/{{ short_version }}/deb/ /
    state: present
    filename: kubernetes

- name: Perform an apt update
  apt:
    update_cache: yes

- name: Install the main kubeadm, kubelet, kubectl
  apt:
    name: 
      - 'kubelet'
      - 'kubeadm'
      - 'kubectl'
    state: present
  ignore_errors: true
```
Now we will make sure the containerd and kubelet service is not masked. If they are masked, we will unmask them. We will make sure that containerd is up and running whereas kubelet service is stopped. [kubeadm init will start the kubelet automatically].
```
- name: Check if the containerd service is unmasked
  shell: "systemctl is-enabled containerd"
  register: masked_containerd
  changed_when: false

- name: Unmask the docker, if masked 
  shell: "systemctl unmask containerd"
  when: masked_containerd.stdout == 'masked'

- name: Check if the kubelet service is unmasked
  shell: "systemctl is-enabled kubelet"
  register: masked_kubelet
  changed_when: false

- name: Unmask the docker, if masked 
  shell: "systemctl unmask kubelet"
  when: masked_kubelet.stdout == 'masked'  

- name: Making sure kubelet service is stopped
  service:
    name: kubelet
    state: stopped
    enabled: true

- name: Making sure containerd service is started
  service:
    name: containerd
    state: started
```
Now that we have settled everything, we will initialize the kubeadm
```
- name: Start the Kubeadm init
  shell: "kubeadm init"
  register: kubeadm_init_result
  ignore_errors: true

- name: Debugging the Result
  debug:
    msg: "{{ kubeadm_init_result }}"
```
After the kubernetes cluster is initialized, we are left with these things
Making sure that we have saved the join command from the output of the kubeadm init. This will be used to run the worker playbook to join the nodes.
Copying the admin.conf to user directory.
Installing calico plugin.

```
- name: Execute these if kubeadm is correctly started
  block:
    - name: Create the directory in user folder
      ansible.builtin.file: 
        path: '/home/{{ regular_user }}/.kube'
        owner: "{{ regular_user }}"
        group: "{{ regular_user }}"
        state: directory
        mode: 755
    
    - name: Export the config
      ansible.builtin.shell: "export KUBECONFIG=/etc/kubernetes/admin.conf"

    - name: Copy the kubeadm admin config from the etc dir to userdir
      ansible.builtin.shell: "{{ item }}"
      loop:
        - "cp /etc/kubernetes/admin.conf /home/{{ regular_user }}/.kube/config"
        - "chown {{ regular_user }}:{{ regular_user }} /home/{{ regular_user }}/.kube/config"

    - name: Install the calico Plugin
      become: false
      ansible.builtin.command: "kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/{{ calico_version }}/manifests/calico.yaml"
    
    - name: Check the cluster status 
      become: false
      ansible.builtin.command: "kubectl get nodes"
      register: cluster_initial

    - name: Copy the join command in a remote file
      ansible.builtin.copy:
        content: |
          "{{ kubeadm_init_result.stdout_lines }}"
        dest: "/tmp/join-command"

    - name: Cat the kubeadm join to get the single line of token
      ansible.builtin.shell: >
        cat /tmp/join-command | 
        awk -F ',' '{print $(NF-1) $(NF)}' |
        sed "s/\\\\' '\\\t//g" | 
        sed "s/\\\--discovery/--discovery/" |  
        sed "s/'\]\"//" | sed "s/'//"
      register: one_line_command

    - name: Rewriting the join command  
      ansible.builtin.copy:
        content: "{{ one_line_command.stdout }}"
        dest: "/tmp/join-command"

    - name: Get the join command from master node to local system
      ansible.builtin.fetch:
        src: "/tmp/join-command"
        dest: "./join-command"
        flat: yes

    - name: Displaying the Kubeadm Init Kubeadm Init Result
      ansible.builtin.debug:
        msg: "{{ kubeadm_init_result }}"

    - name: Displaying the Cluster status
      ansible.builtin.debug:
        msg: "{{ cluster_initial.stdout }}"
```

```
In the above code block, everything will be easy to understand except the part where join command is being copied, let me break that down for you.
First we copy the complete stdout of kubeadm init to a remote file.
name: Copy the join command in a remote file

 ansible.builtin.copy:
 content: |
 "{{ kubeadm_init_result.stdout_lines }}"
 dest: "/tmp/join-command"

This will have a lot many lines and so using awk command we extract the last two line which will contain our kubeadm join command, but this command will have will many extra punctuations and delimeters in it and we remove that as well using sed command multiple time.
name: Cat the kubeadm join to get the single line of token

 ansible.builtin.shell: >
 cat /tmp/join-command | 
 awk -F ',' '{print $(NF-1) $(NF)}' |
 sed "s/\\\\' '\\\t//g" | 
 sed "s/\\\ - discovery/ - discovery/" | 
 sed "s/'\]\"//" | sed "s/'//"
 register: one_line_command

And finally we get the single join command, that we overwrite in the same remote file and then copy the remote file in local machine.
```

The complete playbook should look like this.
```
---
- name: Playboook to configure the master node with kubernetes
  hosts: master
  become: true
  # change this variable as per your requirement
  vars:
    - cni_plugin_version: https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
    - regular_user: devuser
    - short_version: v1.27
    - version: 1.27.6-00 # not used yet, maybe of use later
    - calico_version: v3.27.1
  tasks:
    - name: Firewall configurations
      block:
        - name: Check if firewalld is installed
          stat:
            path: /usr/bin/firewall-cmd
          register: firewalld_installed
          ignore_errors: yes

        - name: Check if ufw is installed
          stat:
            path: /usr/sbin/ufw
          register: ufw_installed
          ignore_errors: yes

        - name: Open ports for firewalld
          when: firewalld_installed.stat.exists
          block:
            - name: Open port 6443 for API Server
              shell: "firewall-cmd --add-port=6443/tcp --permanent"
              ignore_errors: yes

            - name: Open port 2379 for Etcd client communication
              shell: "firewall-cmd --add-port=2379/tcp --permanent"
              ignore_errors: yes

            - name: Open port 2380 for Etcd server-to-server communication
              shell: "firewall-cmd --add-port=2380/tcp --permanent"
              ignore_errors: yes

            - name: Open port 10250 for Kubelet
              shell: "firewall-cmd --add-port=10250/tcp --permanent"
              ignore_errors: yes
            
            - name: Reloading the firewalld
              shell: "firewall-cmd --reload"     
              ignore_errors: yes           

        - name: Open ports for ufw
          when: ufw_installed.stat.exists
          block:
            - name: Open port 6443 for API Server
              shell: "ufw allow 6443/tcp"
              ignore_errors: yes

            - name: Open port 2379 for Etcd client communication
              shell: "ufw allow 2379/tcp"
              ignore_errors: yes

            - name: Open port 2380 for Etcd server-to-server communication
              shell: "ufw allow 2380/tcp"
              ignore_errors: yes

            - name: Open port 10250 for Kubelet
              shell: "ufw allow 10250/tcp"
              ignore_errors: yes

    - name: Update and Upgrade the apt packages before proceeding
      block:
        - name: performing the Update
          ansible.builtin.apt:
            update_cache: true

    - name: Block to manage Swap space
      block:
        - name: Check for swap to be on
          ansible.builtin.shell: "swapon -s | wc -l"
          register: number_of_swap
          changed_when: false

        - name: Turning off swaps
          ansible.builtin.shell: "swapoff -a"
          when: number_of_swap.stdout != 0

        - name: Check for swap in /etc/fstab
          ansible.builtin.shell: "sudo cat /etc/fstab | awk '{print $3}' | grep swap | wc -l"
          register: number_in_fstab
          changed_when: false

        - name: Changing entries in fstab
          ansible.builtin.lineinfile:
            path: /etc/fstab
            regexp: '.*swap.*'
            line: '# \g<0>'
          when: number_in_fstab.stdout != '0'
    
    - name: Change the hostname 
      ansible.builtin.hostname:
        name: master
    
    - name: Making neccesary changes in the etc hosts file
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "127.0.0.1 master"
        state: present

    - name: Configure the sysctl for kubernetes
      ansible.builtin.shell: "{{ item }}"
      loop:
        - "echo 'net.bridge.bridge-nf-call-ip6tables = 1' | tee /etc/sysctl.d/k8s.conf"
        - "echo 'net.bridge.bridge-nf-call-iptables = 1' | tee -a /etc/sysctl.d/k8s.conf"
        - "sysctl --system"
      ignore_errors: yes
    
    - name: Check if br_netfilter is set
      ansible.builtin.shell: "lsmod | grep br_net | wc -l"
      register: lsmod_results

    - name: Load br_netfilter module
      ansible.builtin.shell: "{{ item }}"
      loop:
        - "echo 'br_netfilter' | tee /etc/modules-load.d/k8s.conf"
        - "modprobe br_netfilter"
      ignore_errors: yes
      when: lsmod_results.stdout != '2'

    - name: Checking for ip_forward configurations
      ansible.builtin.shell: "cat /proc/sys/net/ipv4/ip_forward"
      register: ip_forward_result
      changed_when: false

    - name: Forward IP persistently if required
      block:
        - name: Forwarding the IP in sysctl
          ansible.builtin.lineinfile:
            path: /etc/sysctl.conf
            line: "net.ipv4.ip_forward=1"
            state: present
          register: forwarded 
        
        - name: Change the ip_forward for runtime
          ansible.builtin.shell: "sysctl -w net.ipv4.ip_forward=1"
          
        - name: Reloading the sysctl config
          ansible.builtin.shell: "sysctl -p /etc/sysctl.conf"
          when: forwarded is changed 
      when: ip_forward_result.stdout != '1'
    
    - name: Check and Remove docker if previous versions exist 
      block:
        - name: Remove the docker and containerd package
          ansible.builtin.apt:
            name:
              - docker.io
            state: absent

    - name: Add the apt repository of containerd and Install containerd runtime
      block:
        - name: Check if keyring is already present
          ansible.builtin.stat:
            path: /usr/share/keyrings/docker.gpg
          register: containerd_keyring_present

        - name: Adding the Containerd repo key
          ansible.builtin.apt_key:
            url: https://download.docker.com/linux/ubuntu/gpg
            keyring: /usr/share/keyrings/docker.gpg 
          when: containerd_keyring_present.stat.exists != 'true'

        - name: Retreive dpkg architecture 
          ansible.builtin.shell: "echo $(dpkg --print-architecture)"
          register: dpkg_architecture
          changed_when: false
          ignore_errors: true

        - name: The containerd repo
          ansible.builtin.shell: "echo 'deb [arch={{ dpkg_architecture.stdout }} signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable' | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null"
          changed_when: false
          ignore_errors: true

        - name: Installing containerd runtime
          ansible.builtin.apt:
            name: containerd
            state: present
    
    - name: Setting up the containerd runtime configurations
      block:
        - name: Making sure the unarchive directory exists 
          ansible.builtin.file: 
            path: "/opt/cni/bin/"
            state: directory

        - name: Download and unarchive the cni plugin tar file  
          ansible.builtin.unarchive:
            src: "{{ cni_plugin_version }}"
            dest: "/opt/cni/bin/"
            remote_src: yes
        
        - name: Make the directory
          ansible.builtin.file:
            path: /etc/containerd
            state: directory

        - name: Generate CNI configurations
          ansible.builtin.shell: "containerd config default > /etc/containerd/config.toml"
        
        - name: Change the systemd driver of containerd
          ansible.builtin.replace:
            path: /etc/containerd/config.toml
            regexp: 'SystemdCgroup = false'
            replace: 'SystemdCgroup = true'
            
    - name: Changing the user permissions to access container socket
      block:
        - name: Setting up a group
          ansible.builtin.group:
            name: containerd
            state: present
        
        - name: Retrive the group id
          ansible.builtin.shell: "cat /etc/group | grep containerd | awk -F ':' '{print $3}'"
          register: containerd_group_id

        - name: Adding the user to the group
          ansible.builtin.user: 
            name: devuser
            groups: containerd
            append: yes
        
        - name: Add the group id in the config file 
          ansible.builtin.replace:
            path: /etc/containerd/config.toml
            after: '[grpc]\n*address = "/run/containerd/containerd.sock"'
            regexp: gid = 0
            replace: 'gid = {{ containerd_group_id.stdout }}'

        - name: Changing permission of the socket to the group
          ansible.builtin.file: 
            path: "/run/containerd/containerd.sock"
            group: containerd

    - name: Reloading the containerd service
      ansible.builtin.service:
        name: containerd
        state: restarted
        enabled: true

    - name: Install dependent packages
      ansible.builtin.apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - conntrack
        state: present

    - name: Make the Keyring directory
      ansible.builtin.file:
        path: /usr/share/keyrings
        state: directory
        mode: 755

    - name: Add the Apt repository /usr/share/keyrings/kubernetes-archive-keyring.gpg
      block:
        - name: Check if the keyring file is already present
          ansible.builtin.stat:
            path: /usr/share/keyrings/kubernetes-archive-keyring.gpg
          register: gpg_key_present

        - name: Add apt key
          ansible.builtin.apt_key:
            url: https://pkgs.k8s.io/core:/stable:/{{ short_version }}/deb/Release.key
            keyring: /usr/share/keyrings/kubernetes-archive-keyring.gpg
          when: gpg_key_present.stat.exists != 'true'

    - name: Add the Kubernetes Repository
      ansible.builtin.apt_repository:
        repo: deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/{{ short_version }}/deb/ /
        state: present
        filename: kubernetes
    
    - name: Perform an apt update
      ansible.builtin.apt:
        update_cache: yes

    - name: Install the main kubeadm, kubelet, kubectl
      ansible.builtin.apt:
        name: 
          - 'kubelet'
          - 'kubeadm'
          - 'kubectl'
        state: present
      ignore_errors: true

    - name: Check if the containerd service is unmasked
      ansible.builtin.shell: "systemctl is-enabled containerd"
      register: masked_containerd
      changed_when: false

    - name: Unmask the containerd, if masked 
      ansible.builtin.shell: "systemctl unmask containerd"
      when: masked_containerd.stdout == 'masked'

    - name: Check if the kubelet service is unmasked
      ansible.builtin.shell: "systemctl is-enabled kubelet"
      register: masked_kubelet
      changed_when: false

    - name: Unmask the docker, if masked 
      ansible.builtin.shell: "systemctl unmask kubelet"
      when: masked_kubelet.stdout == 'masked'  

    - name: Making sure kubelet service is stopped
      ansible.builtin.service:
        name: kubelet
        state: stopped
        enabled: true
    
    - name: Making sure containerd service is started
      ansible.builtin.service:
        name: containerd
        state: started

    - name: Start the Kubeadm init
      ansible.builtin.shell: "kubeadm init"
      register: kubeadm_init_result
      ignore_errors: true

    - name: Debugging the Result
      ansible.builtin.debug:
        msg: "{{ kubeadm_init_result }}"

    - name: Execute these if kubeadm is correctly started
      block:
        - name: Create the directory in user folder
          ansible.builtin.file: 
            path: '/home/{{ regular_user }}/.kube'
            owner: "{{ regular_user }}"
            group: "{{ regular_user }}"
            state: directory
            mode: 755
        
        - name: Export the config
          ansible.builtin.shell: "export KUBECONFIG=/etc/kubernetes/admin.conf"

        - name: Copy the kubeadm admin config from the etc dir to userdir
          ansible.builtin.shell: "{{ item }}"
          loop:
            - "cp /etc/kubernetes/admin.conf /home/{{ regular_user }}/.kube/config"
            - "chown {{ regular_user }}:{{ regular_user }} /home/{{ regular_user }}/.kube/config"

        - name: Install the calico Plugin
          become: false
          ansible.builtin.command: "kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/{{ calico_version }}/manifests/calico.yaml"
        
        - name: Check the cluster status 
          become: false
          ansible.builtin.command: "kubectl get nodes"
          register: cluster_initial

        - name: Copy the join command in a remote file
          ansible.builtin.copy:
            content: |
              "{{ kubeadm_init_result.stdout_lines }}"
            dest: "/tmp/join-command"

        - name: Cat the kubeadm join to get the single line
          ansible.builtin.shell: >
            cat /tmp/join-command | 
            awk -F ',' '{print $(NF-1) $(NF)}' |
            sed "s/\\\\' '\\\t//g" | 
            sed "s/\\\--discovery/--discovery/" |  
            sed "s/'\]\"//" | sed "s/'//"
          register: one_line_command

        - name: Rewriting the join command  
          ansible.builtin.copy:
            content: "{{ one_line_command.stdout }}"
            dest: "/tmp/join-command"

        - name: Get the join command in main node
          ansible.builtin.fetch:
            src: "/tmp/join-command"
            dest: "./join-command"
            flat: yes

        - name: Displaying the Kubeadm Init Kubeadm Init Result
          ansible.builtin.debug:
            msg: "{{ kubeadm_init_result }}"

        - name: Displaying the Cluster status
          ansible.builtin.debug:
            msg: "{{ cluster_initial.stdout }}"
```    
If you want to see the output of the cluster initialized, in the same playbook write a second play that will run as the regular user and run the kubectl get nodes command and also generate the bash completion so that you can press tab to autocomplete and work with ease.
```
- name: Playbook for checking from regular user
  hosts: master
  become: false
  tasks:
    - name: Checking for the shell of system
      shell: "echo $SHELL | awk -F '/' '{print $3}'"
      register: system_shell
      ignore_errors: true
    
    - name: Setting the bash completion
      shell: "{{ item }}"
      loop:
        - "kubectl completion {{ system_shell.stdout }} > ~/.kube/kubectl_completion.sh"
        - "echo 'source ~/.kube/kubectl_completion.sh' >> ~/.bashrc"
```
Below is the ansible.cfg file for reference
```
[defaults]
inventory=./inventory
remote_user=devuser

[privilege_escalation]
become_user=root
become_ask_pass=false
become_method=sudo
```
Also, here is the inventory sample for you [don't forget to change the IP's]
```
[master]
192.168.1.1

[worker]
192.168.1.2
192.168.1.3
```
Assuming you have configured ssh password-less authentication to the remote host and placed all the files ansible.cfg, inventory and the master.yml in the same directory, run the playbook, sit-back and enjoy the colors in your terminal setting up the master node.
Remember to execute the below command from the same directory where the master.yml, inventory and ansible.cfg files are.
```
#To execute the playbook
ansible-playbook master.yml
```
You can clone the Github Repo as well for easy retrieval of codes:
GitHub - s-vintagew/kubernetes-setup-using-ansible: Install and Configure Kubernetes Master Node
Install and Configure Kubernetes Master Node using Ansible playbook - s-vintagew/kubernetes-setup-using-ansiblegithub.com