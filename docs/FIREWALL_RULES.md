# 🛡️ Matrice de Flux GRC : Politique de Sécurité pfSense

Ce document détaille la politique **Zero Trust** appliquée aux Firewalls du Siège (HQ) et de l'Agence (BR). L'accès entre les zones et les sites est strictement filtré selon le **Principe du Moindre Privilège** (Default Deny).

## 🏛️ SITE A : Siège Central (pfSense HQ - 10.10.10.254)

### 1. 🛑 Interface SECOPS_DMZ (10.50.10.x)

*Rôle : Zone d'hébergement des outils GRC/SecOps (Docker Stack) uniquement. **Isolation Critique**.*

| Action | Proto | Source | Destination | Port | Justification GRC | Preuve Visuelle |
| :---: | :---: | :--- | :--- | :--- | :--- | :--- |
| ✅ Pass | TCP | LAN subnets | SECOPS_DMZ subnets | **Ports\_Management** | **INGRESS Admin :** Autorise les administrateurs du LAN à gérer les services (NetBox:8000, Grafana:3000, Oxidized:8888). | `host_hq/pfsense_aliases_ports.png` |
| **❌ Block**| **\*** | **SECOPS\_DMZ subnets** | **LAN subnets** | **\*** | **ISOLATION CRITIQUE :** Interdit tout mouvement latéral de la DMZ vers le réseau de confiance. (Pilier Zero Trust). | **`host_hq/pfsense_zero_trust_block.png`** |
| ✅ Pass | * | SECOPS\_DMZ subnets | **! RFC1918** | * | Accès Internet uniquement (pour les mises à jour et les repos Docker). | `host_hq/pfsense_nat_dmz.png` |
| ✅ Pass | ICMP | LAN subnets | SECOPS\_DMZ subnets | * | Diagnostic et monitoring (confirmé par le ping HQ->DMZ). | `host_hq/hq_ping_lan_to_dmz_ok.png` |

### 2. 🏠 Interface LAN (10.10.10.x)

*Rôle : Zone de gestion de confiance. Accès illimité en sortie vers toutes les zones.*

| Action | Proto | Source | Destination | Port | Justification GRC |
| :---: | :---: | :--- | :--- | :---: | :--- |
| ✅ Pass | * | LAN subnets | Any | * | Accès complet vers toutes les zones (DMZ, VPN, Internet) pour l'administration. |
| ✅ Pass | ICMP | * | * | * | Diagnostic de base. |

---

## 🏭 SITE B : Agence (pfSense BR - 10.20.10.254)

### 1. 🔗 Interface VPN (WireGuard)

*Rôle : Point d'entrée du management et de la supervision venant du Siège.*

| Action | Proto | Source | Destination | Port | Justification GRC | Preuve Visuelle |
| :---: | :---: | :--- | :--- | :---: | :--- | :--- |
| ✅ Pass | UDP | serveur\_librenms | LAN Agence | **161 (SNMP)** | **Supervision Sécurisée :** Permet au serveur LibreNMS (DMZ HQ) de poller les agents SNMPv3 de l'Agence. | `host_br/br_pfsense_rules_vpn_in.png` |
| ✅ Pass | TCP | Siege Net | LAN Agence | **Ports\_Management** | **Télémaintenance/GRC :** Autorise le Siège à administrer le pfSense BR (GUI/SSH) et l'accès métier. | `host_br/br_pfsense_rules_vpn_in.png` |
| ✅ Pass | ICMP | Siege Net | LAN Agence | * | Diagnostic inter-sites (ping) de la disponibilité (confirmé par le Traceroute). | |
| ❌ Block | * | * | * | * | Règle de blocage implicite pour les autres flux non spécifiés. | |

### 2. 🖥️ Interface LAN (10.20.10.x)

*Rôle : Accès des utilisateurs de l'Agence vers les services du Siège.*

| Action | Proto | Source | Destination | Port | Justification GRC | Preuve Visuelle |
| :---: | :---: | :--- | :--- | :---: | :--- | :--- |
| ✅ Pass | TCP | LAN subnets | DMZ | **Ports\_Management** | **Accès Métier :** Permet aux utilisateurs d'accéder à NetBox et Grafana sur le Siège (confirmé par **`br_app_access_ok.png`**). | `host_br/br_pfsense_rules_lan_out.png` |
| ✅ Pass | * | LAN subnets | **! RFC1918**| * | Accès Internet. | |
| ❌ Block | * | LAN subnets | Siege Net | * | **Restriction :** Empêche l'Agence d'accéder au LAN Admin du Siège. | |

---

## ⚙️ Réglages Systèmes Critiques (Hardening GRC)

Ces configurations prouvent les choix d'ingénierie avancés et le durcissement du système.

### 1. Optimisation Kernel et Intégrité

| Réglage | Valeur | Justification Technique (Preuve) |
| :--- | :--- | :--- |
| **Hardware Checksum Offload** | **DÉSACTIVÉ** | Correction des erreurs de Checksum introduites par les drivers VirtIO en environnement virtualisé. |
| **Preuve Visuelle :** | | `docs/images/host_hq/pfsense_kernel_optimisation.png` |

### 2. Routage WireGuard (Correction d'Ingénierie)

| Élément | Description | Justification GRC (Preuve) |
| :--- | :--- | :--- |
| **Routes Statiques Manuelles** | Ajout de la route `10.20.10.0/24` via la Gateway VPN (`10.10.20.2`) sur le pfSense HQ. | Assure la résilience et la persistance du routage entre les sites, corrigeant les problèmes de routage dynamique WireGuard/FreeBSD. |
| **Preuve Visuelle :** | | `docs/images/host_hq/pfsense_static_route_vpn.png` |

### 3. Supervision Sécurisée (SNMP)

| Élément | Réglage | Justification GRC (Preuve) |
| :--- | :--- | :--- |
| **SNMP Daemon** | **Bind Interfaces** limité à **LAN** et **VPN** | L'agent de supervision n'écoute **jamais** sur l'interface WAN, réduisant la surface d'attaque. |
| **Preuve Visuelle :** | | `docs/images/host_br/br_snmp_binding_secure.png` |
