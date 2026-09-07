# precudis

**Pre**-**cu**tover **dis**covery — a read-only Ansible playbook that collects a comprehensive baseline from any number of Palo Alto firewalls and Cisco core/access switches or stacks at a site. It gathers interfaces, HA state, routing, VPN state, neighbors, VLANs, trunking, spanning tree, MAC/ARP information, inventory, and running configurations.

Nothing is hardcoded for a specific site, and nothing is hardcoded for a specific *device count*. The playbook prompts for a site label, a comma-separated list of firewall management IPs, a comma-separated list of switch/stack management IPs, optional VLAN IDs, and credentials at runtime — enter one device of a type, ten, or none at all (as long as you enter at least one device overall).

Only Palo Alto and Cisco IOS are built in today. If your gear is Fortinet, Juniper, or something else, see [Adding a New Vendor](#adding-a-new-vendor-eg-fortinet-juniper) below — the playbook's structure is meant to be extended, not rewritten.

> Access points are not included. Cloud-managed Meraki AP information should be collected separately from the Meraki Dashboard.

## Repository Files

- `playbook.yml`: Main discovery playbook
- `inventory.yml`: Minimal localhost inventory used to register devices dynamically
- `vars.yml` (`group_vars/all/vars.yml`): Output directory setting
- `ansible.cfg`: Tested Ansible callback and connection settings
- `requirements.txt`: Required Python packages
- `requirements.yml`: Required Ansible collections
- `.gitignore`: Keeps discovery output and local Python environments out of version control

## macOS Installation

### 1. Open Terminal

Press **Cmd + Space**, type `Terminal`, and press **Enter**.

### 2. Download and extract the project

Place the archive in Downloads, then run:

```bash
cd ~/Downloads
unzip precudis.zip
cd precudis
```

If you cloned the GitHub repository, change into the cloned directory instead.

### 3. Verify Python 3

```bash
python3 --version
```

If Python 3 is unavailable, install the Xcode Command Line Tools:

```bash
xcode-select --install
```

### 4. Create and activate a virtual environment

```bash
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
```

The prompt should show `(ansible-venv)`. Activate it again in every new Terminal window before running the playbook.

### 5. Install Python dependencies

```bash
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
```

This installs:

- `ansible-core`
- `pan-python`
- `pan-os-python`
- `xmltodict`

The Palo Alto modules require the three PAN-OS-related Python packages. Installing only the Ansible collection is not sufficient.

### 6. Install Ansible collections

```bash
ansible-galaxy collection install -r requirements.yml
```

The project uses:

- `paloaltonetworks.panos`
- `cisco.ios`
- `ansible.netcommon`

### 7. Verify installation

```bash
which python3
which ansible
which ansible-playbook
ansible --version
ansible-galaxy collection list
```

When the virtual environment is active, the Python interpreter should normally point under:

```text
/Users/<username>/ansible-venv/bin/
```

Verify the PAN-OS dependencies directly:

```bash
python3 -c "import pan, panos, xmltodict; print('PAN-OS dependencies OK')"
```

## Ansible Configuration

The included `ansible.cfg` uses the current default callback with YAML-formatted results:

```ini
[defaults]
inventory = ./inventory.yml
host_key_checking = False
retry_files_enabled = False
stdout_callback = default
callback_result_format = yaml

[persistent_connection]
command_timeout = 60
connect_timeout = 60
```

This avoids the removed `community.general.yaml` callback error seen with newer Ansible versions.

## Validate the Playbook

Before connecting to devices, run:

```bash
ansible-playbook --syntax-check -i inventory.yml playbook.yml
```

Expected result:

```text
playbook: playbook.yml
```

## Run the Discovery

```bash
ansible-playbook -i inventory.yml playbook.yml
```

You will be prompted for:

1. Site short name
2. Firewall management IP(s) — comma-separated, any count, or blank if this run has none
3. Switch/stack management IP(s) — comma-separated, any count, or blank if this run has none
4. Comma-separated VLAN IDs, or blank to skip per-VLAN spanning-tree output
5. Firewall username and password (only asked if at least one firewall IP was entered — this credential is reused for every firewall IP in the list)
6. Switch username, login password, and enable password (only asked if at least one switch IP was entered — reused for every switch/stack IP in the list)

At least one firewall IP or one switch IP is required; entering both empty fails the run immediately with a clear message instead of silently doing nothing.

Passwords are masked and are not written to disk.

Example device prompts (two firewalls, two independent switches — e.g. a site with a non-stacked pair, or two separate closets):

```text
Site short name: troy
Palo Alto firewall management IP(s), comma-separated (leave blank if none this run): 172.31.97.3, 172.31.97.4
Cisco switch/stack management IP(s), comma-separated (leave blank if none this run): 172.31.97.5, 172.31.98.5
Comma-separated VLAN IDs for per-VLAN spanning-tree detail (leave blank to skip):
```

A single-firewall, single-switch site works the same way — just enter one IP in each list. A switch-only run (e.g. you only need switch data this time) works too: leave the firewall prompt blank and only the switch play runs; no firewall credentials are asked.

## Output

Each run creates a site-specific directory, with one subfolder per device — numbered in the order you typed the IPs, regardless of how many there are:

```text
discovery_output/
└── <site_name>/
    ├── <site_name>-fw-1/
    │   ├── system_info.txt
    │   ├── interfaces_all.txt
    │   ├── interfaces_hw.txt
    │   ├── ha_all.txt
    │   ├── ha_state.txt
    │   ├── ha_ha1.txt
    │   ├── ha_ha2.txt
    │   ├── routing_route.txt
    │   ├── routing_static.txt
    │   ├── vpn_ike_sa.txt
    │   ├── vpn_ipsec_sa.txt
    │   ├── vpn_flow.txt
    │   ├── lldp_neighbors.txt
    │   ├── arp_all.txt
    │   └── running-config.xml
    ├── <site_name>-fw-2/
    │   └── same firewall output set
    ├── ... <site_name>-fw-N/  (one per firewall IP entered)
    ├── <site_name>-switch-1/
    │   ├── show_switch.txt
    │   ├── show_switch_stack-ports.txt
    │   ├── show_inventory.txt
    │   ├── show_version.txt
    │   ├── show_interfaces_status.txt
    │   ├── show_interfaces_description.txt
    │   ├── show_vlan_brief.txt
    │   ├── show_interfaces_trunk.txt
    │   ├── show_interfaces_switchport.txt
    │   ├── show_etherchannel_summary.txt
    │   ├── show_cdp_neighbors_detail.txt
    │   ├── show_lldp_neighbors_detail.txt
    │   ├── show_mac_address-table.txt
    │   ├── show_ip_arp.txt
    │   ├── show_spanning-tree_summary.txt
    │   ├── show_power_inline.txt
    │   ├── show_interfaces_transceiver.txt
    │   ├── show_running-config.txt
    │   └── spanning_tree_vlan_<id>.txt
    └── ... <site_name>-switch-N/  (one per switch/stack IP entered)
```

Open the output in Finder:

```bash
open discovery_output
```

## Validation After a Run

Review the recap and confirm each device reports `failed=0`.

Check generated files:

```bash
find discovery_output/<site_name> -type f | sort
```

Search captured output for errors:

```bash
grep -RniE "ERROR:|Missing required library|FAILED" discovery_output/<site_name>
```

No output from the error search is the desired result.

## Compatibility Fixes Included

- Uses `stdout_callback = default` with `callback_result_format = yaml` instead of the removed YAML callback plugin.
- Installs `pan-python`, `pan-os-python`, and `xmltodict` through `requirements.txt`.
- Uses the more broadly compatible PAN-OS command `show arp` instead of `show arp all`.
- Omits the incomplete Cisco command `show interfaces port-channel`. The playbook retains `show etherchannel summary` for site-agnostic EtherChannel discovery.
- Continues past individual unsupported PAN-OS operational commands and writes their error text to the corresponding output file.

## Device-Specific Notes

- PAN-OS operational command support may vary by release. If a command is unsupported, its output file will contain the error while the remaining discovery continues.
- For a Cisco switch stack, enter the shared management IP once. `show switch` and `show switch stack-ports` identify the physical members.
- For independent, non-stacked switches, extend the playbook to register additional switch hosts.
- If an entered VLAN does not exist, the corresponding spanning-tree command may return an error without stopping the full run.

## Adding a New Vendor (e.g. Fortinet, Juniper)

This playbook ships with two vendor types out of the box — Palo Alto (`panos_op`) and Cisco IOS (`ios_command`) — but both follow the same three-part pattern, and a third vendor means repeating that pattern, not restructuring anything:

1. In the first play ("Collect site and device details"), add one more `vars_prompt` entry for that vendor's comma-separated IP list (mirror `fw_ips_csv` / `switch_ips_csv`), compute its `_list` the same way, then add an `add_host` loop that registers each IP into a new group with whatever `ansible_connection` / `ansible_network_os` that vendor's collection needs.
2. Add a brand new play, `hosts: <your_new_group>`, with its own `vars_prompt` for that vendor's username/password (and enable or admin-domain password if it needs one). This play is automatically skipped entirely if nobody entered an IP for that vendor — same as the existing plays.
3. Inside that play, list the vendor's read-only show/operational commands and loop a vendor-appropriate module over them, saving each result to its own file under `{{ discovery_output_dir }}/{{ site }}/{{ inventory_hostname }}/`, with `ignore_errors: true` per command so one unsupported command doesn't kill the run.

### Worked example: Juniper (Junos)

```yaml
# In "Collect site and device details":
    - name: juniper_ips_csv
      prompt: "Juniper device management IP(s), comma-separated (leave blank if none this run)"
      private: false
      default: ""
```

```yaml
    - name: Work out the Juniper IP list
      ansible.builtin.set_fact:
        juniper_ips_list: >-
          {{ (juniper_ips_csv.split(',') | map('trim') | reject('equalto', '') | list)
             if (juniper_ips_csv | default('') | length > 0) else [] }}

    - name: Register every Juniper device entered for this run
      ansible.builtin.add_host:
        name: "{{ site_name }}-juniper-{{ idx + 1 }}"
        groups: junos_devices
        ansible_host: "{{ item }}"
        ansible_connection: network_cli
        ansible_network_os: junipernetworks.junos.junos
        site: "{{ site_name }}"
      loop: "{{ juniper_ips_list }}"
      loop_control:
        index_var: idx
        label: "{{ item }}"
```

```yaml
# New play, added alongside the existing two:
- name: Juniper device discovery
  hosts: junos_devices
  gather_facts: false
  vars_prompt:
    - name: junos_username
      prompt: "Username for all Juniper devices entered above"
      private: false
    - name: junos_password
      prompt: "Password for all Juniper devices entered above"
      private: true
  vars:
    ansible_user: "{{ junos_username }}"
    ansible_password: "{{ junos_password }}"
    junos_commands:
      - "show version"
      - "show chassis hardware"
      - "show interfaces terse"
      - "show configuration"
      - "show route summary"
      - "show lldp neighbors"
      - "show ethernet-switching table"
  tasks:
    - name: Create per-host output directory
      ansible.builtin.file:
        path: "{{ discovery_output_dir }}/{{ site }}/{{ inventory_hostname }}"
        state: directory
      delegate_to: localhost

    - name: Run each show command
      junipernetworks.junos.junos_command:
        commands: "{{ junos_commands }}"
      register: junos_output
      ignore_errors: true

    - name: Save each command's output to its own file
      ansible.builtin.copy:
        content: "{{ item.stdout[0] | default('ERROR: ' ~ (item.msg | default('command failed'))) }}"
        dest: "{{ discovery_output_dir }}/{{ site }}/{{ inventory_hostname }}/{{ junos_commands[idx] | regex_replace('[ /]', '_') }}.txt"
      loop: "{{ junos_output.results }}"
      loop_control:
        index_var: idx
        label: "{{ junos_commands[idx] }}"
      delegate_to: localhost
```

Also add `junipernetworks.junos: ">=5.0.0"` to `requirements.yml`. The module (`junos_command`) and connection settings above are standard, documented `junipernetworks.junos` usage, but the specific command list is illustrative — adjust it to match what you actually need, the same way the Palo Alto and Cisco command lists here were refined against real devices rather than guessed once and left alone.

### Fortinet (FortiOS) — a pointer, not a worked example

FortiGate discovery doesn't map as cleanly onto the "loop of CLI show commands" pattern above. Ansible's `fortinet.fortios` collection is built mainly around FortiOS's REST API — config-object and monitor-fact modules such as `fortios_configuration_fact` and `fortios_monitor_fact` — rather than raw CLI output; raw `diagnose`/`get`/`show` commands over SSH are also possible via `ansible.netcommon.cli_command`, but which approach fits depends on exactly what you want captured. This hasn't been built or tested here, so rather than hand you unverified commands: check the current `fortinet.fortios` collection docs for the facts modules that match what you need, then follow the same three-part pattern above with a `hosts: fortios_devices` play.

## Safety

This project is read-only. It uses operational/show commands and configuration export functions. It does not push or commit configuration changes.

## Publishing to GitHub

Before pushing this to a public (or even private-but-shared) repository:

- **Never commit `discovery_output/`.** It contains live running-configs, VPN peer IPs, MAC/ARP tables, and other real network data from whatever site you ran the playbook against. The included `.gitignore` excludes it by default — don't `git add -f` around it.
- **No credentials are ever written to disk or to the repo.** Everything is prompted interactively at run time (`private: true` masks passwords in the terminal) — there's no vault file or inventory file with secrets to accidentally commit.
- Double-check `git status` before your first commit and any time you add a new site's data locally, to confirm nothing under `discovery_output/` is staged.
- Consider adding a LICENSE file (MIT, Apache-2.0, etc.) — none is included here since that choice is yours to make.

## Cleanup

Deactivate the virtual environment when finished:

```bash
deactivate
```
