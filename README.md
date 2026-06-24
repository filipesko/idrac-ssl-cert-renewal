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
- Network access from the controller to a Windows enrollment client over WinRM.
- Network access from the Windows enrollment client to the AD CS Web Enrollment
  `/certsrv` site.
- An AD CS certificate template, such as `WebServer`, configured to issue
  server-authentication certificates and accept the subject/SAN values supplied
  in the CSR.
- AD CS Web Enrollment with Windows authentication/NTLM enabled.

Install the required collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Configure

Copy the examples and edit them for your environment:

```bash
cp inventory/hosts.example.yml inventory/hosts.yml
cp inventory/group_vars/idrac.example.yml inventory/group_vars/idrac.yml
```

Recommended secret handling:

```bash
export IDRAC_USERNAME=root
export IDRAC_PASSWORD='...'
export ADCS_WINRM_PASSWORD='...'
export ADCS_WEB_USERNAME='CONTOSO\ansible_adcs_enroll'
export ADCS_WEB_PASSWORD='...'
```

You can also move secrets into Ansible Vault instead of environment variables.

Key settings in `inventory/group_vars/idrac.yml`:

- `adcs_web_enrollment_url`: AD CS Web Enrollment URL, for example
  `https://ca01.contoso.com/certsrv`.
- `adcs_template`: certificate template name, for example `WebServer`.
- `adcs_web_enrollment_host`: inventory name of the Windows host that runs the
  NTLM Web Enrollment HTTP requests.
- `adcs_web_username` and `adcs_web_password`: credentials used by the
  Windows enrollment client when posting the CSR to Web Enrollment. Leave both
  empty only if your WinRM session can delegate usable credentials to `/certsrv`.
- `adcs_web_validate_certs`: whether the Windows enrollment client validates
  the TLS certificate on the Web Enrollment site.
- `idrac_cert_*`: CSR subject fields.

Per-host overrides belong in `inventory/hosts.yml`, especially:

- `ansible_host`: iDRAC IP address or DNS name.
- `idrac_cert_common_name`: certificate common name.
- `idrac_cert_subject_alt_name`: DNS names and IP addresses for SANs.

The group variables live under `inventory/group_vars/` because the documented
run command uses `-i inventory/hosts.yml`. With that layout, Ansible loads vars
from the inventory directory.

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
- AD CS Web Enrollment is submitted with PowerShell `Invoke-WebRequest` on the
  Windows enrollment client because `ansible.builtin.uri` cannot authenticate to
  NTLM-only `/certsrv` sites.
- Explicit `adcs_web_username` and `adcs_web_password` avoid the WinRM
  double-hop problem. If you omit them, the PowerShell task uses the WinRM
  session credentials and your environment must allow those credentials to reach
  the AD CS Web Enrollment site.
- `idrac_validate_certs` defaults to `false` in the example because many iDRACs
  start with self-signed certificates. Set it to `true` and provide `idrac_ca_path`
  once your environment can validate the current iDRAC certificate.
- Importing the certificate resets the iDRAC by default so the HTTPS service
  starts using the new certificate. Set `idrac_reset_after_import: false` only if
  you plan to reset it separately.
