# Ansible Role: Teleport Node Service

[![Ansible Galaxy](https://img.shields.io/badge/Ansible%20Galaxy-mdsketch.teleport-blueviolet)](https://galaxy.ansible.com/mdsketch/teleport)
[![Ansible Lint](https://github.com/mdsketch/ansible-teleport/actions/workflows/lint.yml/badge.svg)](https://github.com/mdsketch/ansible-teleport/actions/workflows/lint.yml)
[![molecule_tests](https://github.com/mdsketch/ansible-teleport/actions/workflows/molecule.yml/badge.svg)](https://github.com/mdsketch/ansible-teleport/actions/workflows/molecule.yml)

An ansible role to install or update the teleport node service and teleport config on Linux based systems.

Works with any architecture that teleport has a binary for, see available [teleport downloads](https://goteleport.com/teleport/download/).

If you add your own teleport config file template you can run any node services you want (ssh, app, database, kubernetes).

Please Check the teleport config file [documentation](https://goteleport.com/docs/reference/config/) for more information and confirm it is setup correctly.

## TODO:
- add idempotence tests to verify teleport is updated correctly (config, service and binary)
- add tests for variable templating
- investigate if installing teleport in a docker container is useful (currently not supported)

## Requirements

A running teleport cluster so that you can provide the following information:

- auth token (dynamic or static). Ex: `tctl nodes add --ttl=5m --roles=node | grep "invite token:" | grep -Eo "[0-9a-z]{32}"`
- CA pin
- address of the authentication or proxy server

## Role Variables

These are the default variables with their default values as defined in `defaults/main.yml`

```
teleport_nodename: ""
```
The nodename to apply in the configuration.  Keep it as an empty string to let teleport use the hostname of the machine.

```
teleport_autodetect_version: false
```
Whether or not try to autodetect the server version by querying its API.

```
teleport_version
```
The version of teleport to install. See [teleport downloads](https://goteleport.com/teleport/download/) for available versions. Keep it as an empty string if you want the role to autodetect the server version.

```
teleport_architecture
```
Change `teleport_architecture` any of the following:

- `arm-bin` if you are running on ARMv7 (32-bit) based devices.
- `arm64-bin` if you are running on ARM64/ARMv8 (64-bit) based devices.
- `amd64-bin` if you are running on x86_64/AMD64 based devices.
- `386-bin` if you are running on i386/Intel based devices.

```
teleport_install_method: "tar"
```
The method used for installation, currently supported by the role:
- `tar` Download an archive.
- `apt` Install gravitational keyring and the packages requested via apt.

```
teleport_edition: "oss"
```
This is only used with teleport_install_method: "apt":
- `oss` if you are using the community edition.
- `enterprise` if you are using the self-hosted edition.
- `cloud` if you are using the cloud edition.

```
teleport_config_template
```
The template to use for the teleport configuration file. The default is `templates/default_teleport.yaml.j2`. It contains a basic configuration that will enable the SSH service and add a command label showing node uptime.

There are many [options available](https://goteleport.com/docs/setup/reference/config/) and you can substitute in your own template and add any variables you want.

```
teleport_ssh_labels
```
A list of list of key and values to template into the default teleport_config_template. Examples are shown as defaults above.

```
teleport_ssh_commands
```
A list of dictionaries to template into the default teleport_config_template. Examples are shown as defaults above.

```
teleport_ca_pin
```
The CA pin to use for the teleport configuration. This is optional, but [recommended](https://goteleport.com/docs/setup/admin/adding-nodes/#untrusted-auth-servers).

```
teleport_auth_server
```
The authentication server to use for the teleport configuration. Examples are shown as defaults above.

```
teleport_proxy_server
```
The proxy server to user for the teleport configuration. Examples are shown as defaults above.

```
backup_teleport_config
```
Runs a backup of the teleport configuration file before overwriting it. The default is `yes`. See [Upgrading Teleport](#upgrading-teleport) for more information.

```
teleport_control_systemd
```
Default `yes`. Controls if this role modifies the teleport service.

```
teleport_template_config
```
Default `yes`. Controls if this role modifies the teleport config file.

```
teleport_managed_updates
```
Default `yes`. Controls if this role enable managed updates. See [Agent managed updates](https://goteleport.com/docs/upgrading/agent-managed-updates/) for more information.

```
teleport_force_reinstall
```
Default `no`. Force reinstallation if teleport is already installed.

## Upgrading Teleport

If managed updates are enabled, teleport should update itself to your server version and no further action is done.

## Idempotence

When performing an upgrade, a backup of the current configuration file in `teleport_config_path` will be created and a new configuration file templated in its place. When doing this a `teleport_auth_token` and `teleport_ca_pin` do not need to be provided, as they are pulled from the existing configuration file, and then templated into the new configuration file.

This allows you to update values in the configuration file like labels and commands without having to store the auth token and ca pin.

## Reloads

This role reloads `teleport.service` after any of the following occur:

- Teleport is installed
- Teleport configuration file is updated
- Teleport service file is updated
- Teleport managed updates are activated and an update if performed (albeit this reload should be gracefully handled by `teleport-update`)

## Dependencies

None

## Example Playbook
For example to install teleport on a node:
```
- hosts: all
  roles:
    - mdsketch.teleport
  vars:
    # optional ssh labels
    teleport_ssh_labels:
      - k: "label_key"
        v: "label_value"
    teleport_ssh_commands:
      - name: hostname
        command: [hostname]
        period: 60m0s
      - name: uptime
        command: [uptime, -p]
        period: 5m0s
      - name: version
        command: [teleport, version]
        period: 60m0s
      - name: ip-address
        command: ["/bin/sh","-c", "hostname -I | awk '{print $1}'"]
        period: 60m0s
    teleport_auth_token: "super secret auth token"
    teleport_ca_pin: "not as secret ca pin"
    teleport_auth_server: "auth server"
```

*Created Teleport Config to `/etc/teleport.yaml`*

```
---
version: v3
teleport:
  auth_token: "super secret auth token"
  ca_pin: "not as secret ca pin"
  auth_server: auth server
  log:
    output: stderr
    severity: INFO
    format:
      output: text
  diag_addr: ""
ssh_service:
  enabled: "yes"
  labels:
    label_key: label_value
  commands:
    - name: hostname
      command: [hostname]
      period: 60m0s
    - name: uptime
      command: [uptime, -p]
      period: 5m0s
    - name: version
      command: [teleport, version]
      period: 60m0s
    - name: ip-address
      command: ["/bin/sh","-c", "hostname -I | awk '{print $1}'"]
      period: 60m0s
proxy_service:
  enabled: "no"
  https_keypairs: []
  https_keypairs_reload_interval: 0s
  acme: {}
auth_service:
  enabled: "no"
```

## License

MIT / BSD

## Author Information

This role was created in 2021 by Matthew Draws for [KPM Power](https://github.com/kpmteam/ansible-teleport)

## Maintainers
- Matthew Draws: [mdsketch](https://github.com/mdsketch)
