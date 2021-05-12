# Librarian
Librarian is a stateless CMS based on static files. It uses the same format as DEV.to for markdown files with a front matter and liquid tags for custom functionality.

# Ansible Setup

The setup included in this repository is the same used by the [Librarian documentation](https://librarianphp.dev) (which is a Librarian app) for setting up the remote server and deploying the application to production with Nginx + PHP-FPM.

## How to Use It
You can download the files in this repository and save them in the root folder of your Librarian application. You may want to include the inventory file and the variable files in your project's `.gitignore` at your own discretion, but overall you should want to keep the provisioning scripts in version control to take full advantage of Ansible's infrastructure as code capabilities.

## Playbooks Included

Two playbooks are included within this setup: `server-setup.yml` and `deploy.yml`. 

### Setting Up Inventory

You'll need to edit the `inventory` file and replace the example IP address with your own server's IP address.

```ini
[dev]
librarian ansible_host=192.168.10.10

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

After saving and closing the file, you can test out your connection with a `ping` command:

```shell
ansible all -i inventory -m ping -u root
```

### Setting Up Required Variables

Both playbooks use variables defined within `group_vars/all.yml`. There are two important variables that need to be changed here:
- `http_host` - The domain name or IP address for your Librarian website, this will be used to configure Nginx.
- `app_root_dir` - For the `rsync` command to work without issues, you need to set the application directory name here.

You may also want to change the `remote_system_user`. This user will be created on the server as a passwordless `sudo` user that you'll then use to run the `deploy` playbook to deploy your application whenever you want to.
That is important for the `rsync` module to work without issues.

```yml
# group_gars/all.yml
---
# Initial Server Setup
remote_system_user: librarian
sys_packages: [ 'curl', 'vim', 'git', 'ufw', 'acl', 'imagemagick', 'curl']

# PHP
php_packages: [ "php-imagick", "php-curl", "php-json", "php-mbstring", "php-bcmath", "php-xml" ]

# Web Server Setup
http_host: librarianphp.dev
remote_www_root: /var/www
app_root_dir: librarian
document_root: "{{ remote_www_root }}/{{ app_root_dir }}/web"
```

## Running the Server Setup

The `server-setup.yml` includes roles to set up an Nginx + PHP-FPM server.

The `server-setup.yml` playbook can be used on a **fresh Ubuntu server** to set up all requirements including Composer.
```yml
# server-setup.yml
---
- hosts: all
  become: true
  roles:
    - { role: setup, tags: ['setup'] }

    - { role: php, tags: ['php', 'web', 'php-fpm'] }

    - { role: nginx, tags: ['nginx', 'web', 'http'] }

    - { role: composer, tags: ['composer'] }
```

You can run this playbook with:

```shell
ansible-playbook -i inventory server-setup.yml -u root
```
## Running the Deploy Playbook

The `deploy.yml` playbook will make sure the remote document root exists, and will `rsync` the current folder (`.`) with the remote server.

It will then try to execute `composer install` on the application's root folder.

```yaml
---
- hosts: all
  become: true
  vars_files:
    - ansible/group_vars/all.yml
  tasks:
    - name: Make sure the remote app root exists and has the right permissions
      file:
        path: "{{ remote_www_root }}/{{ app_root_dir }}"
        state: directory
        mode: '0755'
        owner: "{{ remote_system_user }}"
        group: "{{ remote_system_user }}"

    - name: Rsync application files to the remote server
      synchronize:
        src: "."
        dest: "{{ remote_www_root }}"
        rsync_opts:
          - "--no-motd"
          - "--exclude=.git"
          - "--exclude=.github"
          - "--exclude=ansible"
          - "--exclude=docker-compose*"
          - "--exclude=Dockerfile"
          - "--exclude=vendor"
      tags: [ 'sync' ]

    - name: Set up additional directory permissions for www-data user on storage folder
      acl:
        path: "{{ remote_www_root }}/{{ app_root_dir }}/var/"
        entry: group:www-data:rwX
        recursive: yes
        state: present

    - name: Install Dependencies with Composer
      become: false
      composer:
        command: install
        working_dir: "{{ remote_www_root }}/{{ app_root_dir }}"
      tags: [ 'composer' ]
```

Considering you have previously run the `server-setup.yml` playbook and has a passwordless sudo created and named `librarian`, you can run the deploy playbook with:

```shell
ansible-playbook -i inventory deploy.yml -u librarian
```

### Only Syncing Files

If all you want to do is synchronize the app files (upload changes), then you can run:

```shell
ansible-playbook -i inventory deploy.yml -u librarian --tags=sync
```

### Running only Composer

If you just want to run a `composer install` on the app's folder, you can run the deploy playbook like this:

```shell
ansible-playbook -i inventory deploy.yml -u librarian --tags=composer
```