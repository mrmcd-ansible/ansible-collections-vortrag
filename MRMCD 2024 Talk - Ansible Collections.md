---
type: slide
tags: mrmcd
controls: true
title: MRMCD 2024 Talk - Ansible Collections
slideOptions:
  transition: slide
  slideNumber: false
---

<style>
  pre code {
    width: 100%;
  }
  .reveal pre {
    width: 115%;
    margin-left: -50px;
  }
</style>

<img width="512px" style="border: none; background: rgba(0,0,0,0); box-shadow: none;" src="https://md.entropia.de/uploads/41128c4c-029d-49d6-b821-bc8ef1b3dce0.svg" />


---

In diesem Vortrag wollen wir euch ein Einstieg in Ansible Collections geben.
+ Was sind Collections?
+ Wie nutzt man Sie?
+ Wie erstellt ihr welche?

---

**Ansibl**e ist ein Open-Source-Automatisierungstool.

Es vereint Softwareverteilung, die Ausführung von Ad-hoc-Befehlen und das Management von Software Konfigurationen.

----

**Ansible Collections** bieten die Möglichkeit, dezentral Rollen, Module und Plugins zu schreiben und diese in anderen Projekten wieder einzubinden.

----

2020 wurden mit Ansible 2.10 Fully Qualified Collection Names (FQCN) eingeführt.
Dies ermöglicht es, Ansible Module, Filter, Plugins und Rollen in seine Projekte zu integrieren und mit der Community zu teilen.

----

