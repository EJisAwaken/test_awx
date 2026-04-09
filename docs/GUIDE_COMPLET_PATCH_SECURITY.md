# Guide complet – Playbook/Rôle `patch_security` (Ansible)

Objectif : documenter **tout le fonctionnement** du playbook `playbooks/patch_security.yml` et du rôle `roles/patch_security/` afin de comprendre :
- comment on cible les hôtes (inventories)
- comment les variables sont chargées (group_vars, defaults)
- ce que fait le rôle (Debian/RedHat/SUSE)
- le reboot (handlers)
- le reporting
- comment exécuter en prod (sudo, SSH, mots de passe / clés)
- comment tester (Molecule)

---

## 1) Vue d’ensemble : que fait ce projet ?

Ce dépôt permet d’appliquer des patchs système de manière « ciblée » :
- **par mode** (sécurité uniquement quand possible vs toutes mises à jour)
- **ou par liste de packages** (ex: ne patcher que `openssl`, `sudo`, etc.)

Il gère aussi :
- un **reboot contrôlé** (`auto|always|never`)
- un **rapport JSON** écrit sur la machine distante
- des **tests Molecule** pour valider le rôle dans un conteneur.

---

## 2) Arborescence et rôle des dossiers

- `playbooks/` : playbooks d’entrée (orchestration)
- `roles/patch_security/` : implémentation (tâches/handlers/templates)
- `inventories/` : hôtes ciblés (dev/prod)
- `group_vars/` : variables d’environnement (globales ou par groupe)
- `molecule/` : scénarios de tests automatiques
- `ansible.cfg` : configuration Ansible (inventaire par défaut, callback, etc.)

---

## 3) Inventaires : dev vs prod

### 3.1 `inventories/dev/hosts.ini`
Utilisé pour tester localement (sans SSH) :

```ini
[all]
localhost ansible_connection=local
```

### 3.2 `inventories/prod/hosts.ini`
Utilisé pour cibler un serveur distant.

Exemple (sans mot de passe en clair) :

```ini
[all]
host1 ansible_host=10.0.36.55 ansible_user=vboxuser

[all:vars]
# Recommandé: utiliser des clés SSH
# ansible_ssh_private_key_file=~/.ssh/id_rsa

# Pour sudo (become), exécuter avec --ask-become-pass
# ou stocker ansible_become_password avec Ansible Vault.
```

### 3.3 Pourquoi ton run en prod échouait (erreur sudo)
Ton erreur :

> `sudo: il est nécessaire de saisir un mot de passe`

Le playbook est lancé avec `become: true` (sudo). Sur la machine distante, sudo demande un mot de passe.

**Solution simple** (interactive) :

```bash
ansible-playbook -i inventories/prod/hosts.ini playbooks/patch_security.yml \
  --ask-pass \
  --ask-become-pass \
  --diff
```

- `--ask-pass` : demande le mot de passe SSH (si tu n’utilises pas de clé)
- `--ask-become-pass` : demande le mot de passe sudo

**Solution recommandée** : mettre une clé SSH et/ou autoriser sudo sans mot de passe pour l’utilisateur Ansible (selon politique de sécurité), ou utiliser Ansible Vault.

---

## 4) Configuration Ansible (`ansible.cfg`)

Dans `ansible.cfg` :

- `inventory = inventories/dev/hosts.ini` : inventaire par défaut
- `roles_path = roles` : où Ansible trouve les rôles
- `stdout_callback = ansible.builtin.default` + `result_format = yaml` : sortie lisible
- `become = True` : élévation sudo activée par défaut

Tu peux toujours surcharger l’inventaire avec `-i`.

---

## 5) Variables : defaults vs group_vars

### 5.1 `roles/patch_security/defaults/main.yml`
Ce sont les **valeurs par défaut** du rôle (utilisées si tu ne définis rien ailleurs).

### 5.2 `group_vars/all/patch_security.yml`
Ce fichier définit des variables appliquées à **tous les hôtes** (groupe `all`).

> Règle générale : `group_vars` a priorité sur `defaults`.

---

## 6) Le playbook d’entrée (`playbooks/patch_security.yml`)

Contenu :
- `hosts: all` : applique à tous les hôtes de l’inventaire choisi
- `gather_facts: true` : récupère les informations système (OS, version…)
- `roles: patch_security` : exécute le rôle

C’est le fichier à lancer le plus souvent.

---

## 7) Le rôle `patch_security` : fonctionnement détaillé

### 7.1 Point d’entrée : `roles/patch_security/tasks/main.yml`
Il se passe typiquement :

1) **Assert OS supporté**
- Debian
- RedHat
- Suse

2) **Initialise un rapport** dans une variable `patch_security_report`
- infos d’OS
- mode
- paquets ciblés
- indicateur `changed`

3) **Inclut des tâches spécifiques OS** :
- `debian.yml` si `ansible_facts.os_family == 'Debian'`
- `redhat.yml` si `ansible_facts.os_family == 'RedHat'`
- `suse.yml` si `ansible_facts.os_family == 'Suse'`

4) **Décide si un reboot est requis**
- en `auto`, on détecte un besoin reboot (Debian: `/var/run/reboot-required`, RHEL/SUSE: `needs-restarting -r`)

