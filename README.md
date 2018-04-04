CheckMkClient
=============

Configures a system's check_mk client.

- Installs check-mk-agent on clients; Does use locally provided packages only; See example playbook and comments;
- Puts basic check_mk plugins in place
- Adds ssh hostkey to the check_mk site user
- Creates a user and group to run check-mk-agent with
- Creates sudo configuration for this monitoring user
- Puts ssh-public-key in place for the monitoring user

Requirements
------------

No special requirements;

Role Variables
--------------

- Check_Mk Server host; required: yes
  The ansible_inventory name of the check_mk server.

      cmkclient_server: 

- Check_Mk Package name and version; required: yes
  These are used to check if the desired package is already installed.
   
      cmkclient_pkgname: check-mk-agent
      cmkclient_pkgver: 1.4.0p27-1

- Check_Mk Site user name; required: yes
  The username of the user running your Check_Mk site.

      cmkclient_site_user: example-site

- Check_Mk package and plugins; (`vars/apt.yml` or `vars/dnf.yml`)
  These vars are defined in files dependend on the `ansible_pkg_mgr` var. Below defaults are only showing the defaults for Debian.

  - Packages; required: no
    Either one of these should be defined to install a package. If `cmkclient_pkgmanager_pkg` is defined the systems configured repositories will be used to install the package. If `cmkclient_package` is defined you have to provide the package in some way (eg. in your 'files' directory and provide the full path). If both are defined the distributions packagemanager has precedence.

        cmkclient_pkgmanager_pkg: files/checkmk/check-mk-agent
        cmkclient_package: check-mk-agent_1.4.0p27-1_all.deb

  - Plugins; required: no

        cmkclient_plugins: 
           - plugin_src: files/checkmk/mk_apt
             plugin_dst: /usr/lib/check_mk_agent/plugins/86400/mk_apt

- Command to used for checking if the package  is already installed; required: yes

      cmkclient_pkg_test: "dpkg -l | grep -P \'^ii\\s+{{ cmkclient_pkgname }}\\s+{{ cmkclient_pkgver }}\'"

- User configuration if check-mk-agent is not executed as `root`                                                                                                                                         
  - Create user or not; required: yes; (True/False)

        cmkclient_createuser: False
  - Group configuration

        cmkclient_group:
          name: checkmk
          gid: 601
  - User configuration
    Sudo configuration is only applied if `cmkclient_createuser` is `True` and `cmkclient_user.sudo` is not `False`.

        cmkclient_user:
          name: checkmk
          uid: 601
          comment: "check_mk monitoring user"
          group: checkmk
          shell: /bin/bash
          password: '!!'
          home: /home/checkmk
          pwexpire: '99999' # This must be a string!
          sudo: checkmk
          publickey: "{{ lookup('file', 'files/authorized_keys/checkmk') }}"



Dependencies
------------

None;

Example Playbook
----------------

- A very basic way of using the role that needs to have your check-mk-agent packages and plugins stored locally.

      - name: Configure Check_MK Client
        hosts: MonitoredServers
        roles:
          - { role: CheckMkClient }

- A more sophisticated aproach that temporarly fetches packages and plugins from the Check_Mk Server.

      - hosts: localhost
        tasks:
        - name: Retrieving checks and clients from check_mk system
          tempfile:
            state: directory
            prefix: ansible.cmk.
          register: localtmpdir

      - hosts: checkmk.example.com
        vars:
          all_modules:
            - /omd/sites/example-site/local/share/check_mk/agents/plugins/mk_apt
            - /omd/sites/example-site/local/share/check_mk/agents/plugins/yum
          all_packages:
            - /omd/sites/example-site/share/check_mk/agents/check-mk-agent-1.4.0p27-1.noarch.rpm
            - /omd/sites/example-site/share/check_mk/agents/check-mk-agent_1.4.0p27-1_all.deb

        tasks:
        - name: Fetch packages and plugins from check_mk server
          fetch:
            flat: yes
            fail_on_missing: yes
            src: "{{ item }}"
            dest: "{{ hostvars['localhost']['localtmpdir']['path'] }}/"
          with_items:
            - "{{ all_modules }}"
            - "{{ all_packages }}"

      - name: Configure Check_MK Client 
        hosts: MonitoredServers
        gather_facts: true
        roles:
          - { role: CheckMkClient, localtmpdir: "{{ hostvars['localhost']['localtmpdir'] }}" }

      - hosts: localhost
        gather_facts: false
        tasks:
        - name: Clean tmp data
          file:
            path: "{{ localtmpdir.path }}/"
            state: absent

License
-------

BSD

Author Information
------------------

Created in 2018 by [Thomas](https://blog.waan.name/).
