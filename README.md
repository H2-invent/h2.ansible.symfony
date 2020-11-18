ansistrano-symfony-deploy
=========
Extention by h2 invent
------------
A larger set of Symfony deployment activities in combination with ansitrano deploy playbook. In this playbook, the extra steps are included in the automated deployment:
* Create an .env.local
* Start encryption
* Use Webpack with encore for Symfony
* Make a security Check from Composer

This role included an run_once as default vaule in doctrine migration. It is important to enable this feature when the Symfony Application runs in a LAMP cluster with a centralized database server or cluster.

#### IMPORTANT
To securely store the env variables, it is usefull to create ansible-vault file with all the .env parameters. Do not store env and encryption key paramters in plain text on git or somewhere else.

A set of [Ansible](http://docs.ansible.com/) tasks for deploying PHP applications developed using the Symfony framework (incl. flex) onto *nix servers in a "Capistrano" fashion (releases, shared, current->releases/X).

This role is more or less just a collection of the most common post-installation setup tasks (i.e. getting a Composer executable, installing dependencies and autoloader, perform cache warming, deploy migrations etc). It does not itself deal with setting up the directory structures or getting files onto your servers - that part is handled by the more generic `ansistrano-deploy` role.

The way this is implemented is by defining a couple of the `ansistrano_(before|after)_X` vars (see the [ansistrano docs](https://github.com/ansistrano/deploy#main-workflow) for details).

Requirements
------------

Due to shortcomings in the current version of Ansible, it __is__ a requirement that the `ansistrano-symfony-deploy` and `ansistrano-deploy` share the same parent directory. This will be the normal case so shouldn't be an issue as long as you install via `ansible-galaxy`.

The tasks will _probably not_ work for Windows target hosts (untested).

Role Variables
--------------

The `defaults` vars declared in this module:

```YAML
symfony_env: prod
symfony_console_path: 'bin/console'
symfony_php_path: php # The PHP executable to use for all command line tasks

symfony_run_composer: true
symfony_composer_path: "{{ ansistrano_release_path.stdout }}/composer.phar"
symfony_composer_options: '--no-dev --optimize-autoloader --no-interaction'
symfony_composer_self_update: true # Always attempt a composer self-update

symfony_run_assets_install: true
symfony_assets_options: '--no-interaction'

symfony_run_assetic_dump: false
symfony_assetic_options: '--no-interaction'

symfony_create_env: true
symfony_create_env_path: "{{ansistrano_release_path.stdout}}/.env.local" # Path and name of the .env or .env.local file
symfony_create_env_vars: []
#  - name:
#    value:
#    comment:
#    activ: true

symfony_create_env_vars_host: []
#  - name:
#    value:
#    comment:

symfony_create_encryption: false
symfony_create_encryption_key: ''
symfony_create_encryption_key_path: "{{ansistrano_release_path.stdout}}/.Halite" # Path and name of the key, pem or Halite file

symfony_run_cache_clear_and_warmup: true
symfony_cache_options: ''

symfony_run_webpack_update: true
symfony_npm_run_install_command: npm install
symfony_npm_run_build_command: npm run build

symfony_run_secure_check: true
symfony_secure_check_path: "{{ ansistrano_release_path.stdout }}/security-checker.phar"
symfony_composer_lock_path: "{{ ansistrano_release_path.stdout }}/composer.lock"
symfony_secure_check_command: "{{ symfony_php_path }} {{ symfony_secure_check_path }} security:check {{ symfony_composer_lock_path }}"

symfony_run_doctrine_migrations: false
symfony_doctrine_options: '--no-interaction'
symfony_run_doctrine_migration_once: true

symfony_run_mongodb_schema_update: false
symfony_mongodb_options: ''
```

Hooks
-----

This role supports ansistrano like hooks before and after every task

```YAML
ansistrano_symfony_before_composer_tasks_file
ansistrano_symfony_after_composer_tasks_file

ansistrano_symfony_before_env_tasks_file
ansistrano_symfony_after_env_tasks_file

ansistrano_symfony_before_encryption_tasks_file
ansistrano_symfony_after_encryption_tasks_file

ansistrano_symfony_before_assets_tasks_file
ansistrano_symfony_after_assets_tasks_file

ansistrano_symfony_before_assetic_tasks_file
ansistrano_symfony_after_assetic_tasks_file

ansistrano_symfony_before_cache_tasks_file
ansistrano_symfony_after_cache_tasks_file

ansistrano_symfony_before_doctrine_tasks_file
ansistrano_symfony_after_doctrine_tasks_file

ansistrano_symfony_before_mongodb_tasks_file
ansistrano_symfony_after_mongodb_tasks_file

ansistrano_symfony_before_webpack_tasks_file
ansistrano_symfony_after_webpack_tasks_file

ansistrano_symfony_before_secure_tasks_file
ansistrano_symfony_after_secure_tasks_file
```

In addition to this, please also refer to the [list of variables used by ansistrano](https://github.com/ansistrano/deploy#role-variables).

Note about ORM/ODM schema migrations
------------------------------------

Database schema migrations can generally NOT be run in parallel across multiple hosts! For this reason, the `symfony_run_doctrine_migrations` and `symfony_run_mongodb_schema_update` options both come turned off by default.

In order to get around the parallel exection, you can do the following:

1. Specify your own `ansistrano_before_symlink_tasks_file`, perhaps with the one in this project as a template (look in h2.ansible.symfony/config/steps/).
2. Pick one of the following approaches:
  - (a) Organize hosts into groups such that the task will run on only the _first_ host in some group:
    `when: groups['www-production'][0] == inventory_hostname`
  - (b) Use the `run_once: true` perhaps with `delegate_to: some_primary_host` ([Docs: Playbook delegation](http://docs.ansible.com/ansible/playbooks_delegation.html#run-once))

Dependencies
------------

- [ansistrano.deploy](https://galaxy.ansible.com/ansistrano/deploy)

Installing from the command line via a requirements.yml `src: git+git@github.com:H2-invent/h2.ansible.symfony.git` should pull down the external role as a dependency, so no extra step neccessary.

Example playbook
----------------

As a bare minimum, you probably need to declare the `ansistrano_deploy_from` and `ansistrano_deploy_to` variables in your play. Ansistrano defaults to using rsync from a local directory (again, see the [ansistrano docs](https://github.com/ansistrano/deploy)).

Let's assume there is a `my-app-infrastructure/deploy.yml`:
```YAML
---
- hosts: all
  gather_facts: false
  vars:
    ansistrano_deploy_to: /home/app-user/my-project-deploy/
    ansistrano_deploy_via: "git"
    ansistrano_git_repo: "https://github.com/H2-invent/open-datenschutzcenter.git"
    ansistrano_before_symlink_tasks_file: "{{playbook_dir}}/config/app_specific_setup.yml"
  roles:
    - h2.ansible.symfony
```

This playbook should be executed like any other, i.e. `ansible-playbook -i some_hosts_file deploy.yml`.

It probably makes sense to keep your one-off system prep tasks in a separate playbook, e.g. `my-app-infrastructure/setup.yml`.

License
-------

MIT

Author Information
------------------
- ansistrano-symfony-deploy, written by Conny Brunnkvist <cbrunnkvist@gmail.com>
- The underlying role is maintained by the `ansistrano-deploy` team
- Some code was taken from/inspiried by the `symfony2-deploy` role by the Servergrove team
