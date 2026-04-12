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

## Campagne complète (pré-checks, backup, patch, post-checks, collecte)

Le dépôt inclut maintenant un playbook d’orchestration : `playbooks/patch_campaign.yml`.

### Exécution (inventaire dev)

```bash
ansible-playbook playbooks/patch_campaign.yml --diff
```

### Variables de campagne

Voir `group_vars/all/patch_campaign.yml`.

- `campaign_serial` : déploiement par vagues (ex: `1`, `2`, `10%`)
- `campaign_services_required` : services à valider avant/après
- `campaign_backup_enabled` + `campaign_backup_paths` : sauvegarde tar.gz minimale avant patch
- `campaign_report_root` : répertoire local (control node) pour stocker les rapports de campagne (`./reports`)
