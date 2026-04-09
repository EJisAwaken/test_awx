# Patchs de sécurité (Ansible)

Repo minimal pour **appliquer des patchs de sécurité de manière ciblée** (par familles d’OS / packages), avec reboot contrôlé et tests Molecule.

## Démarrage rapide

```bash
cd /home/ejhack/Bureau/pratique/projetSamedi
python -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
```

### Exécuter le playbook (inventaire dev)

```bash
ansible-playbook playbooks/patch_security.yml --diff
```

### Mode audit (sans appliquer)

```bash
ansible-playbook playbooks/patch_security.yml --check --diff
```

## Test (Molecule)

```bash
molecule test
```

## Variables principales

Voir `group_vars/all/patch_security.yml`.

## Documentation (explications)

Voir `docs/EXPLICATIONS_STRUCTURE_ANSIBLE.md`.

## Guide complet du fonctionnement du patching

Voir `docs/GUIDE_COMPLET_PATCH_SECURITY.md`.

