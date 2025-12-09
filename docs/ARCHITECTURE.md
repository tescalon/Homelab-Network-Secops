# 📐 Documentation de l'Architecture Technique

Ce document détaille les choix de conception, la topologie réseau, le plan d'adressage (IPAM) et les configurations de sécurité appliquées au projet Home Lab.

---

## 1. Vue d'Ensemble Logique

L'architecture repose sur une topologie **Hub-and-Spoke** sécurisée via un tunnel VPN WireGuard.
* **Le Siège (HQ)** : Héberge l'infrastructure physique (Proxmox dans le LAN) et logique (Services dans la DMZ).
* **L'Agence (BR)** : Site distant connecté via VPN pour la supervision déportée.

```mermaid
flowchart TD
    %% Styles
    classDef firewall fill:#e74c3c,stroke:#333,stroke-width:2px,color:white;
    classDef wan fill:#34495e,stroke:#333,stroke-width:2px,color:white;
    classDef lan fill:#27ae60,stroke:#333,stroke-width:2px,color:white;
    classDef dmz fill:#f39c12,stroke:#333,stroke-width:2px,color:white;
    
    subgraph Internet
        ISP[FAI / Internet]:::wan
    end

    %% SITE PRINCIPAL (SIÈGE)
    subgraph HQ [🏢 SITE SIÈGE - Infra Proxmox]
        
        %% Firewall HQ
        pfHQ[("🔥 pfSense HQ
        GW: 10.10.10.254")]:::firewall
        
        %% Interfaces Physiques/Virtuelles distinctes (Pas de Trunk)
        ISP -- "WAN (em0)" --> pfHQ
        
        %% Zone DMZ (Réseau Isolé)
        subgraph Zone_DMZ [Zone DMZ - 10.50.10.0/24]
            pfHQ -- "em2 (DMZ)
            10.50.10.254" --> SwitchDMZ[vSwitch/Bridge DMZ]
            SwitchDMZ -- "10.50.10.10" --> DockerHost[("🐳 Srv-Admin
            (Docker sur LXC)
            Netbox, LibreNMS, Ansible, Grafana, Oxidized")]:::dmz
        end

        %% Zone LAN (Réseau de Management)
        subgraph Zone_LAN [Zone LAN - 10.10.10.0/24]
            pfHQ -- "em1 (LAN)
            10.10.10.254" --> SwitchLAN[vSwitch/Bridge LAN]
            SwitchLAN -- "10.10.10.15" --> PVE[Hyperviseur Proxmox]:::lan
        end
    end

    %% SITE AGENCE
    subgraph BR [🏠 SITE AGENCE]
        pfBR[("🔥 pfSense Agence
        GW: 10.20.10.254
        + ntopng (Edge)")]:::firewall
        
        ISP -- "WAN (em0)" --> pfBR

        subgraph Zone_LAN_BR [LAN Agence - 10.20.10.0/24]
            pfBR -- "LAN
            10.20.10.254" --> ClientBR[Poste Client
            Debian: 10.20.10.130]:::lan
        end
    end

    %% Relations Logiques
    %% Tunnel VPN
    pfHQ <-->|🔒 WireGuard VPN em3 - Tunnel: 10.10.20.0/24| pfBR
    
    %% Lien d'Hébergement (Physique/Virtuel)
    PVE -.->|Héberge le Conteneur LXC| DockerHost
