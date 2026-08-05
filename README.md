Ansible Role: UCSC Genome Browser
=================================

Installs and configures the [UCSC Genome Browser](https://genome.ucsc.edu/),
a web-based tool for visualizing genomic data. It works on RHEL-based
distributions (see [meta/main.yml](./meta/main.yml) for details).

## Requirements

None.

## Role Variables

Ansible variables are listed below (except for
`ucsc_genome_browser_assemblies_env`), along with default values (see `defaults/main.yml`):

```yaml
ucsc_genome_browser_setup_script: https://raw.githubusercontent.com/ucscGenomeBrowser/kent/master/src/product/installer/browserSetup.sh
ucsc_genome_browser_setup_script_checksum: "{{ undef() }}"  # e.g. "sha256:e47cd21c479a53d3157a4cd54f867dc26420d5f825d0b6954257e3ee3054882d"

ucsc_genome_browser_offline_mode: false

ucsc_genome_browser_assemblies: all  # e.g. ["mm10", "wuhCor1"]
```

Use `ucsc_genome_browser_setup_script` to pin the installer script to a
specific version or to run a custom installer script. The variable
`ucsc_genome_browser_setup_script_checksum` can be used to verify the
integrity of the script; leave it blank to skip the verification.

The UCSC Genome Browser can operate in two modes: online and offline. In
online mode, data is loaded from UCSC when not present locally. In offline
mode, data is loaded only from the local MySQL database and file system. The
operation mode can be set using `ucsc_genome_browser_offline_mode`. The
default is the online mode.

The genome assemblies to install can be customized using the
`ucsc_genome_browser_assemblies` variable. By default, all available
assemblies are installed.

## Dependencies

None.

## Example Playbook

```yaml
- hosts: all
  roles:
    - role: usegalaxy_eu.ucsc_genome_browser
      become: true
```

## License

[GPLv3](LICENSE.md)

Please contribute back to this role by submitting pull requests or reporting issues.

## Author Information

This role was created in 2026 by [José Manuel Domínguez](https://github.com/domgz).
