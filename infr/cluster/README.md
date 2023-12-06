# Ansible folder to build Kubernetes Cluster


### copy ssh key to remote server
```bash
    ssh-keygen -f "/home/allen/.ssh/known_hosts" -R "10.171.184.193"
    ssh-keygen -f "/home/allen/.ssh/known_hosts" -R "10.171.184.194"
    ssh-keygen -f "/home/allen/.ssh/known_hosts" -R "10.171.184.195"
    ssh-keygen -f "/home/allen/.ssh/known_hosts" -R "10.171.184.197"

    sshpass -p "default" ssh-copy-id -i ~/.ssh/node108.pub -o StrictHostKeyChecking=accept-new  root@10.171.184.193
    sshpass -p "default" ssh-copy-id -i ~/.ssh/node108.pub -o StrictHostKeyChecking=accept-new  root@10.171.184.194
    sshpass -p "default" ssh-copy-id -i ~/.ssh/node108.pub -o StrictHostKeyChecking=accept-new  root@10.171.184.195
    sshpass -p "default" ssh-copy-id -i ~/.ssh/node108.pub -o StrictHostKeyChecking=accept-new  root@10.171.184.197

sshpass -p "default" ssh-copy-id -i ~/.ssh/node108.pub -o StrictHostKeyChecking=accept-new  root@runner
sshpass -p "default" ssh-copy-id -i ~/.ssh/node108.pub -o StrictHostKeyChecking=accept-new  root@master01
sshpass -p "default" ssh-copy-id -i ~/.ssh/node108.pub -o StrictHostKeyChecking=accept-new  root@worker01
sshpass -p "default" ssh-copy-id -i ~/.ssh/node108.pub -o StrictHostKeyChecking=accept-new  root@worker02
```

### test connectivity
```bash
ansible-playbook 000-connectivity.yml 
```

### configure /etc/hosts file
```bash
ansible-playbook 001-set-hosts.yml -e "domain_name=master01 domain_ip=10.61.255.201"
ansible-playbook 001-set-hosts.yml -e "domain_name=worker01 domain_ip=10.61.255.202"
ansible-playbook 001-set-hosts.yml -e "domain_name=worker02 domain_ip=10.61.255.203"
ansible-playbook 001-set-hosts.yml -e "domain_name=runner domain_ip=10.61.255.101"


ansible-playbook 001-set-hosts.yml -e "domain_name=master01 domain_ip=10.81.255.201"
ansible-playbook 001-set-hosts.yml -e "domain_name=worker01 domain_ip=10.81.255.202"
ansible-playbook 001-set-hosts.yml -e "domain_name=worker02 domain_ip=10.81.255.203"
ansible-playbook 001-set-hosts.yml -e "domain_name=runner domain_ip=10.81.255.101"
```

### system level setup for all nodes
```bash
ansible-playbook 010-install-kubenode.yml 
```

### create 1st master node
```bash
ansible-playbook 020-create-1st-masternode.yml 
```

### create a runner
```bash
ansible-playbook 021-install-runner.yml 
```

### add worker nodes into cluster
```bash
ansible-playbook 030-add-workers.yml 
```

### install cni
```bash
ansible-playbook 031-install-cni.yml 
```