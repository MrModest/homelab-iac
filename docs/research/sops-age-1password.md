# SOPS + age via 1Password CLI in Ansible

Research for [#2](https://github.com/MrModest/homelab-iac/issues/2). Researched 2026-10-23. Facts and trade-offs only, no decisions.

## Versions at time of research

| Component | Latest stable | Date | Source |
|---|---|---|---|
| sops | v3.13.3 | 2026-07-23 | [getsops/sops releases](https://github.com/getsops/sops/releases) |
| community.sops | 2.5.0 | 2026-10-04 | [releases](https://github.com/ansible-collections/community.sops/releases), `galaxy.yml` |
| ansible-core | 2.21.5 (2.22.0rc1 out 2026-10-13) | 2026-10-05 | [ansible/ansible releases](https://github.com/ansible/ansible/releases) |

## 1. community.sops and the age key

Three ways to consume encrypted vars, all running on the **controller**:

- **vars plugin** `community.sops.sops`: auto-loads `*.sops.yaml|yml|json` from `group_vars/` and `host_vars/` (extensions configurable via `valid_extensions`). Caches decrypted files per run (`cache: true` default). Options can be set via `ansible.cfg` `[community.sops]` section or env vars, **not** via inventory vars (its docs do not include the `ansible_variables` fragment). [source](https://github.com/ansible-collections/community.sops/blob/main/plugins/vars/sops.py)
- **`community.sops.load_vars`** action: like `include_vars` for a sops file. Runs fully on the controller, check_mode support "full", returns `_ansible_no_log: True` on its own result. On ansible-core 2.21+ it sets real variables (`return_method: auto` = `vars-only`); before 2.21 it returns `ansible_facts`. Options are task arguments only. [module](https://github.com/ansible-collections/community.sops/blob/main/plugins/modules/load_vars.py), [action](https://github.com/ansible-collections/community.sops/blob/main/plugins/action/load_vars.py)
- **lookup** `community.sops.sops`: returns raw decrypted file content (or `extract=` a path). Options via task args, vars (`sops_*`), env, or ini. [source](https://github.com/ansible-collections/community.sops/blob/main/plugins/lookup/sops.py)

**How the key is located.** community.sops does not read keys itself. It runs the `sops` binary with `os.environ.copy()` plus env vars mapped from its options ([module_utils/sops.py](https://github.com/ansible-collections/community.sops/blob/main/plugins/module_utils/sops.py)):

| Option (since) | Sets env var | ini key `[community.sops]` | Ansible env var | Var (lookup only) |
|---|---|---|---|---|
| `age_key` (1.4.0) | `SOPS_AGE_KEY` | `age_key` | `ANSIBLE_SOPS_AGE_KEY` | `sops_age_key` |
| `age_keyfile` (1.4.0) | `SOPS_AGE_KEY_FILE` | `age_keyfile` | `ANSIBLE_SOPS_AGE_KEYFILE` | `sops_age_keyfile` |
| **`age_key_cmd` (2.3.0)** | **`SOPS_AGE_KEY_CMD`** | `age_key_cmd` | `ANSIBLE_SOPS_AGE_KEY_CMD` | `sops_age_key_cmd` |
| `age_ssh_private_key_cmd` (2.3.0) | `SOPS_AGE_SSH_PRIVATE_KEY_CMD` | `age_ssh_private_key_cmd` | `ANSIBLE_SOPS_AGE_SSH_PRIVATE_KEY_CMD` | `sops_age_ssh_private_key_cmd` |

Source: [doc_fragments/sops.py](https://github.com/ansible-collections/community.sops/blob/main/plugins/doc_fragments/sops.py); `age_key_cmd` added in 2.3.0 (2026-04-19) per [CHANGELOG](https://github.com/ansible-collections/community.sops/blob/main/CHANGELOG.md).

Because the parent env is inherited, simply exporting `SOPS_AGE_KEY_CMD` in the shell that runs `ansible-playbook` also works with any community.sops version; `age_key_cmd` just makes it configurable from Ansible.

**Supplying the key from 1Password.** `SOPS_AGE_KEY_CMD="op read op://<vault>/<item>/<field>"` works per run. Details from sops source ([age/keysource.go](https://github.com/getsops/sops/blob/main/age/keysource.go)):

- The command string is split with `shlex` and exec'd directly, **no shell**: no pipes, `$VAR` expansion or `&&`. Wrap in `sh -c '...'` if needed.
- Stdout is parsed as an age identity list (one `AGE-SECRET-KEY-1...` per line, `#` comments ignored). Output stays in memory; nothing is written to disk. `op read` prints a trailing newline by default (`-n` suppresses it); harmless for the parser.
- Since sops 3.12.0, the command gets `SOPS_AGE_RECIPIENT=<public key>` in its env, so one command can return the right key for each recipient.
- sops loads **all** configured sources (`SOPS_AGE_KEY`, `SOPS_AGE_KEY_FILE`, `SOPS_AGE_KEY_CMD`, plus `~/.config/sops/age/keys.txt` if present) and tries every identity. A stray default `keys.txt` would also be used.
- The command runs once per sops process per age stanza needing identities (identities are cached on the master key after first load). Each decrypted file is a new sops process, so expect **one `op read` per encrypted file per run** (vars plugin caches per file; `load_vars`/lookup calls are not cached across tasks).

Alternative: `age_key` / `SOPS_AGE_KEY` holding the key itself (e.g. `export SOPS_AGE_KEY=$(op read ...)`). Also avoids disk, but the key then sits in the environment of the `ansible-playbook` process and every child for the whole run (readable via `/proc/<pid>/environ` by the same user). community.sops 2.5.0 marks `age_key` as a secret on ansible-core 2.22+.

### 1Password CLI session behaviour (desktop-app integration)

From [app integration security](https://www.1password.dev/cli/app-integration-security/) and [app integration](https://www.1password.dev/cli/app-integration/):

- Linux needs PolKit and a running PolKit auth agent; unlock via system auth (fingerprint/password).
- "On macOS and Linux, authorization is confined to a terminal session but extends to sub-shell processes in that window." Session ID = current `tty` + start time. So `op` spawned by sops spawned by Ansible in the same terminal should reuse one authorization.
- Expires after 10 minutes of inactivity (refreshed on each use), hard limit 12 hours, or on lock / `op signout`.
- `op read <reference> [-n] [-o file --file-mode 0600]`. [reference](https://www.1password.dev/cli/reference/commands/read/)

**Not verified:** behaviour when Ansible runs without a tty (cron, CI) or in forked workers; headless use would need a 1Password service account (`OP_SERVICE_ACCOUNT_TOKEN`), not researched here.

## 2. sops version requirements

- `SOPS_AGE_KEY_CMD`: **sops 3.10.0** (released 2025-03-30), PR [#1811](https://github.com/getsops/sops/pull/1811). Also in 3.10.0: age SSH keys, age plugins, passphrase-protected identities.
- `SOPS_AGE_RECIPIENT` passed to the command and `SOPS_AGE_SSH_PRIVATE_KEY_CMD`: **sops 3.12.0**.
- Source: [sops CHANGELOG](https://github.com/getsops/sops/blob/main/CHANGELOG.md).
- **Older sops (≤3.9.x):** the age key source only reads `SOPS_AGE_KEY` and `SOPS_AGE_KEY_FILE` ([v3.9.4 keysource.go](https://github.com/getsops/sops/blob/v3.9.4/age/keysource.go)). `SOPS_AGE_KEY_CMD` is silently ignored, so decryption fails with "failed to load age identities" / no matching key (unless a default `keys.txt` exists). community.sops does **not** version-check `age_key_cmd`; it only sets the env var, so the failure surfaces as a sops decrypt error.
- community.sops detects the sops version (`sops --version --disable-version-check`) and switches CLI syntax for ≥3.9.0. `handle_unencrypted_files: skip|warn` needs sops ≥3.9.0.
- Note: sops 3.13.2 had a MAC bug with YAML lists containing comments, fixed in 3.13.3.

## 3. Encrypted YAML in git

From [first steps](https://github.com/getsops/docs/blob/main/content/en/docs/usage/first-steps/_index.md) and [reference](https://github.com/getsops/docs/blob/main/content/en/docs/reference/_index.md):

- **Keys stay plaintext, leaf values are encrypted** individually (AES256-GCM, one data key per file, unique IV per value). Key names are used as AEAD additional data, so renaming/moving a key breaks decryption of that value. Docs: "We expect that keys do not carry sensitive information, and keeping them in cleartext allows for better diff."
- A file-wide MAC over all values detects added/removed values. With `mac_only_encrypted: true` only encrypted values count toward the MAC.
- Shape:

```yaml
db:
    user: ENC[AES256_GCM,data:CwE4O1s=,iv:2k=,aad:o=,tag:w==]
    password: ENC[AES256_GCM,data:p673w==,iv:YY=,aad:UQ=,tag:A=]
sops:
    age:
        - recipient: age1...
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            ...
    lastmodified: "..."
    mac: ENC[AES256_GCM,...]
    version: 3.13.3
```

  (age stanza field names `recipient`/`enc` from [stores/stores.go](https://github.com/getsops/sops/blob/main/stores/stores.go).)
- Partial encryption (set in `.sops.yaml` creation rule or CLI; **at most one** of these per file):
  - `unencrypted_suffix`: keys ending with it stay plaintext, everything else encrypted.
  - `encrypted_suffix`: only keys ending with it are encrypted.
  - `unencrypted_regex` / `encrypted_regex`: same, by regex on key name.
  - `unencrypted_comment_regex` / `encrypted_comment_regex`: by a marker comment.
- Comments are encrypted by default ("Comments are always encrypted" under the suffix/regex options).
- `.sops.yaml` must be named exactly that (`.yml` is not auto-discovered).

## 4. `.sops.yaml` creation rules and multiple recipients

From [config file](https://github.com/getsops/docs/blob/main/content/en/docs/usage/identities/config-file/_index.md), [reference](https://github.com/getsops/docs/blob/main/content/en/docs/reference/_index.md), [age](https://github.com/getsops/docs/blob/main/content/en/docs/usage/identities/age/_index.md):

- `creation_rules` is a list; **first match wins**; a rule without `path_regex` matches everything (catch-all last).
- `path_regex` matches the file path relative to the `.sops.yaml`. sops finds `.sops.yaml` by walking up from the **current working directory**, not the file's directory; override with `--config` / `SOPS_CONFIG` (community.sops: `config_path`).
- Creation rules apply when **encrypting/creating** (and `sops updatekeys`). Decryption only uses the recipient metadata stored in the file itself.
- Multiple recipients: `age:` as a comma-separated string or YAML list. Any one listed private key decrypts the file (each recipient gets its own wrapped copy of the data key), so a second "recovery" age key works by listing it.

```yaml
creation_rules:
  - path_regex: stacks/.*\.sops\.yaml$
    encrypted_regex: ^(password|token|secret)$   # optional
    age:
      - age1primary...   # key held in 1Password
      - age1recovery...  # offline recovery key
```

- `key_groups` + `shamir_threshold` instead require N groups to cooperate (opposite of recovery). Not needed for "either key works".
- Adding/removing a recipient later: edit `.sops.yaml`, then `sops updatekeys <file>` per file (re-wraps the data key; needs a key that can currently decrypt).
- Docs: "the configuration file is ignored when KMS or PGP parameters are passed on the SOPS command line or in environment variables" (presumably same for `--age` / `SOPS_AGE_RECIPIENTS`; not verified).

## 5. `--check`, `--diff`, `no_log`

- Decryption is not a task: the vars plugin decrypts while building vars, so it runs the same under `--check`. `load_vars` has full check_mode support (it always runs). Both cost an `op read` even in check mode.
- `load_vars` result itself is `no_log`; it never prints the values. Plain variables from the vars plugin are not secret-tagged by default.
- **Leaks happen in consuming tasks**, not in sops: e.g. `template`/`copy` with `--diff` print before/after file content. Ansible docs: "Because the `--diff` option can reveal sensitive information, you can disable it for a task by specifying `diff: false`." ([check mode docs](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html))
- `no_log: true` on a task also suppresses the diff: callbacks get a censored result keeping only a fixed set of keys (`changed`, `warnings`, `attempts`, ...), and `diff` is not among them; the default callback's `v2_on_file_diff` reads `diff` from that censored result. Verified in ansible-core source on devel and `stable-2.21` ([executor/task_result.py](https://github.com/ansible/ansible/blob/stable-2.21/lib/ansible/executor/task_result.py), [_internal/_task.py](https://github.com/ansible/ansible/blob/devel/lib/ansible/_internal/_task.py), [callback/default.py](https://github.com/ansible/ansible/blob/devel/lib/ansible/plugins/callback/default.py)). Trade-off: `no_log` also hides useful output/errors; `diff: false` only hides the diff.
- ansible-core 2.22 (rc at time of writing) adds secret masking: community.sops 2.5.0 options `register_values_as_secrets` (vars plugin, `load_vars`; default false, marks *all* loaded values) and `register_result_as_secret` (lookup, `decrypt` filter; default true). Masked values are redacted wherever they appear in output. Not available on 2.21.

## Unverified / open

- `op` app-integration auth from Ansible forked workers or without a tty (only docs read, not tested).
- Exact error text from sops <3.10 when only `SOPS_AGE_KEY_CMD` is set (inferred from source, not run).
- 1Password service-account path for headless runs: not researched.
