# Explications – structure du repo Ansible (inventories, group_vars, rôles, Molecule)

Ce document explique les « briques » ajoutées dans ce dépôt Ansible et comment elles s’enchaînent.

## 1) Playbook vs Rôle (concepts)

### Playbook (`playbooks/patch_security.yml`)
Un **playbook** est un fichier YAML qui décrit **sur quels hôtes** exécuter Ansible et **quoi exécuter** (rôles, tâches, variables, etc.).

Dans `playbooks/patch_security.yml` :
- `hosts: all` : cible tous les hôtes présents dans l’inventaire actif.
- `become: true` : exécute avec élévation de privilèges (sudo).
- `roles: - patch_security` : lance le rôle `patch_security`.

### Rôle (`roles/patch_security/`)
Un **rôle** est un « package » réutilisable qui contient :
- des tâches (`tasks/`)
- des variables par défaut (`defaults/`)
- des handlers (`handlers/`)
- des templates (`templates/`)

Un rôle correspond généralement à une fonctionnalité : ici, **appliquer des patchs** + éventuel **reboot** + **rapport**.

---

## 2) Inventories : dev vs prod (`inventories/`)

Un **inventaire** est la liste des machines (et groupes de machines) sur lesquelles Ansible va agir.

### `inventories/dev/hosts.ini`
Contient un exemple de dev :

```ini
[all]
localhost ansible_connection=local
```

Interprétation :
- on cible `localhost`
- `ansible_connection=local` signifie « pas de SSH », exécution locale.

### `inventories/prod/hosts.ini`
Un gabarit à compléter avec tes serveurs de prod (souvent via SSH) :

```ini
[all]
# host1 ansible_host=10.0.0.10
```

### Comment Ansible choisit l’inventaire ?
Dans `ansible.cfg`, on a par défaut :

```ini
inventory = inventories/dev/hosts.ini
```

Donc, sans option, Ansible utilise l’inventaire **dev**.

Tu peux surcharger à la ligne de commande :

```bash
ansible-playbook -i inventories/prod/hosts.ini playbooks/patch_security.yml
```

---

## 3) `group_vars` : variables pour des groupes d’hôtes

Ansible charge automatiquement :
- `group_vars/<nom_groupe>/*.yml`

Ici on a :
- `group_vars/all/patch_security.yml`

Le groupe spécial `all` signifie : ces variables s’appliquent à **tous** les hôtes.

### Pourquoi c’est utile ?
Parce que tu peux configurer le comportement du rôle **sans modifier le code du rôle**.

Variables importantes :
- `patch_security_mode` :
  - `security` : tenter de n’installer que des updates sécurité (selon OS)
  - `all` : toutes les mises à jour (fallback/option)
- `patch_security_packages` : liste de paquets à mettre à jour (ciblage par paquets).
- `patch_security_allow_fallback_to_all` : autorise un fallback si « security only » n’est pas supporté.
- `patch_security_reboot` : `auto|always|never`.

---

## 4) Structure du rôle `patch_security`

### `roles/patch_security/tasks/main.yml`
C’est le point d’entrée des tâches du rôle.

Il fait principalement :
1. vérifie que l’OS est supporté,
2. initialise un rapport,
3. inclut des tâches spécifiques selon la famille d’OS :
   - `tasks/debian.yml` (Debian/Ubuntu via `apt`)
   - `tasks/redhat.yml` (RHEL-like via `dnf`)
   - `tasks/suse.yml` (SUSE via `zypper`)
4. décide si un reboot est requis,
5. déclenche un handler de reboot si demandé,
6. écrit un rapport.

### `roles/patch_security/defaults/main.yml`
Valeurs par défaut (si tu ne fournis pas de variables dans `group_vars`).

### `roles/patch_security/handlers/main.yml`
Un **handler** est une action déclenchée seulement si une tâche fait `notify`.

Ici : `Reboot system`.

### `roles/patch_security/templates/report.json.j2`
Template Jinja2 pour générer le rapport JSON.

Note idempotence : le rapport n’est écrit que si nécessaire afin que relancer le playbook ne marque pas “changed” en permanence.

---

## 5) Molecule : tests automatisés du rôle (`molecule/`)

### Pourquoi Molecule ?
Molecule sert à tester un rôle Ansible dans un environnement propre (souvent Docker) pour vérifier :
- **converge** : le rôle s’exécute sans erreur,
- **idempotence** : relancer ne provoque pas de changements inutiles,
- **verify** : certains invariants (fichiers, services…) sont corrects.

### Fichiers importants
- `molecule/default/molecule.yml` : configuration du scénario de test (driver docker, image, variables d’environnement…).
- `molecule/default/converge.yml` : playbook lancé par Molecule pour appliquer le rôle dans le conteneur.
- `molecule/default/verify.yml` : vérifications post-exécution.

### Idempotence
Molecule exécute `converge` une seconde fois et échoue si des tâches marquent encore `changed`.

---

## 6) Flow d’exécution (ce qui se passe quand tu lances Ansible)

Commande :

```bash
ansible-playbook playbooks/patch_security.yml
```

Ordre simplifié :
1. charge `ansible.cfg` (donc inventaire par défaut = dev)
2. lit l’inventaire (liste d’hôtes)
3. charge les variables `group_vars/all/patch_security.yml`
4. exécute le playbook
5. lance le rôle `patch_security`
6. le rôle choisit les tâches selon l’OS
7. applique les patchs selon `patch_security_mode` et/ou `patch_security_packages`
8. décide reboot selon `patch_security_reboot`
9. écrit un rapport.

---

## 7) Commandes utiles

### Exécuter en dev (local)
```bash
ansible-playbook playbooks/patch_security.yml --diff
```

### Audit (sans appliquer)
```bash
ansible-playbook playbooks/patch_security.yml --check --diff
```

### Utiliser l’inventaire prod
```bash
ansible-playbook -i inventories/prod/hosts.ini playbooks/patch_security.yml --diff
```

### Lancer les tests Molecule
```bash
molecule test
```

