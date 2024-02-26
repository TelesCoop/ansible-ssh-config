# Configure and update TelesCoop servers
This configuration file works for Ubuntu 22.
Run `sudo ls` then run `ansible-playbook server-config.yml` to configure ssh access
and global installation on TelesCoop servers.

Then run `ansible-playbook mail.yml` to configure server mail
to use Mailgun.
The `vault.key` is required only because the smtp password for
mailgun is stored. If you don't need mail configuration, you
can remove/comment the line `vault_password_file = vault.key`
in `ansible.cfg` and it will stop complainig about not found
key file.