Diese Ansible Collections werden meist auf [galaxy.ansible.com](https://galaxy.ansible.com) geteilt.

---

**Anslyse der Ansible Collection
l3d.git** (galaxy.ansible.com/l3d/git)
und wie Sie aufgebaut ist.

---

Aufbau der Collection: 


<img width="512px" style="border: none; background: rgba(0,0,0,0); box-shadow: none;" src="https://md.entropia.de/uploads/d3708399-b617-4405-9d6a-d4abe9fbd1b8.svg" />

----

Ansible Collection Metadaten *(``galaxy.yml``)*

```yml
namespace: l3d 
version: 1.1.6 
readme: README.md 
authors:
  - L3D <l3d@c3woc.de> 
license:
  - MIT 
tags:
  - gitea
  - linux 
dependencies: 
  "community.general": ">=9.4.0"
repository: https://github.com/roles-ansible/ansible_coll... 
homepage: https://ansible.l3d.space/#l3d.git
```


---

Ansible Rolle **l3d.git.gitea**

<img width="512px" style="border: none; background: rgba(0,0,0,0); box-shadow: none;" src="https://md.entropia.de/uploads/b1822541-e76b-485f-b1a7-bd43c913e577.svg" />



----

**``l3d.git.gitea``**

``tasks/main.yml``

```yml=1
---
- name: Perform optional versionscheck
  ansible.builtin.include_tasks:
    file: "versioncheck.yml"
  when: submodules_versioncheck | bool

- name: Gather installed packages for checks later on
  ansible.builtin.package_facts:
    manager: "auto"
    
```

----

``tasks/main.yml``

```yml=11
- name: Prepare gitea/forgejo variable import
  block:
    - name: Gather vars for gitea or forgejo
      ansible.builtin.include_vars:
        file: "{{ lookup('ansible.builtin.first_found',
                 gitea_fork_variables) }}"
  rescue:
    - name: Gitea/Forgejo import info
      ansible.builtin.fail:
        msg: "Only {{ gitea_supported_forks }} are supported."
```

``vars/main.yml``
```yaml=13
gitea_fork_variables:
  files:
    - "fork_{{ gitea_fork | lower }}.yml"
  paths:
    - 'vars'
```

----

``tasks/main.yml``

```yml=
- name: Gather Gitea/Forgejo UI Theme variables
  ansible.builtin.include_vars:
    file: "{{ lookup('ansible.builtin.first_found', params) }}"
  vars:
    params:
      files:
        - "{{ gitea_fork }}.yml"
      paths:
        - "defaults"
```

``defaults/gitea.yml``
```yml= 
---
gitea_theme_default: "gitea-auto"
gitea_themes: "gitea-auto,gitea-light,gitea-dark"
```

---

**Version Ermitteln**

``tasks/set_gitea_version.yml``
```yml=
---
- name: "Check gitea installed version"
  ansible.builtin.shell: |
    set -eo pipefail
    {{ gitea_full_executable_path }} -v | cut -d' ' -f 3
  args:
    executable: /bin/bash
  register: gitea_active_version
  changed_when: false
  failed_when: false
```

----

``tasks/set_gitea_version.yml``
```yml=13
- name: "Get latest gitea release metadata"
  ansible.builtin.uri:
    url: https://api.github.com/repos/go-gitea/gitea/releases/latest
    return_content: true
  register: gitea_remote_metadata
  become: false
  when: not ansible_check_mode
```
```yml=27
- name: "Set fact latest gitea release"
  ansible.builtin.set_fact:
    gitea_remote_version: "{{ gitea_remote_metadata.json.tag_name[1:] }}"
  when: not ansible_check_mode
```

----

``tasks/set_gitea_version.yml``
```yml=35
- name: "Set gitea version target (latest)"
  ansible.builtin.set_fact:
    gitea_version_target: "{{ gitea_remote_version }}"
  when: not ansible_check_mode
```
```yml=40
- name: "Set gitea version target {{ gitea_version }}"
  ansible.builtin.set_fact:
    gitea_version_target: "{{ gitea_version }}"
  when: gitea_version != "latest"
```

----

``tasks/set_gitea_version.yml``
```yml=45
- name: 'Assert that remote version is higher'
  ansible.builtin.assert:
    that:
      - gitea_active_version is version(gitea_remote_version, 'lt')
    fail_msg: ERROR - Remote version is lower then current version!
  when: gitea_version == "latest" and gitea_active_version.stderr == ""
```

---

**Create Secrets**
``tasks/gitea_secrets.yml``
```yml=1
---
- name: Generate gitea SECRET_KEY if not provided
  become: true
  ansible.builtin.shell: |
    umask 077
    gitea generate secret SECRET_KEY > {{ gitea_conf }}/gitea_secret_key
  args:
    creates: '{{ gitea_conf }}/gitea_secret_key'
  when: gitea_secret_key | string | length == 0
```

----

``tasks/gitea_secrets.yml``
```yml=9
- name: Read gitea SECRET_KEY from file
  become: true
  ansible.builtin.slurp:
    src: '{{ gitea_conf }}/gitea_secret_key'
  register: remote_secret_key
  when: gitea_secret_key | string | length == 0
```
```yml=16
  - name: Set fact gitea_secret_key
  ansible.builtin.set_fact:
    gitea_secret_key: "{{ remote_secret_key['content'] | b64decode }}"
  when: gitea_secret_key | string | length == 0
```

---

Configure Gitea
``tasks/configure.yml``
```yml=10
- name: "Configure gitea"
  become: true
  ansible.builtin.template:
    src: 'templates/gitea.ini.j2'
    dest: "{{ gitea_conf }}/gitea.ini"
    owner: "{{ gitea_user }}"
    group: "{{ gitea_group }}"
    mode: '0640'
  notify: "systemctl restart gitea"
```

----

``handlers/main.yml``
```yml=
---                                                    
- name: "Restart gitea"
  become: true
  listen: "systemctl restart gitea"
  ansible.builtin.systemd_service:
    name: gitea
    state: restarted
  when: ansible_service_mgr == "systemd"
```

---

### Warum sind Collections besser als Rollen?
* einfacher zu teilen
* schöner zu updaten
* besser integrtion in Ansible Galaxy
* mehr untergliederungsmöglichkeiten

----

##### Ansible Rolle Updaten:

<pre><code class="sh hljs">ansible-galaxy <span class="hljs-keyword">role</span> <span class="hljs-title">install</span> -r requirements.yml <span style="color: red">--force</span>
</code></pre>
##### Ansible Collection Updaten:
<pre><code class="sh hljs">ansible-galaxy <span class="hljs-keyword">collection</span> <span class="hljs-title">install</span> -r requirements.yml <span style="color: lawngreen">--upgrade</span>
</code></pre>

---

## Vielen Dank für euere Aufmerksamkeit
Habt ihr noch Fragen?
<img width="256px" style="border: none; background: rgba(0,0,0,0); box-shadow: none;" src="https://md.entropia.de/uploads/41128c4c-029d-49d6-b821-bc8ef1b3dce0.svg" />