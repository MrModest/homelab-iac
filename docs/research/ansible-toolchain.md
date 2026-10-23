# Ansible toolchain facts for the generator

Research for [#4](https://github.com/MrModest/homelab-iac/issues/4). Facts and trade-offs only, no decision.
Checked 2026-10-23 against PyPI, Galaxy, collection source (installed tarballs), ansible-core source, `ansible/ansible-documentation`, `docker/compose` source, and local runs (see §7).

## 1. Versions

| Package | Current | Released | Requirement |
|---|---|---|---|
| `ansible-core` | **2.21.5** | 2026-10-05 | controller Python `>=3.12` |
| `ansible` (community package) | **14.5.0** | 2026-10-06 | pins `ansible-core 2.21.5`, Python `>=3.12` |
| `community.docker` | **5.4.0** | 2026-10-04 | `requires_ansible: '>=2.17.0'`; `docker_compose_v2` needs Docker CLI + Compose plugin `>=2.18.0` |
| `community.general` | **13.5.0** | 2026-10-05 | `requires_ansible: '>=2.18.0'` |
| `ansible.utils` | **6.1.1** | 2026-09-17 | `requires_ansible: ">=2.16.0"`; `jsonschema` engine needs the `jsonschema` Python lib on the controller |

- `ansible 14.5.0` bundles exactly `community.docker 5.4.0`, `community.general 13.5.0`, `ansible.utils 6.1.1` ([ansible-14.5.0.deps](https://github.com/ansible-community/ansible-build-data/blob/main/14/ansible-14.5.0.deps)). So `ansible` alone covers all three; `ansible-core` + `requirements.yml` is the slimmer alternative.
- `community.docker` 5.0.0 made all doc fragments, module utils and plugin utils **private** ("can change at any time, … even in bugfix releases"), so nothing in the generator should import `community.docker` module_utils (CHANGELOG v5.0.0).
- `community.docker` adjusted `docker_compose_v2` event parsing for Docker Compose 5.0.0 (CHANGELOG, PR #1219). Latest Docker Compose is v5.6.1 (2026-10-21, GitHub releases API).
- ansible-core 2.19 (inherited by 2.21) rewrote templating: data tagging, lazy containers and an inverted template trust model ([porting guide 2.19](https://github.com/ansible/ansible-documentation/blob/devel/docs/docsite/rst/porting_guides/porting_guide_core_2.19.rst)). Several facts below depend on it.

Sources: PyPI JSON API, Galaxy v3 API (`highest_version`), each collection's `meta/runtime.yml` and `CHANGELOG.rst`.

## 2. `community.docker.docker_compose_v2`

Read from `plugins/modules/docker_compose_v2.py` and `plugins/doc_fragments/_compose_v2.py` in 5.4.0.

### Recreate decision

- The module does **not** diff anything itself. `state: present` runs `docker compose up --detach --no-color --quiet-pull [flags] -- [services]` and derives `changed` from Compose's event stream.
- `recreate`: `auto` (default, no flag) / `always` → `--force-recreate` / `never` → `--no-recreate`.
- Compose decides recreation in `mustRecreate` ([pkg/compose/reconcile.go](https://github.com/docker/compose/blob/main/pkg/compose/reconcile.go)): recreate if the parent namespace container was recreated, the container's config-hash label differs from the expected hash, the image digest differs, the image-volume digest differs, a running container's networks mismatch, or volumes mismatch.
- The config hash is `sha256(json(ServiceConfig))` with `build`, `pull_policy`, `scale`, `deploy.replicas`, `depends_on`, `profiles` trimmed out ([pkg/compose/hash.go](https://github.com/docker/compose/blob/main/pkg/compose/hash.go)). It hashes the **service definition**, not file contents on disk.
- Consequence: changing the **content** of a bind-mounted file (or anything under a bind-mounted dir) never triggers a recreate. Changing the compose service definition (env, labels, image tag, mounts list, …) does.
- An env file referenced by `env_file:` is resolved into `environment` when Compose loads the project, so its values land in the hash (standard Compose loading; not separately tested here).

### `build`

- `build: policy` (default, no flag) / `always` → `--build` / `never` → `--no-build`.
- `ignore_build_events: true` (default, since 4.2.0): with `build: always`, a rebuild that does not restart a container is **not** reported as `changed`. Compose `>=2.31.0` emits build events; older behaves as if `true`.
- `build:` is excluded from the config hash; a rebuilt image is picked up through the image digest check.

### `--check` / `--diff`

- `check_mode: full`. In check mode the module appends `--dry-run` to the same Compose command and parses `DRY-RUN MODE -` events. Pulling an image in check mode is not reported as changed.
- `diff_mode: none`. The module returns no diff. A `--diff` run shows changes only from tasks that write files (e.g. `template`/`copy` of `compose.yml`).
- Check-mode interaction (derived, not run): in `--check`, a preceding `template` task does not write the new `compose.yml`, so the dry-run `compose up` sees the **old** file on disk. The dry run then reports what the old file would do, not the new one. `definition:` (inline dict, needs PyYAML and `project_name`) avoids this because the module writes it to a temp dir per run, but then no `compose.yml` stays on the server.
- Return values: `containers`, `images`, and `actions` (list of `{what, id, status}`; `status` in `Starting, Exiting, Restarting, Creating, Stopping, Killing, Removing, Recreating, Pulling, Building`). `actions` lets later tasks see which services were recreated.

### Restarting specific services after a bind-mounted config changed

- `state: restarted` runs `docker compose restart [--no-deps] [--timeout N] -- <services>`. Not idempotent (always changed).
- `services: [a, b]` limits it to those services.
- `dependencies` defaults to **`true`**; without `dependencies: false` the restart also includes linked (`depends_on`) services.
- Typical pattern: register the file task, then `docker_compose_v2: {project_src, state: restarted, services: [...], dependencies: false}` when changed (or a handler). Skipping services that the `up` in the same run already recreated needs the `actions` return value.
- Alternative module: `community.docker.docker_container` with `name` + `restart: true` (not reviewed in detail here).

## 3. `ansible.builtin.validate_argument_spec` on nested data

Run against ansible-core 2.21.5 (§7, `vas.yml`). Source: `ansible/plugins/action/validate_argument_spec.py`, `ansible/playbook/role/__init__.py`, [role argument spec docs](https://github.com/ansible/ansible-documentation/blob/devel/docs/docsite/rst/playbook_guide/playbooks_reuse_roles.rst).

| Need | Expressible? | Observed |
|---|---|---|
| Fixed nested dicts | Yes, `type: dict` + `options` (recursive) | ✓ |
| List of dicts | Yes, `type: list, elements: dict, options` | ✓ |
| **Map keyed by arbitrary names** (`stacks.<slug>`, `containers.<name>`) | **No.** `type: dict` without `options` accepts any value; with `options` the keys must be known in advance. No `additionalProperties`/`patternProperties` equivalent | `stacks: {immich: 42, other: "not a dict"}` passed |
| Unknown keys inside `options` | Rejected | `containers.typo_key. Supported parameters include: …` |
| `mutually_exclusive`, `required_one_of`, `required_if` (also `required_together`, `required_by`) inside a nested option | Yes, on the option that has `options` | all three fired |
| Same keys at role entry-point level (`meta/argument_specs.yml`) | **No.** Role validation passes only `argument_spec.get('options', {})` to the action, so entry-point-level `mutually_exclusive` etc. are dropped. They work inside nested options. The docs list for role specs does not mention them | source |
| `choices`, `type` coercion | Yes | `port: "8080"` accepted as `int` |
| Ranges, regex/pattern on values or names, uniqueness, cross-references (e.g. `expose` must name an existing container) | No | — |
| Error location | Path without list index or map key: `missing required arguments: name found in containers`, `… found in containers -> expose` | which list element failed is not reported |
| All errors at once | Yes, list in `argument_errors` | 7 errors in one run |
| Coerced values / defaults returned | **No.** The action returns only `msg`, `argument_errors`, `argument_spec_data`, `validate_args_context`. Docs: "at runtime roles receive the original uncoerced values"; defaults come from `defaults/main.yml`, not the spec | result keys were only `changed, failed, msg, validate_args_context` (+ loop keys) |
| Extra top-level vars | Ignored (only spec keys are pulled from task vars) | source (`get_args_from_task_vars`) |

Workaround for maps: convert to a list first (`dict2items`, or a filter) and validate `elements: dict`. Error messages then lose the slug as well.

### JSON Schema alternative (`ansible.utils.validate`, engine `ansible.utils.jsonschema`)

Run in §7 (`jsonschema.yml`), draft 2020-12, `jsonschema` 4.x in the controller venv.

- Supports drafts 3, 4, 6, 7, 2019-09, 2020-12 (2019-09/2020-12 need `jsonschema>=4`). Available as module, filter, lookup and test.
- Expresses what the arg spec cannot: `additionalProperties: {schema}` for maps keyed by names, `propertyNames: {pattern}` for slug format, `minimum/maximum`, `oneOf` (exclusive `image`/`build`), `if/then` (required-if), `additionalProperties: false` (unknown keys).
- Errors carry a full path including map keys: `stacks.immich.containers.web.expose.port: 70000 is greater than the maximum of 65535`, `stacks: 'Bad-Slug' does not match '^[a-z][a-z0-9_]*$'`.
- Messages for combinators are raw (`… is valid under each of {'required': ['build']}, {'required': ['image']}`).
- Cross-references between parts of the document still need code (neither tool does them).
- Costs: an extra collection and Python dependency; the schema is not readable by `ansible-doc -t role` (the role arg spec is, and doubles as docs).
- A Python filter that calls `jsonschema` directly (or hand-written checks) is the third option: same expressiveness, plus cross-references, testable with pytest, no `ansible.utils`.

## 4. Filter plugins: discovery and pytest

- **Discovery** ([developing_locally.rst](https://github.com/ansible/ansible-documentation/blob/devel/docs/docsite/rst/dev_guide/developing_locally.rst)): Ansible auto-loads plugins from `<type>_plugins/` next to the playbook (available to that playbook/playbooks in that dir) and from `<type>_plugins/` inside a role (available once the role runs). Configured paths: `ANSIBLE_FILTER_PLUGINS` / `filter_plugins` in `ansible.cfg`. Confirmed: `filter_plugins/stack.py` next to the playbook worked with no config (§7).
- **Shape**: a module with `class FilterModule` whose `filters()` returns `{name: callable}`. Raise `ansible.errors.AnsibleFilterError` for user-facing errors.
- **pytest outside Ansible**: import the module directly. `filter_plugins/` needs no `__init__.py` (namespace package) but must be on `sys.path`; `pytest.ini` with `pythonpath = .` (pytest ≥7) is enough. 3 tests passed (pytest 9.1.1) importing `from filter_plugins.stack import FilterModule, to_mounts` and asserting `AnsibleFilterError`. Only `ansible-core` is needed in the test venv, for `ansible.errors`.
- **Runtime input type differs from tests**: inside a play the filter received `ansible._internal._templating._lazy_containers._AnsibleLazyTemplateDict` (a `dict` subclass, `isinstance(x, dict)` is true) with `_AnsibleTaggedStr` keys. Nested values are templated lazily on access. Plain-dict tests cover the logic; anything checking `type(x) is dict` would behave differently.

## 5. RFC §3.9 (dict keys are not templated): **still holds**

- `mounts: {"{{ app_root }}/config": /config}` → key stays literally `"{{ app_root }}/config"` on 2.21.5 and 2.18.19 (§7).
- The 2.19+ lazy containers render values only (`_proxy_or_render_lazy_value(key, value)` in `ansible/_internal/_templating/_lazy_containers.py`); keys are never passed to the templar.
- Building a dict inside an expression does work: `{{ dict([[app_root ~ '/config', '/config']]) }}` → `{"/tank/apps/config": "/config"}`.
- No explicit statement in the official docs was found; this rests on source + experiment.

## 6. RFC §3.7 (self-reference fails with recursive loop): **no longer holds on ansible-core ≥2.19**

Same playbook (§7, `templating.yml`, `loading.yml`) on both versions:

| Case | 2.18.19 | 2.21.5 |
|---|---|---|
| Play var `stack: {slug: immich, root: "/tank/{{ stack.slug }}", settings: {db_host: "{{ stack.slug }}_database"}}`, access `stack.root`, `stack.settings.db_host`, or whole `stack` | **fails**: `recursive loop detected in template string: /tank/{{ stack.slug }}. maximum recursion depth exceeded` | **works**: `/tank/immich / immich_database` |
| `sibling: {a: 1, b: "{{ sibling.a + 1 }}"}` | fails (same error) | works, `b: 2` (native int) |
| Same file loaded with `include_vars: {file: stack.yml, name: stack}` | fails | works |
| Real cycle `x: "{{ self_loop.y }}"`, `y: "{{ self_loop.x }}"` | fails | fails: `Recursive loop detected in template: maximum recursion depth exceeded` |
| `set_fact: {other: {slug: x, root: "/tank/{{ other.slug }}"}}` | fails: `'other' is undefined` | fails: `'other' is undefined` (var does not exist yet while its own args are templated) |

- Cause: the 2.19 "Lazy templating" change: "The new template engine lazily defers nearly all recursion and templating until values are accessed" (porting guide 2.19). Only the key actually accessed is rendered, so a sibling reference no longer re-renders the whole parent.
- The RFC's reason for loading `settings` as a separate variable therefore applies only to ansible-core ≤2.18. True cycles and `set_fact` self-reference still fail.

### Related: template trust (2.19+)

- Strings from playbooks, vars files and `include_vars` are trusted and rendered. Strings produced at runtime (e.g. `lookup('file', 'stack.yml') | from_yaml`) are **untrusted**: `{{ … }}` inside them is returned as-is, silently by default. Observed: `"/tank/{{ stack.slug }} / {{ stack.slug }}_database"` (2.18 behaved the same here because lookup results were already marked unsafe).
- `_ANSIBLE_TEMPLAR_UNTRUSTED_TEMPLATE_BEHAVIOR=warning|error` surfaces this; plugins can opt in with `ansible.template.trust_as_template` (porting guide 2.19, "Template trust model inversion", "Applying template trust to individual values").
- So whether templates inside `stack.yml` are rendered depends on how the catalog is loaded: `include_vars`/`vars_files` render them; `lookup(...) | from_yaml` does not.

## 7. Experiments

Throwaway venvs in a scratch dir (not committed):

```sh
uv venv -p 3.12 venv && uv pip install -p venv/bin/python ansible-core==2.21.5 pytest jsonschema
uv venv -p 3.12 venv218 && uv pip install -p venv218/bin/python 'ansible-core>=2.18,<2.19'   # → 2.18.19
venv/bin/ansible-galaxy collection install -p colls community.docker:==5.4.0 community.general:==13.5.0 ansible.utils:==6.1.1
# ansible.cfg: [defaults] collections_path = colls
ANSIBLE_CONFIG=exp/ansible.cfg venv/bin/ansible-playbook -i localhost, -c local exp/<playbook>.yml
venv/bin/python -m pytest -q -c exp/pytest.ini exp/tests     # 3 passed
```

`vas.yml` spec (abridged) and the failing input's output:

```yaml
containers:
  type: list
  elements: dict
  options: {name: {required: true}, image: {}, build: {}, kind: {choices: [app, db]}, backup: {type: dict},
            expose: {type: dict, options: {port: {type: int, required: true}}}}
  mutually_exclusive: [[image, build]]
  required_one_of: [[image, build]]
  required_if: [[kind, db, [backup]]]
```

```text
parameters are mutually exclusive: image|build found in containers
kind is db but all of the following are missing: backup found in containers
missing required arguments: name found in containers
missing required arguments: port found in containers -> expose
value of kind must be one of: app, db, got: nope found in containers
argument 'port' is of type str found in 'containers -> expose'. and we were unable to convert to int: "'abc'" cannot be converted to an int
containers.typo_key. Supported parameters include: backup, build, expose, image, kind, name.
```

Filter type probe inside a play: `{'outer': 'ansible._internal._templating._lazy_containers._AnsibleLazyTemplateDict', 'key': '_AnsibleTaggedStr', 'is_dict': True}`.

## 8. Trade-offs observed

- **Validation**: `validate_argument_spec` is built in and the spec doubles as role docs, but cannot type maps keyed by names (the shape of `stacks` and containers in the RFC), reports no list index/key in errors, and returns no coerced values. JSON Schema covers maps, patterns and ranges with precise paths, at the cost of `ansible.utils` + `jsonschema` and a second schema format. A Python filter can do either plus cross-references.
- **Check mode**: `docker_compose_v2` gives a real `--dry-run` but no diff, and the dry run reads whatever `compose.yml` is on disk at that moment.
- **Bind-mounted config changes** never recreate containers by themselves; an explicit `state: restarted` (with `dependencies: false`) is needed.
- **Self-reference**: on the current toolchain a stack's nested keys can reference siblings; the separate-`settings` workaround is only needed if ansible-core ≤2.18 must be supported.

## 9. Not verified

- No Docker on the research machine: recreate/restart/dry-run behaviour is from module and Compose source, not from a run. That includes the check-mode "old file on disk" interaction and `env_file` values entering the config hash.
- Whether top-level Compose `configs:`/`secrets:` with `file:` trigger a recreate when the file **content** changes. `hash.go` hashes the service's `configs`/`secrets` references; content handling elsewhere in Compose was not traced.
- `community.docker.docker_container` restart semantics were not reviewed.
- No official doc statement that dict keys are not templated (source + experiment only).
