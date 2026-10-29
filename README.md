# php2shell

> "php2shell" is what the internet decided to call CVE-2026-109618 and CVE-2026-109619: two unauthenticated RCEs in PHP core that dropped right before a patch, because of course they did. If you're reading this, it's probably because someone in a Slack channel said "hey are we patched for that PHP thing" and nobody actually knew. This fixes that.

This is **not** an exploit, not a PoC, not a "one-click shell" scriptkiddies it's the boring cousin nobody invites to the security conference: it just reads `php -v` on every box and tells you if you're cooked. No payloads, no memory corruption, no crafted multipart requests nested 62 levels deep like some kind of regex-induced fever dream. Just version strings and a comparison table. Riveting stuff, but it won't get you fired.

---

## What it actually does

1. SSHes into every host in your farm (you already trust Ansible with this, so)
2. Looks for every PHP binary it can find (`php`, `php-fpm`, `php8.2`, `php-fpm8.3`, you get the idea)
3. Runs `php -v` like it's 2004
4. Compares the version against the patched builds (8.2.35 / 8.3.36 / 8.4.27 / 8.5.12 thanks PHP team, October 21 2026, we were all there)
5. Spits out a JSON verdict per host: `VULNERABLE`, `NOT VULNERABLE`, `UNKNOWN/EOL`, or `PHP NOT FOUND` (lucky you)

No network probes, no "let's see if we can OOM the worker," none of that. If your InfoSec team asks "did this scanner touch prod," the honest answer is "it ran a version check a sysadmin could've typed manually, just... at scale, and without the 3am Slack panic."

---

## Three ways to run this, pick your fighter

| Method | For when |
|---|---|
| **Plain Ansible** | You have a VDI, SSH keys, and mild trust issues about dashboards |
| **AAP (Automation Platform)** | Your org pays for Ansible to have a UI and you're contractually obligated to use it |
| **Red Hat Satellite** | You already live in Satellite and refuse to open one more tool, understandable honestly |

All three run the exact same core playbook. The only thing that changes is who's clicking the buttons.

---

## Repo layout
```
php2shell/
├── playbooks/
│ └── scan.yml.yml # the actual brain of the operation
├── inventory/
│ └── inventory.ini # your hosts, plain and simple
├── aap/
│ ├── bootstrap_project.yml # sets up AAP so you don’t have to click 40 times
│ └── job_template_vars.yml # secrets go here, vault them, don’t be that person
├── satellite/
│ └── remote_execution_job.yml # thin wrapper so Satellite recognizes it
└── reports/ # where the receipts live, generated at runtime
```

---

## Method 1: Plain Ansible (the "I just want it to run" option)

```bash
cd php2shell-scan
mkdir -p reports
ansible-playbook -i inventory/inventory.ini playbooks/scan.yml.yml
```

Got hundreds of hosts and the patience of a caffeinated squirrel? Crank the parallelism:

```bash
ansible-playbook -i inventory/inventory.ini playbooks/scan.yml.yml -f 50
```

### Inventory example

```ini
# inventory/inventory.ini
[webservers]
web01.intranet.local
web02.intranet.local
web03.intranet.local ansible_user=deploy

[webservers:vars]
ansible_user=root
ansible_ssh_private_key_file=~/.ssh/id_rsa_farm
```

### Pre-flight check (always do this, trust issues are healthy)

```bash
ansible -i inventory/inventory.ini all -m ping
```

If half your farm doesn't pong back, fix that before you blame the scanner.

---

## Method 2: Ansible Automation Platform

AAP doesn't vibe with "run a script from my laptop" energy, it wants a **Project**, an **Inventory**, and a **Job Template**, because enterprise software loves ceremony.

### 2a. The clicky UI way

1. **Projects** → New → point it at the git repo with this playbook
2. **Inventories** → New → your farm (or sync from wherever your source of truth lives)
3. **Credentials** → New → SSH machine credential
4. **Job Templates** → New → playbook `playbooks/scan.yml.yml`, pick the inventory + credential from above
5. **Launch** and go refill your coffee

### 2b. The infra-as-code way (for people who hate clicking)

```bash
ansible-galaxy collection install ansible.controller
ansible-playbook aap/bootstrap_project.yml --ask-vault-pass
```

