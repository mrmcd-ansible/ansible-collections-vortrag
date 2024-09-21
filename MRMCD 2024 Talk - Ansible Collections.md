---
type: slide
tags: mrmcd
controls: true
title: MRMCD 2024 Talk - Ansible Collections
slideOptions:
  transition: slide
---

<img width="320x" style="border: none; background: rgba(0,0,0,0)" src="https://md.entropia.de/uploads/41128c4c-029d-49d6-b821-bc8ef1b3dce0.svg" />


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


<img width="420x" style="border: none; background: rgba(0,0,0,0); fill: white;" src="https://md.entropia.de/uploads/d3708399-b617-4405-9d6a-d4abe9fbd1b8.svg" />


----

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