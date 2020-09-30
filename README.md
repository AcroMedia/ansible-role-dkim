# acromedia.ansible-role-dkim

Add OpenDKIM to Postfix.

This role is intended for use on web servers, so any mail they send out (order confirmations, blog digests, etc) has a better chance of making it to end users.


### Requirements

- Ubuntu >= 14 or CentOS >= 6
- Your role must gather facts
- If you're using RedHat / CentOS, you will need to apply an opendkim SELinux policy separately, otherwise you will experience errors.

### Dependencies

If postfix isn't present yet, the role will try and install it, but that's really just so the role can pass tests. Letting this role force the install of postfix isn't recommended.

### Example playbooks

See also: [Role Variables](#role-variables)

#### Single server: Let the role generate a private key for you

```yaml
---
- hosts: app-nodes
  become: true
  roles:
    - role: acromedia.postfix
      vars:
        default_mail_recipient: webmaster@example.com

    - role: acromedia.dkim
      vars:
        dkim_domains:
          - name: example.com
            selector: default
            private_key: ''  # Lets the role generate the private key.
            signing_table_pattern: '*@example.com'
        dkim_report_recipient: postmaster@example.com
```

#### Load-balanced servers: Upload a prepared private key to all servers

```yaml
---
- hosts: app-nodes
  become: true
  roles:
    - role: acromedia.postfix
      vars:
        default_mail_recipient: webmaster@example.com

    - role: acromedia.dkim
      vars:
        dkim_domains:
          - name: example.com
            selector: default
            # Use lookup() to set the value of `private_key` to be the decrypted content of a vault-encrypted template file
            private_key: "{{ lookup('file', 'templates/my-responsibly-vault-encrypted-private-key.j2') }}"  
            signing_table_pattern: '*@example.com'
        dkim_report_recipient: postmaster@example.com
```

### Role variables

Refer to [Exmaple Playbooks](#example-playbooks) for usage. See also: [defaults/main.yml](defaults/main.yml) for less frequently used variables.

#### dkim_domains[]:

- Defaults to an empty list.


- Whatever you specify here will be used by remote systems to construct a request for DNS TXT record. E.g. if your dkim selector is `foo`, and your domain is `example.com`, remote systems will look for the public DKIM key at `foo._domainkey.exmaple.com`.

- #### name

  - The domain name part of of email addresses that will have DKIM signatures added to.

  - The values for `.name` and `.selector` are used together to construct filename paths and write configuration templates.

- #### selector

  - The identifier on the domain name which specifies which DKIM public key was used to sign outgoing mail.

  - If you only use DKIM on one system, you can use the string `default`.

- #### private_key

  - The content of the RSA private key, as opposed to the filename.

  - If left blank, the role will use `opendkim-genkey` to create an RSA key pair for you.

  - The private key will be placed at`/etc/opendkim/keys/{{ dkim_domains.name }}/{{ dkim_domains.selector }}.private` on the server.

  - For single server systems, it's fine to let the role generate a key for you. If you have multiple app nodes, or use a load balancer, generate a private key ahead of time, and use that instead.

  - If you let the key pair be generated, the public half can be found in the form of a DNS text record at `/etc/opendkim/keys/{{ dkim_domains.name }}/{{ dkim_domains.selector }}.txt`. **Caveat**: You may need to massage the formatting of the generated public key record if you're adding it to AWS Route 53.

- #### signing_table_pattern

  - A string that tells opendkim which outgoing mail should be signed on its way out. Wilcards are supported. See http://www.opendkim.org/opendkim-README


#### dkim_report_recipient

An email address which specifies who should receive signature failure reports.


### How to generate a RSA key pair manually for use with DKIM
Install `opendkim-tools`, which provides `opendkim-keygen`. Use it like so:
```bash
opendkim-genkey \
 --selector=default \
 --domain=example.com \
 --bits=1024 \
 --append \
 --verbose \
 --directory=./
```
The above is equivalent to running `openssl genrsa -out ./default.private  1024 && openssl rsa -in ./default.private -pubout -out ./default.txt`, and then massaging the format of default.txt into a usable DNS record, but opendkim-genkey saves you the hassle of that last step.

### How to retreive your auto-generated public DKIM key from the server

If you let the role generate your DKIM key pair for you, (e.g. if didn't upload your private key), the public half will be on the server at:
```
/etc/opendkim/keys/{{ dkim_domain }}/{{ dkim_selector }}.txt
```
Its contents will look something like:
```
foo._domainkey	IN	TXT	( "v=DKIM1; k=rsa; "
	  "p=MIGfMx... truncated for readability ...XIDAQAB" )  ; ----- DKIM public key "foo" selector for "example.com" domain
```


### How to extract a public key from a private key
```
openssl rsa -in /path/to/private.key -pubout
```
will extract your public key, but you still need to massage it into the form of a DNS TXT record. E.g. you would need to convert:
```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA2QFd+mOaTavpOQAQi7jI
9Uo8K1C7NtJ6wMpDS0XA+KakPkNI6rehdg7mJxrXz7MD+mkFeahJtWwhOKTxLyXd
DQIDAQAB
-----END PUBLIC KEY-----
```
to:
```
default._domainkey 14400 IN TXT ("v=DKIM1; k=rsa; p="
"MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA2QFd+mOaTavpOQAQi7jI"
"9Uo8K1C7NtJ6wMpDS0XA+KakPkNI6rehdg7mJxrXz7MD+mkFeahJtWwhOKTxLyXd"
"DQIDAQAB")
```
### Caveats

1. DNS TXT records have an individual string size limit of 255 characters. To overcome this, multiple quoted strings must be concatenated to achieve a record longer than 255 characters. This will occur if you create a private key of more than 1024 bits. The record may end up something like this:

1. Not all DNS providers support strings longer than 255 chars. If this is the case, your key generation cannot use more than 1024 bits.

1. Not all DNS providers support the format of multiple lines inside braces. For example, AWS's Route 53 interface will *mangle* your DNS record like a boss if you don't strip those braces, newlines,  comments out, and paste it in all as a single line:
```
foo._domainkey.example.com. 3600 IN TXT "v=DKIM1; k=rsa;" "p=MIGfMx...xxxxx" "yyyyy...XIDAQAB"
````