5) **Reboot**
- si requis, on déclenche le handler `Reboot system`

6) **Rapport**
- le rôle écrit un JSON dans `patch_security_report_dir/patch_security_report_filename`
- pour rester idempotent, le rapport n’est écrit **que si** patch appliqué (`report.changed`) ou reboot requis.

---

## 8) Détail par OS

### 8.1 Debian/Ubuntu : `roles/patch_security/tasks/debian.yml`
Logique :

1) `apt update_cache`
2) Si `patch_security_packages` n’est pas vide :
   - met à jour uniquement ces paquets (`only_upgrade: true`)
3) Sinon si `patch_security_mode == 'security'` :
   - tentative via `unattended-upgrades` s’il est installé
   - sinon fallback possible vers `apt upgrade dist` si `patch_security_allow_fallback_to_all: true`
4) Sinon si `patch_security_mode == 'all'` :
   - `apt upgrade dist`

Ensuite on met à jour `patch_security_report.changed` et on gardes des détails dans `patch_security_report.details`.

### 8.2 RHEL-like : `roles/patch_security/tasks/redhat.yml`
- installe `yum-utils` (utile pour `needs-restarting`)
- mise à jour packages ciblés ou :
  - `dnf security: true` pour la sécurité (si mode `security`)
  - sinon `dnf update_only` pour tout (`all`)

### 8.3 SUSE : `roles/patch_security/tasks/suse.yml`
- utilise `community.general.zypper`
- mode `security` applique des patches (`type: patch`)
- mode `all` met à jour tout

---

## 9) Reboot : `auto|always|never`

Variable : `patch_security_reboot`.

- `never` : ne reboot jamais
- `always` : reboot systématiquement
- `auto` : reboot seulement si détection « reboot requis »

Le reboot est fait via un handler : `roles/patch_security/handlers/main.yml`.

---

## 10) Rapport JSON (reporting)

Chemin :
- dossier : `patch_security_report_dir` (par défaut `/var/log/ansible`)
- fichier : `patch_security_report_filename` (par défaut `patch_security_report.json`)

Contenu :
- informations hôte/OS
- mode
- paquets ciblés
- `changed`
- `reboot_requested`
- `details`: sorties des modules (apt/dnf/zypper)

Template : `roles/patch_security/templates/report.json.j2`.

Note : le rapport est écrit seulement si utile (sinon Molecule idempotence échoue).

---

## 11) Exécutions recommandées (dev/prod)

### 11.1 Dev local
```bash
ansible-playbook playbooks/patch_security.yml --diff
```

### 11.2 Prod via SSH password + sudo password (simple, interactif)
```bash
ansible-playbook -i inventories/prod/hosts.ini playbooks/patch_security.yml \
  --ask-pass \
  --ask-become-pass \
  --diff
```

### 11.3 Prod en audit (sans appliquer)
```bash
ansible-playbook -i inventories/prod/hosts.ini playbooks/patch_security.yml \
  --check --diff \
  --ask-pass --ask-become-pass
```

### 11.4 Vérifier la connectivité avant le playbook
```bash
ansible -i inventories/prod/hosts.ini host1 -m ping --ask-pass --become --ask-become-pass
```

---

## 12) Tests automatisés (Molecule)

### 12.1 Pourquoi
- vérifier que ça marche dans un OS propre
- vérifier l’idempotence
- vérifier le rapport

### 12.2 Exécution
```bash
molecule test
```

### 12.3 Fichiers Molecule
- `molecule/default/molecule.yml` : driver docker + image + env
- `molecule/default/converge.yml` : applique le rôle
- `molecule/default/verify.yml` : assertions (rapport présent)

---

## 13) Bonnes pratiques de sécurité (important)

### 13.1 Ne pas stocker les mots de passe en clair dans `hosts.ini`
Évite :
- `ansible_ssh_pass=...`
- `ansible_become_password=...`

À la place :
- clés SSH
- `--ask-pass` / `--ask-become-pass` (interactif)
- Ansible Vault (chiffrement)

### 13.2 Ansible Vault (idée)
Tu peux créer un fichier chiffré (ex. `group_vars/all/vault.yml`) et définir :
- `ansible_become_password: "..."`

Puis exécuter avec :
```bash
ansible-playbook ... --ask-vault-pass
```

---

## 14) Dépannage rapide

### Erreur “sudo: il est nécessaire de saisir un mot de passe”
➡️ Ajoute `--ask-become-pass`.

### Erreur facts / setup qui échoue avec du texte dans stdout
Souvent causé par :
- sudo qui demande un mot de passe
- bannière SSH interactive
- script shell qui écrit dans stdout

➡️ Vérifier d’abord avec `ansible -m ping` + `--ask-become-pass`.

---

## 15) Prochaines améliorations possibles

- Variables différents par environnement :
  - `inventories/prod/group_vars/all.yml`
  - ou `group_vars/prod/*.yml`
- Reporting centralisé (envoyer le JSON vers un contrôleur)
- Fenêtre de maintenance (bloquer si hors fenêtre)
- Mode « appliquer uniquement des CVE spécifiques » (nécessite mapping CVE↔packages)