This provisions the Project, Inventory, and Job Template via API so you never have to touch the UI again. You're welcome.

**Heads up**: AAP execution environments are ephemeral, the `reports/` folder doesn't survive between runs unless you push it somewhere persistent (S3, NFS, commit back to git, whatever your org's flavor of "durable storage" is). Add that as a final task if you care about history.

---

## Method 3: Red Hat Satellite (Remote Execution)

Satellite runs Ansible via Remote Execution jobs. Import the playbook as Ansible content, then fire it at a host collection like you're launching a very polite missile.

### 3a. Import the content

```bash
hammer import-templates \
  --repo "https://git.intranet.local/infra/php2shell-scan.git" \
  --organization "Your_Org"
```

### 3b. Register the Job Template

```bash
hammer job-template create \
  --name "php2shell Version Scan" \
  --file satellite/remote_execution_job.yml \
  --provider-type "Ansible" \
  --job-category "Compliance" \
  --organizations "Your_Org"
```

### 3c. Actually run it

```bash
hammer job-invocation create \
  --job-template "php2shell Version Scan" \
  --search-query "hostgroup = webservers" \
  --organization "Your_Org"
```

Or just click **Hosts → Schedule Remote Job** in the UI like a normal person.

---

## Reading the results

Every host drops a file: `reports/<hostname>_php2shell.json`

```json
[
  {
    "binary": "/usr/bin/php8.2",
    "version": "8.2.10",
    "status": "VULNERABLE",
    "detail": "Installed 8.2.10, fix is 8.2.35 (CVE-2026-109618, CVE-2026-109619)"
  }
]
```

Fastest way to find out which hosts need an immediate "sorry about your weekend" Slack message:

```bash
for f in reports/*.json; do
  grep -q '"status": "VULNERABLE"' "$f" && echo "VULNERABLE: $(basename "$f" _php2shell.json)"
done
```

## Oh and there's a CSV now too

Because nobody wants to `cat` forty JSON files to find out who's cooked. After the scan, the playbook automatically merges every `reports/*_php2shell.json` into one file: `reports/php2shell_summary.csv`

Columns: `host, binary, version, status, detail`. Open it in Excel, grep it, pipe it into whatever dashboard makes your manager happy. It's generated fresh on every run, so don't bother diffing it — just re-run the scan.

```bash
column -s, -t < reports/php2shell_summary.csv | less -S
```

**Heads up for AAP**: the CSV lands in the same ephemeral execution environment as the JSON files if you're persisting `reports/` to S3/NFS/git, persist this file along with it, same deal as before.

---

## Scheduling (because checking once and forgetting is how we got here)

- **Plain Ansible**: cron on the control node, like it's 1999 but it works
- **AAP**: native Schedules on the Job Template, very fancy
- **Satellite**: `hammer job-invocation create --cron-line "..."` or recurring jobs in the UI

---

## Troubleshooting

| Problem | Probable reason | 
|---|---|
| `UNREACHABLE` | SSH is sad, wrong user/key, or the box is actually on fire |
| No PHP found but you *know* it's there | Non-standard install path (looking at you, RHEL/Remi `php82-php`) add it to `php_binary_candidates` |
| AAP job runs fine but reports vanish | Told you, ephemeral filesystem, persist it somewhere |
| Satellite shows nothing in output | Check the content actually synced and `provider-type` is `Ansible`, not `SSH` |

---

## Final disclaimer nobody reads

This only tells you what version is installed. It doesn't patch anything, doesn't hug your sysadmins, and definitely doesn't replace actually reading the PHP security advisory. Patch your stuff. Go.

---

## License
```
        DO WHAT THE FUCK YOU WANT TO PUBLIC LICENSE 
                    Version 2, December 2004 

 Copyright (C) 2004 Sam Hocevar <sam@hocevar.net> 

 Everyone is permitted to copy and distribute verbatim or modified 
 copies of this license document, and changing it is allowed as long 
 as the name is changed. 

            DO WHAT THE FUCK YOU WANT TO PUBLIC LICENSE 
   TERMS AND CONDITIONS FOR COPYING, DISTRIBUTION AND MODIFICATION 

  0. You just DO WHAT THE FUCK YOU WANT TO.
```
