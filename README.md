# ansible-dkim
A simple Ansible role for adding opendkim to postfix. The role will install postfix if it's not already present.


### Requirements

- Ubuntu 14 or CentOS 6

### Role variables
- `dkim_selector`

  Just use set this to the string `default` if you don't know what to do with this.


- `dkim_domains`

  A list of domains that you want to generate private DKIM keys for. Keys will not be overwritten once created.


- `dkim_wildcard_sign_all_with_domain`

  Optional. For servers that need to sign mail from domains that don't have their own key (ie. a web server with multiple virtual hosts on it).

  Caveat: If you use the "wildcard sign all" option, email recipients may see things like:

  ```
  From: sender@somedomain.com (via example.org)
  ```
  Where `somedomain.com` is the website that generated the email, but does not have its own DKIM key, and `example.org` is the domain key used to sign the outgoing message.


- `admin_email`

  Optional.


### Dependencies

If you're using CentOS, you will need to apply an opendkim SELinux policy separately, otherwise you will experience errors.


### Example playbook
```yaml
---
- hosts: myserver
  become: true
  roles:
    - role: AcroMedia.dkim
      dkim_selector: default
      dkim_domains:
       - example.org
       - domain2.tld
      dkim_wildcard_sign_all_with_domain: example.org
      admin_email: somebody@example.org

```
