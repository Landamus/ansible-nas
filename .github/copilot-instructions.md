# Repository guidance

## Setup and commands

- Install Ansible roles and collections with `ansible-galaxy install -r requirements.yml`.
- Install Python test tools with `python3 -m pip install -r requirements-dev.txt`.
- Install website dependencies from `website/` with `npm ci`.
- Run the full local role test suite with `bash tests/test.sh`. This invokes Molecule for every role under `roles/*/molecule/`.
- Run one role's Molecule scenario from the repository root:
  `cd roles/<role> && molecule -c ../../tests/molecule/base.yml test`
- Run the lint suite locally with `pre-commit run --all-files`. The underlying checks are `ansible-lint nas.yml`, `yamllint .`, and Markdownlint.
- Build the Docusaurus site with `cd website && npm run build`.
- Run the playbook against an inventory:
  `ansible-playbook -i inventories/<inventory>/inventory nas.yml`
  Use `--tags <role-or-group-tag>` to limit execution, as roles in `nas.yml` are tagged.
- CI runs Ansible lint, YAML lint, Markdownlint, and the Docusaurus build on pull requests. The integration workflow runs Molecule across roles containing a `molecule/` directory.

## Architecture

- `nas.yml` is the main playbook. It applies shared system roles, Docker setup, application stacks, and each enabled application role to hosts in the `all` group.
- `group_vars/all.yml` is the public catalog of defaults and shared NAS paths. Do not edit it for machine-specific configuration; put overrides in `inventories/<inventory>/group_vars/nas.yml`.
- `roles/` contains independent Ansible roles. Core roles set up users, filesystems/packages, Docker, and general host configuration; application roles deploy Docker containers.
- Shared dependencies are declared in `requirements.yml` (external Ansible roles and collections). Docker container operations use `community.docker.docker_container`.
- The `website/` directory is a Docusaurus site; application documentation lives under `website/docs/applications/` and contributor guidance under `website/docs/contributing/`.
- Ansible targets Ubuntu Server 22.04 LTS and runs fact caching from `/tmp/facts_cache` (`ansible.cfg`).

## Role conventions

- Add a new application by modeling it on `roles/hello_world/`, including:
  - `defaults/main.yml` for configurable variables;
  - `tasks/main.yml` for start and stop behavior;
  - `molecule/default/` tests using the repository Molecule base configuration;
  - documentation under `roles/<role>/docs/` and the matching Docusaurus page;
  - registration in `nas.yml` and the application list in `README.md`.
- Application defaults use prefixed variables, for example `hello_world_enabled`, `hello_world_available_externally`, `hello_world_data_directory`, `hello_world_hostname`, `hello_world_port`, `hello_world_memory`, and `hello_world_container_name`.
- Application tasks normally have two blocks: start the container and remove it when `<role>_enabled` is false. Container state, mounts, environment, ports, and restart policy are therefore controlled by role variables.
- Use `{{ docker_home }}/<role>` for role-owned persistent data. Use shared media and share variables such as `movies_root`, `tv_root`, `downloads_root`, and `books_root` for host data mounts.
- Containers use `restart_policy: unless-stopped`. External access is expressed with Traefik labels gated by `<role>_available_externally`; labels also commonly include Homepage metadata and `wud.watch: "true"` for updates.
- Molecule tests should verify both running and stopped container states:
  - `molecule/default/molecule.yml` enables the role by setting `<role>_enabled: true`;
  - `verify.yml` asserts the container is running;
  - `side_effect.yml` reruns the role with `<role>_enabled: false`;
  - `verify_stopped.yml` asserts the container is absent.
- Keep role-specific implementation changes isolated; update `nas.yml`, defaults, Molecule coverage, README/application documentation, and the Docusaurus page together for a new application.
