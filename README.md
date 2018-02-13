# ansible-dkim
Ansible role for opendkim with postfix configuration on ubuntu

Also works for CentOS 6, **BUT** you will need to add an opendkim SELinux policy separately, otherwise the service will not restart.

### Example playbook
```yaml
---
- hosts: myserver
  user: root
  sudo: False
  roles:
    - role: sunfoxcz.dkim
      dkim_selector: mail
      dkim_domains:
       - domain1.tld
       - domain2.tld
```
