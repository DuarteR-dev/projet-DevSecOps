# Projet Sécurité des OS
Ce projet automatise le déploiement, la configuration et la sécurisation d'une architecture réseau à l'aide d'Ansible. L'infrastructure s'appuie sur des machines virtuelles hébergeant des conteneurs Incus, interconnectés via un tunnel VXLAN et protégés par un ensemble de règles de pare-feu strictes et des mesures de durcissement système.

## Architecture du Projet

L'infrastructure déployée comprend les éléments suivants :
* Hyperviseur (`oscar`) : Hôte physique ou virtuel principal préparant les machines virtuelles.
* VM Routeur (`vm-red`) : Fait office de passerelle, gère le NAT et le routage entre l'underlay et l'accès Internet.
* VMs Tenants (`vm-blue` et `vm-green`) : Hébergent les conteneurs Incus.
* Réseau Overlay (VXLAN) : Tunnel chiffré permettant aux conteneurs de la VM Bleue et de la VM Verte de communiquer de manière transparente.
* Haute Disponibilité (Keepalived) : Fournit une Virtual IP pour la passerelle des conteneurs, assurant la tolérance aux pannes.
* Conteneurs (Incus) : Conteneurs Alpine Linux faisant office de serveurs DHCP (`dnsmasq`), serveurs Web (`nginx`) et de clients.

## Sécurité

Le projet intègre la sécurité dès le déploiement ("Security by default") :
* Durcissement du noyau Linux : Désactivation des modules vulnérables, protection de la mémoire, paramètres sysctl stricts.
* Sécurisation SSH : Désactivation de root, authentification par clé uniquement.
* Pare-feu UFW : Filtrage strict des flux entrants, sortants et du routage asymétrique.
* Politique de mots de passe : Chiffrement robuste, règles d'expiration.
* Audit et vérification de la conformité via Lynis.

## Prérequis et Préparation

### IMPORTANT : Configuration des secrets (`.iac_passwd.yaml`)

Pour des raisons évidentes de sécurité, les mots de passe et les secrets ne sont pas versionnés sur Git. Vous devez créer manuellement un fichier nommé `.iac_passwd.yaml` à la racine du projet avant toute exécution.

1. Créez le fichier à la racine du dépôt :
```bash
touch .iac_passwd.yaml
```

2. Ajoutez-y vos variables sensibles :
```yaml
hypervisor_user: "xxxxxxxx"
hypervisor_pass: "xxxxxxx"
keepalived_auth_pass: "Secr3t"
ansible_become_pass: "-etu-"
```

## Exécution et Déploiement

Le projet se déploie à l'aide de playbooks spécifiques.

### 1. Déploiement complet de l'infrastructure

Ce playbook s'occupe de provisionner les VMs, appliquer les règles de sécurité (common_sec), configurer le routage, monter le tunnel VXLAN et déployer les conteneurs Incus.

```bash
ansible-playbook deploy.yaml
```
Il est à noter que ce playbook exécute les playbooks suivants.

### 2. Validation du réseau

Une fois le déploiement terminé, vous pouvez vérifier que le tunnel VXLAN fonctionne, que le routage est opérationnel et que les conteneurs ont bien accès à Internet via la VIP Keepalived :

```bash
ansible-playbook check_network.yaml
```

### 3. Audit de sécurité

Pour lancer un audit de conformité sur les VMs déployées et générer les rapports Lynis :

```bash
ansible-playbook check_security.yaml
```

On peut afficher le score d'audit :
```bash
cat reports/lynis-vm-blue.dat | grep hardening_index
```

## Structure du dépôt

```text
.
├── .github
│   └── workflows
│       └── ci-security.yaml
├── group_vars
│   └── all.yaml
├── host_vars
│   ├── oscar.yaml
│   ├── vm-blue.yaml
│   ├── vm-green.yaml
│   └── vm-red.yaml
├── inventory
│   └── hosts.yaml
├── reports
│   ├── lynis-vm-blue.dat
│   ├── lynis-vm-green.dat
│   └── lynis-vm-red.dat
├── roles
│   ├── common_sec
│   │   ├── handlers
│   │   │   └── main.yaml
│   │   └── tasks
│   │       └── main.yaml
│   ├── incus
│   │   └── tasks
│   │       └── main.yaml
│   ├── network_router
│   │   ├── handlers
│   │   │   └── main.yaml
│   │   ├── tasks
│   │   │   └── main.yaml
│   │   └── templates
│   │       └── 01-netcfg.yaml.j2
│   ├── network_tenant
│   │   ├── defaults
│   │   │   └── main.yaml
│   │   ├── handlers
│   │   │   └── main.yaml
│   │   ├── tasks
│   │   │   └── main.yaml
│   │   └── templates
│   │       ├── 01-netcfg.yaml.j2
│   │       ├── keepalived.conf.j2
│   │       └── radvd.j2
│   └── provision_vms
│       ├── tasks
│       │   └── main.yaml
│       └── templates
│           ├── baking-netplan.j2
│           ├── lab.yaml.j2
│           └── switch.yaml.j2
├── ansible.cfg
├── .ansible-lint
├── check_network.yaml
├── check_security.yaml
├── deploy.yaml
├── .gitignore
├── .iac_passwd.yaml
├── requirements.txt
├── requirements.yaml
├── site.yaml
└── .vault_pass.txt
```
