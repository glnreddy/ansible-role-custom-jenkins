Ansible Role Custom Jenkins
===========================

<img align="left" margin="10px 10px 10px 10px" src="https://wiki.jenkins-ci.org/download/attachments/2916393/jenkins-thpr.svg"></img>

This role will attempt to deliver a customized [Jenkins](http://www.jenkis-ci.org); there is no difference from the "regular Vanilla [Jenkins](http://www.jenkis-ci.org)" except for it's built in configurations ...
I hope this role will not be too opinionated ;) ...
P.S I couldn;t of chosen a better icon/log myself ;) - Credits to -> [masanobu imai](https://github.com/masanobuimai)
<br/><br/><br/><br/>

State of Mind
-------------
**Automate everything !**
* Plugins
* Authentication & Authorization [ Matrix || Github Auth || LDAP || Active Directory ]
* 3rd party integrations such as: 
    * Git (hub || lab || bitucket || Gerrit), 
    * Artifact repo [ Artifactory || Nexus ], 
    * Tools [ JDK's || NPM || Others ...] etc etc, 
    * some still [WIP] ...
* Proxy configuration for plugin and rpm/apt caching { seperate machine || corporate proxy }
* All configuration are perfromed either via the CLI / Groovy interface { init.groovy directory | cli groovy input } - avoid copying files into jenkins home and restarting in case api changes between releases.
* For demo purposes insert a seed DSL groovy script and trigger it.
* Optionally use Jenkins Material Theme


Requirements
------------

Java 1.8_x 

Features OOTB (Out Of The Box)
------------------------------

1. Admin users [ "admin", "jjb" ] with corresponding passwords - recommended to use [Ansible vault](http://docs.ansible.com/ansible/playbooks_vault.html) for this
2. 'Regular' users users [ "user1", "user2" ] with corresponding passwords (as mentioned probebly move th LDAP | Github auth in the near future)
3. Matrix Authorization Strategy (More to come ...), granting admin users overall admin privileges and Overall Read for regular users and anonymous  
4. "Wizard" plugins - the default plugins you would install following the Jenkins 2.x startup wizard
5. "Goodies" plugins - stuff like [Greenballs](https://wiki.jenkins-ci.org/display/JENKINS/Green+Balls) (who dosen't use green balls ha ?!) 
5. Jenkins proxy - **disabled by default** 
6. [Still WIP] SSH Authorization - this is a prep for adding an ssh key via vault so we can start with git clone at the end of the play - see `tasks/cj-ssh-key.yml` it will reject an existing key if present in `~jenkins/.ssh` 

Role Variables & Defaults
-------------------------
* `jenkins_http_port: 8080` - https support when added will be done via nginx / apache2 
* `Jenkins_version: latest-LTS` - this controls what repository will the rpm/qpt be taken from see the `vars/{{ ansible_os_family }}.yml` to see the urls.
* `jenkins_application_context: ''` - defaults to none which is really `http://jenkins-host/` change it to your liking ...
*  Jenkins directories used thought the role
```yamlex
    jenkins_home: /var/lib/jenkins                          # the default home directory just like the apt/rpm
    jenkins_log_dir: /var/log/jenkins                       # default log file
    jenkins_cache_dir: /var/cache/jenkins                   # cache directory { used to focus on jenkins-cli.jar
    jenkins_init_dir: "{{ jenkins_home }}/init.groovy.d"    # post installation scripts go here 
    jenkins_update_dir: "{{ jenkins_home }}/updates"        # unpadte.center.json goes here
    jenkins_webroot: /var/cache/jenkins/war                 # 
    jenkins_plugins_tmp_folder: /tmp/plugins                # will probebly be removed in next release { still figuring out offline instllation }
```
*  Set jenkins configuration [ I used the [squid proxy](https://github.com/shelleg/ansible-role-squid) role]
```yamlex
    jenkins_set_proxy: false
    jenkins_proxy_host: "{{ proxy_server }}"
    jenkins_proxy_port: "{{ proxy_port |d('3148') }}"
    #jenkins_no_proxy_hosts: will be set in tasks
    jenkins_proxy_conf_file: "{{ jenkins_home }}/proxy.xml"
    # in case we want to test jenkins without a proxy when jenkins_set_proxy is set to true
    jenkins_remove_proxy: false
```
* Disable the startup wizard (I will install these plugins anyway later on) 
```yamlex
    jenkins_run_startup_wizard: "false" # String becuase the jvm arg becomes False ...
    jenkins_jvm_args: "-Djava.awt.headless=true -Djenkins.install.runSetupWizard={{ jenkins_run_startup_wizard }}"
    jenkins_cmd_args: "--webroot={{ jenkins_webroot }} --httpPort={{ jenkins_http_port }}"
```
* DEPRECATED / USE AT OWN RISK - running jenkins as a different user, this will probebly still work so I did not throw away a couple of hours of work ... 
```yamlex
    # The system level user Jenkins should runas
    jenkins_runas: false
    #jenkins_custom_user: user
    #jenkins_custom_group: user
    jenkins_runas_user: "{{ jenkins_custom_user | d('jenkins') }}"
    jenkins_runas_group: "{{ jenkins_custom_group | d('jenkins') }}"
    jenkins_runas_sudoer: true
```
* Authorization strategy basic - used to populate / generate the `templates/basic-security.groovy.j2` file inserted into `init.groovy.d` directory
```yamlex
    jenkins_allow_anon_read: true
    jenkins_basic_auth: true
    jenkins_local_admin_users:
      - { username: admin, password: admin }
      - { username: jjb, password: jjb }
    
    jenkins_local_regular_users:
      - { username: user1, password: user1 }
      - { username: user2, password: user2 }
```

* jenkins-cli related - most plugin ops are done via CLI so basically you can perfrom these remotely or via Docker running jenkins (same goes for groovy scripts metioned above)
```yamlex
    jenkins_cli_jar_location: "{{ jenkins_cache_dir }}/war/WEB-INF/jenkins-cli.jar"`
    jenkins_cli_hostname: 'localhost'
```

* Jenkins material theme
```yamlex
    # could be: red, pink, purple, deep-purple see full list @ -> http://afonsof.com/jenkins-material-theme/
    jenkins_theme_color: amber
    jenkins_theme_conf_file: org.codefirst.SimpleThemeDecorator.xml
    jenkins_theme_css_url: https://jenkins-contrib-themes.github.io/jenkins-material-theme/dist/material-{{ jenkins_theme_color }}.css
    jenkins_theme_js_url: https://cdn.rawgit.com/djonsson/jenkins-atlassian-theme/gh-pages/theme.js

```
* Jenkins plugin sets ... this still needs some work, this is the begining which will change based on requirements
```yamlex
    # choose plugin sets to use
    jenkins_wizard: true
    jenkins_theme: true
    jenkins_setup_matrix_auth_strategy: true
    jenkins_setup_dsl_plugins: true
    jenkins_goodies: true
```
* Jenkins short see job demo:
```yamlex
  jenkins_seed_demo: true # this will probebly be set to false upon release
```
* Jenkins plugin lists - this is also a preparation for offline installation mode - as an example:
```yamlex
    jenkins_wizard_plugins:
      - { name: "bouncycastle-api", version: "2.16.0", enabled: "true", extention: "hpi" }
      - { name: "cloudbees-folder", version: "5.15", enabled: "true", extention: "hpi" }
      - ...
    jenkins_goodie_plugins:
      - { name: "greenballs" }
```

Dependencies
------------

Java 1.8 I used the following ansible role:

```yamlex
- src: https://github.com/shelleg/ansible-role-oracle-java.git
  version: 2.9.0
  name: oracle-java
```
See requirements.yml attached to this repo


Example Playbook
----------------

Full featured + proxy -> end of play should result with http://custom-jenkins:8080/ login with admin/admin (or jjb/jjb, user1/user1 etc):

    - hosts: custom-jenkins
      become: true
      vars:
        jenkins_set_proxy: true             # could be any proxy [tested with squid]
        proxy_server: "172.16.1.99"         # the custom proxy IP|hostname
        noproxy_hosts:                      # what not to proxy ...
          - '127.0.0.1'
          - '172.16.1.*'
      roles:                                
        - role: squid                       # set proxy client for apt/yum (more to come)
          squid_server_mode: false      
        - role: oracle-java                 # install java 1.8
        - role: epel                        # redundent ...
        - role: git                         # always comes in handy
        - role: ansible-role-custom-jenkins # this role ...
        #- role: jenkins-populate-jobs      # planned adding your own jobs via seperate addon role [ WIP see `tasks/cj-seed.yml` as a starting point ]

Custom Jenkins - disable defaults:

    - hosts: custom-jenkins
      become: true
      roles:                                
        - role: oracle-java                 # install java 1.8
        - role: ansible-role-custom-jenkins # this role ...
          jenkins_allow_anon_read: false    # no anonymous read !
          jenkins_basic_auth: false         # no authentication !
          jenkins_theme: false              # stay with classic jenkins ui  

Contributing
------------
- Testing, Testing, Testing ...
    - testing the different flows, and there are a few ...

- Groovy Groovy Groovy:
  Basically anything you want done could be passed as a groovy script which is what I am aiming at the end see the example of `tasks/cj-init_config.yml`
  Add your script to the templates folder (toknized or not) and it will make it's way to Jenkins init ...
```yamlex
    - name: "Generate groovy script for jenkins bootstrap execution"
      template:
        src: "{{ item }}.j2"
        dest: "{{ jenkins_init_dir }}/{{ item }}"
        owner:  "{{ jenkins_runas_user | d('jenkins')}}"
        group:  "{{ jenkins_runas_group | d('jenkins')}}"
      notify: "service jenkins restart"
      with_items:
        - basic-security.groovy
        - your scritp goes here ...
```

- Adding a new plugin set:
    1. Manually install plugins ... on the Jenkins master run the following command:
```bash
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s http://localhost:8080 list-plugins \
    | awk '{print $1, $NF;}' \
    | awk -F " " ' {print " - { name: \""$1 "\","" version: \""  $2 "\", enabled: \"true\" }"} '
```
The returned list will be formed like so:
```yamlex
      - { name: "bouncycastle-api", version: "2.16.0", enabled: "true", extention: "hpi" }
```

   2. Add a new plugin set variable such as: `jenkins_cool_plugin_set:` as the list name with your plugin list like so in `defaults/main.yml` like so:
```yamlex
    jenkins_cool_plugin_set:
      - { name: "my-cool-plugin", version: "some_ver", enabled: "true", extention: "hpi" }
```  
   3. Add a boolean "controller | feature toggle" variable such as:  jenkins_cool_plugin_set_enabled: true | false
   
   4. Merge the plugin lists in `tasks/cj-plugin-sets.yml` like so:

```yamlex
    - name: "merge plugin lists -> jenkins cool plugins"
      set_fact:
        jenkins_plugins: "{{ jenkins_plugins + jenkins_cool_plugin_set_enabled }}"
      when: (jenkins_cool_plugin_set_enabled is defined) and (jenkins_cool_plugin_set_enabled | bool)
```  

- Finding a way for offline installation and dependecney resolution between plugins

License
-------

MIT

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
