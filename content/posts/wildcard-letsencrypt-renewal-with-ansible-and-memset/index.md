---
title: "Wildcard LetsEncrypt renewal with Ansible and Memset"
date: "2018-08-07"
categories: 
  - "ansible"
  - "automation"
  - "letsencrypt"
---

# Obtaining a wildcard LetsEncrypt cert with Ansible

Earlier this year, LetsEncrypt made their wildcard x509 certificates available to the general public. Whilst this is a massive step forward over individual certificates for each domain, it does come with the overhead of having to distribute the wildcard certificate to the (possibly many) places you would use it. Ignoring that issue for now, I wrote a quick Ansible playbook which uses the `dns-01` challenge method and my Memset DNS management modules (available in Ansible 2.6+) to provide the verification.

Without further ado:

```
---
- hosts: localhost
  gather_facts: no

  vars:
    tmpdir: "/tmp/le/"
    account_key: "le-account-key.pem"
    keyname: "star-domain-com-key.pem"
    csrname: "star-domain-com.csr"
    certname: "star-domain-com.pem"
    fullchain: "star-domain-com-fullchain.pem"
    common_name: "*.domain.com"

  tasks:
  - name: localhost | create temp dir
    file:
      path: "{{ tmpdir }}"
      state: directory

  - name: localhost | create temp account key
    openssl_privatekey:
      path: "{{ tmpdir }}{{ account_key }}"

  - name: localhost | create private key
    openssl_privatekey:
      path: "{{ tmpdir }}{{ keyname }}"

  - name: localhost | create CSR
    openssl_csr:
      path: "{{ tmpdir }}{{ csrname }}"
      privatekey_path: "{{ tmpdir }}{{ keyname }}"
      common_name: "{{ common_name }}"
      country_name: GB
      organization_name: my-OU
      email_address: ops@domain.com

  - name: LetsEncrypt | submit request
    acme_certificate:
      account_key_src: "{{ tmpdir }}{{ account_key }}"
      account_email: me@domain.com
      src: "{{ tmpdir }}{{ csrname }}"
      fullchain_dest: "{{ tmpdir }}{{ certname }}"
      challenge: dns-01
      acme_directory: https://acme-staging-v02.api.letsencrypt.org/directory
      acme_version: 2
      terms_agreed: yes
      remaining_days: 60
    register: challenge

  - name: Memset | create DNS challenge record
    memset_zone_record:
      api_key: "{{ memset_dns_api_key }}"
      state: present
      zone: domain.com
      type: TXT
      record: "{{ challenge['challenge_data']['*.domain.com']['dns-01']['resource'] }}"
      data: "{{ challenge['challenge_data']['*.domain.com']['dns-01']['resource_value'] }}"

  - name: Memset | request DNS reload
    memset_dns_reload:
      api_key: "{{ memset_dns_api_key }}"
      poll: true

  - name: LetsEncrypt | retrieve cert
    acme_certificate:
      account_key_src: "{{ tmpdir }}{{ account_key }}"
      account_email: me@domain.com
      src: "{{ tmpdir }}{{ csrname }}"
      dest: "{{ tmpdir }}{{ certname }}"
      fullchain_dest: "{{ tmpdir }}{{ fullchain }}"
      challenge: dns-01
      acme_directory: https://acme-staging-v02.api.letsencrypt.org/directory
      acme_version: 2
      terms_agreed: yes
      remaining_days: 60
      data: "{{ challenge }}"
    register: cert_retrieval

  - name: Memset | delete DNS challenge record
    memset_zone_record:
      api_key: "{{ memset_dns_api_key }}"
      state: absent
      zone: domain.com
      type: TXT
      record: "{{ challenge['challenge_data']['*.domain.com']['dns-01']['resource'] }}"
      data: "{{ challenge['challenge_data']['*.domain.com']['dns-01']['resource_value'] }}"
    when: cert_retrieval is changed

  - name: localhost | remove the account key
    file:
      path: "{{ tmpdir }}{{ account_key }}"
      state: absent
    when: cert_retrieval is changed
```

## Points to note

I've deliberately used the staging endpoint provided by LetsEncrypt; the certs this issues won't be valid for use but it allows you to test your playbook without hitting the [account rate limits](https://letsencrypt.org/docs/rate-limits/).

The last two tasks cleanup the account key and DNS challenge record, but only if the certificate was successfully issued.

## Testing

```
$ openssl x509 -in star-domain-com.pem -noout -text | grep "Subject: CN"
        Subject: CN = *.domain.com
```

There we have it; one wildcard certificate from LetsEncrypt!
