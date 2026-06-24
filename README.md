# Dell iDRAC SSL certificate renewal with AD CS

This repository contains an Ansible playbook that renews Dell iDRAC HTTPS
certificates by:

1. Generating a CSR directly on the iDRAC.
2. Submitting the CSR to Microsoft AD CS Web Enrollment.
3. Installing the issued certificate back onto the iDRAC interface.

The iDRAC private key stays on the iDRAC because the CSR is generated there.

## Requirements

- Ansible controller with Python 3.
- Network access from the controller to each iDRAC.
- Network access from the controller to the AD CS Web Enrollment `/certsrv`
  site.
- An AD CS certificate template, such as `WebServer`, configured to issue
  server-authentication certificates and accept the subject/SAN values supplied
  in the CSR.
- AD CS Web Enrollment authentication that the Ansible controller can use:
  Basic over HTTPS, or Kerberos/GSSAPI with the Python `gssapi` library
  available to Ansible.

Install the required collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Configure

Copy the examples and edit them for your environment:

```bash
cp inventory/hosts.example.yml inventory/hosts.yml
cp group_vars/idrac.example.yml group_vars/idrac.yml
```

Recommended secret handling:

```bash
export IDRAC_USERNAME=root
export IDRAC_PASSWORD='...'
export ADCS_WEB_USERNAME='CONTOSO\ansible_adcs_enroll'
export ADCS_WEB_PASSWORD='...'
```

You can also move secrets into Ansible Vault instead of environment variables.

Key settings in `group_vars/idrac.yml`:

- `adcs_web_enrollment_url`: AD CS Web Enrollment URL, for example
  `https://ca01.contoso.com/certsrv`.
- `adcs_template`: certificate template name, for example `WebServer`.
- `adcs_web_username` and `adcs_web_password`: credentials used by the
  controller when posting the CSR to Web Enrollment.
- `adcs_web_validate_certs`: whether the controller validates the TLS
  certificate on the Web Enrollment site.
- `adcs_web_ca_path`: optional PEM CA bundle for validating the Web Enrollment
  site certificate.
- `idrac_cert_*`: CSR subject fields.

Per-host overrides belong in `inventory/hosts.yml`, especially:

- `ansible_host`: iDRAC IP address or DNS name.
- `idrac_cert_common_name`: certificate common name.
- `idrac_cert_subject_alt_name`: DNS names and IP addresses for SANs.

## Run

```bash
ansible-playbook -i inventory/hosts.yml playbooks/renew_idrac_ssl.yml
```

To renew a single iDRAC:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/renew_idrac_ssl.yml --limit server01-idrac
```

Generated CSRs and signed certificates are written under `artifacts/` on the
controller. The directory is ignored by git.

## Notes

- The playbook assumes AD CS auto-issues the request. If your template requires
  approval, Web Enrollment will not return an issued certificate download link
  and the playbook stops before importing anything to iDRAC.
- `ansible.builtin.uri` does not support NTLM authentication. If your `/certsrv`
  site only allows NTLM, either enable Basic over HTTPS, enable Kerberos/GSSAPI
  from the controller, or run this workflow from a Windows automation host with
  an implementation that can use Windows integrated credentials.
- `idrac_validate_certs` defaults to `false` in the example because many iDRACs
  start with self-signed certificates. Set it to `true` and provide `idrac_ca_path`
  once your environment can validate the current iDRAC certificate.
- Importing the certificate resets the iDRAC by default so the HTTPS service
  starts using the new certificate. Set `idrac_reset_after_import: false` only if
  you plan to reset it separately.
