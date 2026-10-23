# InfraReady

InfraReady est un laboratoire d'administration de systèmes Linux et Windows réalisé avec VMware Workstation Pro. Le projet couvre la mise en place d'une infrastructure, l'automatisation de tâches d'administration, la supervision et le diagnostic d'un incident.

- **Équipe :** Ilies BOSSUYT et Léo COLLOWLAD
- **Formation :** Bachelor 3

## Architecture

Les trois machines virtuelles communiquent sur le réseau privé VMware NAT **VMnet8 (`192.168.50.0/24`)**, dont la passerelle est `192.168.50.2`. Le domaine Active Directory est **`infraready.test`**.

| Machine | Système | Adresse IP | Rôle principal |
| --- | --- | --- | --- |
| SRV-AD-WIN2025 | Windows Server 2025 | `192.168.50.10` | Active Directory, DNS, partages et permissions NTFS, Windows Exporter |
| SRV-SERVICE1-UBUNTU26 | Ubuntu Server 26.04 | `192.168.50.20` | Nginx, comptes et stockage Linux, Node Exporter |
| SRV-OPS1-UBUNTU26 | Ubuntu Server 26.04 | `192.168.50.30` | Ansible, Prometheus, Grafana et poste d'administration |

Depuis OPS1, Ansible administre SERVICE1 par SSH et le serveur Windows par WinRM/Kerberos. Prometheus collecte les métriques des exporters Linux (`9100`) et Windows (`9182`), puis Grafana les présente. Le serveur Windows assure la résolution DNS du laboratoire.

## Technologies utilisées

- VMware Workstation Pro et réseau NAT VMnet8 ;
- Windows Server 2025, Active Directory et DNS ;
- Ubuntu Server 26.04 et Nginx ;
- Ansible avec la collection `ansible.windows` ;
- Prometheus, Node Exporter, Windows Exporter et Grafana ;
- Git, GitHub et GitHub Actions.

## Fonctionnalités

- Trois VM configurées avec des adresses IP fixes et une résolution DNS sur le réseau NAT.
- Domaine Active Directory avec six utilisateurs, trois groupes de sécurité et des permissions NTFS.
- Administration Linux des comptes, groupes, permissions, services, journaux et d'un disque ext4 de 10 Go monté de façon persistante sur `/srv/infraready`.
- Installation et activation de Nginx, ainsi que maintenance des paquets Ubuntu, avec Ansible.
- Mises à jour Windows réalisées dans le laboratoire. Le playbook versionné contrôle la connectivité, crée un répertoire et un fichier de test, puis vérifie le service DNS ; il n'installe pas les mises à jour.
- Supervision de la disponibilité, du CPU, de la RAM et du stockage ; alerte de perte de collecte et diagnostic d'un incident Node Exporter avec retour à la normale.
- Validation automatique des fichiers Ansible et du YAML Prometheus avec GitHub Actions.

## Organisation du dépôt

```text
.
├── .github/
│   └── workflows/
│       └── validation.yml
├── .gitignore
├── README.md
├── docs/
│   └── .gitkeep
├── inventory/
│   ├── group_vars/
│   │   ├── linux.yml
│   │   └── windows.yml
│   └── hosts.ini
├── monitoring/
│   ├── dashboards/
│   │   └── infraready-supervision.json
│   └── prometheus.yml
├── playbooks/
│   ├── maintenance.yml
│   ├── nginx.yml
│   └── windows.yml
└── requirements.yml
```

- `inventory/` définit les hôtes Linux et Windows ainsi que leurs variables de groupe.
- `playbooks/` contient les tâches Ansible pour Nginx, Windows et la maintenance Linux.
- `monitoring/` contient la configuration Prometheus et l'export du dashboard Grafana.
- `.github/workflows/` contient le workflow de validation continue.
- `requirements.yml` référence la collection Ansible nécessaire au playbook Windows.

## Utilisation des playbooks Ansible

Ces commandes se lancent depuis la racine du dépôt, sur un contrôleur ayant accès au réseau du laboratoire, notamment OPS1. Il faut disposer d'Ansible, de la collection indiquée dans `requirements.yml`, des droits SSH et d'élévation pour Linux, ainsi que de `pywinrm[kerberos]` et des outils Kerberos pour Windows.

L'inventaire utilise une clé SSH présente sur le contrôleur pour SERVICE1. Pour Windows, il utilise le nom DNS `srv-ad-win2025.infraready.test`, WinRM sur le port `5985` et un ticket Kerberos obtenu préalablement : le mode Kerberos configuré est manuel. Aucun mot de passe ne doit être ajouté au dépôt.

Installer la collection et vérifier l'inventaire :

```bash
ansible-galaxy collection install -r requirements.yml
ansible-inventory -i inventory/hosts.ini --graph
```

Exécuter les trois playbooks, selon l'opération souhaitée :

```bash
ansible-playbook -i inventory/hosts.ini playbooks/nginx.yml
ansible-playbook -i inventory/hosts.ini playbooks/windows.yml
ansible-playbook -i inventory/hosts.ini playbooks/maintenance.yml
```

Ces playbooks ciblent les tâches décrites ci-dessus ; ils ne déploient pas les trois VM ni l'ensemble de leurs services.

## Supervision

Les exporters exposent les métriques, **Prometheus** les collecte et **Grafana** les affiche. La configuration versionnée dans `monitoring/prometheus.yml` collecte notamment les cibles Linux et Windows du laboratoire. Le dashboard `monitoring/dashboards/infraready-supervision.json` contient quatre panneaux : disponibilité, utilisation CPU, RAM et stockage.

Une alerte Grafana surveillant `up < 1` signale une perte de collecte des métriques. Cela ne prouve pas, à lui seul, l'arrêt complet du serveur : l'exporter, le réseau ou Prometheus peuvent aussi être en cause. Cette règle d'alerte ne figure pas dans l'export JSON du dashboard.

## GitHub Actions

Le workflow `.github/workflows/validation.yml` s'exécute sur chaque `push` et `pull_request` ; il peut aussi être lancé manuellement. Il installe Ansible et la collection requise, affiche le graphe de l'inventaire, vérifie la syntaxe des trois playbooks et analyse le YAML Prometheus. Une erreur dans l'un de ces contrôles fait échouer le workflow.

Une exécution du workflow a déjà réussi. Les runners GitHub n'ont pas accès au réseau VMware privé : le workflow vérifie les fichiers, sans contacter les VM ni exécuter les playbooks sur elles.

## Sécurité et limites

- L'administration Linux repose sur SSH ; l'administration Windows utilise WinRM avec Kerberos et chiffrement des messages.
- Les VM sont des machines de laboratoire sur un réseau privé, avec des accès réseau restreints.
- Le dépôt ne contient ni mot de passe ni clé privée ; l'inventaire référence seulement le chemin local d'une clé SSH.
- Les serveurs Ubuntu ne sont pas intégrés au domaine Active Directory.
- Le laboratoire utilise Ubuntu Server 26.04, alors que le sujet initial mentionnait Ubuntu 24.04 LTS.

## Auteurs

- Ilies BOSSUYT
- Léo COLLOWLAD
- Bachelor 3
