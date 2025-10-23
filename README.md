# Remplacer-une-Livebox-5-par-un-MikroTik-hAP-ax-Orange-Fibre-
Ce guide explique comment remplacer une Livebox 5 par un routeur MikroTik hAP ax² en DHCP sur VLAN 832 (méthode actuelle Orange), après récupération d’un ONT externe. Il inclut : sniff des trames de la Livebox (pour récupérer les options DHCP), configuration WinBox (GUI) et RouterOS , ainsi que quelques notes sur l’ancien accès PPPoE sur VLAN 835

## Pour votre information la mise en forme du markdown et la correction des erreurs a été faites par ChatGPT , bon courage .

# 🧠 Remplacer une Livebox 5 par un **MikroTik hAP ax²** (Orange Fibre)

> **Objectif :** remplacer une Livebox 5 par un routeur MikroTik hAP ax², en **DHCP sur VLAN 832** (méthode actuelle Orange) après récupération d’un **ONT externe**.  
> Ce guide couvre : **sniff des trames DHCP**, configuration **WinBox (GUI)** et **RouterOS (CLI)**, plus des notes sur l’ancien **PPPoE VLAN 835**.

---

## ⚠️ Avertissements

- Ce tutoriel est fourni **à titre informatif**.  
  Vous êtes **seul responsable** de la configuration de votre réseau.
- **Ne publiez jamais** :
  - votre identifiant `fti/...`
  - votre mot de passe Orange
  - votre Option DHCP 90 complète (authentification)
- Orange migre **du PPPoE (VLAN 835)** vers **DHCP (VLAN 832)**.  
  Sur les lignes récentes, **seul DHCP/832** fonctionne.
- **Priorité VLAN (CoS 6)** obligatoire sur les requêtes DHCP.

---

## 🧩 Matériel requis

| Équipement | Description |
|-------------|--------------|
| **Routeur** | MikroTik hAP ax² (RouterOS v7.x) |
| **ONT externe** | Modem fibre (fourni par Orange ou compatible) |
| **PC** | Windows 10/11 avec 2 ports Ethernet (ou adaptateur USB-Ethernet) |
| **Logiciel** | [Wireshark](https://www.wireshark.org/) |

---

## 🔌 Schéma de connexion

```
Fibre (PTO)
    │
    ▼
ONT (Ethernet)
    │
    ▼
ether1 (WAN) ── MikroTik hAP ax² ── Bridge LAN → Switch / Wi-Fi / PC
```

---

## Étape 1 — Obtenir et brancher l’ONT externe

1. Contacter le **support Orange** pour demander un **ONT externe** (ou activer le vôtre) et demander vos identifiant et motdepasse (FTI).


## Étape 2 — Sniffer la Livebox pour récupérer les options DHCP

### 🎯 Objectif
Capter les trames DHCPv4/v6 envoyées par la Livebox pour extraire les **options Orange** :
`60, 61, 77, 90` + **CoS 6**

### 🧱 2.1. Pont réseau Windows pour sniff

1. Connecter :
   - ONT ↔ PC Avec un cable Ethernet 
   - PC ↔ Livebox WAN Connecter au meme pc a  
2. Dans **Panneau de configuration → Réseau → Connexions réseau** :
   - Sélectionner les deux interfaces
   - **Clic-droit → Connexions par pont (Bridge)**
   - <img width="1133" height="630" alt="Screenshot 2025-10-23 205913" src="https://github.com/user-attachments/assets/1297972d-da72-4f6a-8c31-fc371bb12e98" />
   - ![capture](https://github.com/user-attachments/assets/2133c2b8-35fc-4f10-8478-5fec066c2085)


3. Dans les propriétés du pont :
   - Cocher `Microsoft Network Monitor Driver` (ou `Npcap/WinPcap`)
   - Cocher bien les 2 interfaces de l'ont et la livebox
4. Ouvrir **Wireshark** → interface du pont réseau
5. Filtre conseillé :
   ```
   udp.port==67 || udp.port==68 || udp.port==546 || udp.port==547 || dhcp || dhcpv6
   ```
   ![thumbnail_image-1761239614806](https://github.com/user-attachments/assets/ddae5d12-bc7f-4a96-851c-fa23d2c113d7)


6. **Redémarrer la Livebox**, capturer quelques secondes.

---

### 🧾 2.2. Extraire les données

#### DHCPv4 (DISCOVER/REQUEST)
| Élément | Exemple |
|----------|----------|
| VLAN | ID = 832, Priority = 6 |
| Option 60 | `sagem` |
| Option 61 | `01:<MAC Livebox>` |
| Option 77 | `+FSVDSL_livebox.Internet.softathome.Livebox5` |
| Option 90 | Blob contenant `fti/...` (authentification) |


Pour l’option 90, il faut aller dans File → Export Packet Dissections → As JSON dans Wireshark afin d’obtenir la valeur complète.
Les valeurs apparaissent sous la forme 00:AA:BB:CC:ZZ:SS:DD:GG:HH:GG.
Supprime tous les : pour ne garder que la chaîne brute, puis ajoute 0x au début afin d’obtenir la valeur hexadécimale finale.
La meme chsoe pour l'adresse MAC faut recuperer celle de la livebox .

#### DHCPv6 (SOLICIT/REQUEST)
- DUID client (basé sur MAC)
- Option 11 (Auth)
- Option 15 (User Class)
- Option 16 (Vendor Class)
- Option 17 (Vendor Specific)

💡 Sauvegarder la capture en `.pcapng` pour analyse ultérieure.

---

## Étape 3 — Préparer le MikroTik

1. **Interface principale :** WinBox  
2. (Optionnel) **Reset configuration** :  
   `System → Reset Configuration`
3. (Optionnel) **Cloner la MAC Livebox** :  
   `Bridge → FTTH → General → Admin MAC Address = <MAC Livebox>`  
   (désactiver Auto-MAC)

---

## Étape 4 — Configuration dans WinBox (GUI)

### 4.1. Bridge WAN & VLAN 832
- **Bridge → +**
  ```
  Name = FTTH
  Protocol Mode = none
  Fast Forward = ON
  ```
  La photo bridge1.png
  
- **Interfaces → VLAN → +**
  ```
  Name = vlan832
  VLAN ID = 832
  Interface = ether1
  ```
  La photo vlan832_creation.png
  
- **Bridge → Ports → +**
  ```
  Interface = vlan832
  Bridge = FTTH
  ```
 <img width="1545" height="683" alt="bridge port" src="https://github.com/user-attachments/assets/324dcb80-e92a-465c-82ed-84fec24c5fa2" />

---

### 4.2. Forcer CoS = 6 sur DHCP

Bridge → Filters → +  
```
Chain = output
Out Interface = vlan832
MAC Protocol = ip
IP Protocol = udp
Src Port = 68
Dst Port = 67
Action = set priority
New Priority = 6
```
La photo bridgefilter**.png chaque filtre a 2 photo pour les 2 onglets de parametres

IPv6 :
```
MAC Protocol = ipv6
IPv6 Protocol = udp
Dst Port = 546
Action = set priority
New Priority = 6
```

---

### 4.3. Créer les DHCP Client Options

IP → DHCP Client → Options → +

| Option | Code | Exemple (hex) |
|---------|------|----------------|
| Vendor | 60 | `0x736167656d` |
| ClientID | 61 | `0x01<MAC Livebox>` |
| User Class | 77 | `0x2b46535644534c5f6c697665626f782e496e7465726e65742e736f66746174686f6d652e4c697665626f7835` |
| Auth (Option 90) | 90 | `0x00000000000000000000001a0900000558<FTI_HEX>` |

<img width="1580" height="385" alt="dhcpclientOptions" src="https://github.com/user-attachments/assets/32f44792-b8e4-4a0f-8cb8-ffb4a7322958" />


- Pour la livebox 5 Userclass et Vendor reste les memes pour client  id c'est 0x:mac_adresse_sans_:_
l'auth c'est la tram 90 sur wireshark sans les : et coller avec 0x au debut.
---

### 4.4. DHCP Client

```
Interface = FTTH
DHCP Options = vendor, clientid, userclass, authsend
Use Peer DNS = yes
Add Default Route = yes
```
<img width="831" height="851" alt="dhclient" src="https://github.com/user-attachments/assets/5a549105-eb5d-496c-bbec-aa78aa7743ee" />
<img width="858" height="850" alt="dhclientadvanced" src="https://github.com/user-attachments/assets/b44a0212-2d1b-434e-80cf-a66e5681210a" />

---

### 4.5. IPv6 DHCP Client (optionnel)
```
Interface = FTTH
Add Default Route = yes
Request = prefix
Advanced: Use Interface DUID = yes
```
j'ai que ipv4 pour mon cas.
---

### 4.6. NAT & DNS
```
IP → Firewall → NAT → +
Chain = srcnat
Out Interface = FTTH
Action = masquerade
```
```
IP → DNS : Allow Remote Requests = yes
```
<img width="1035" height="1055" alt="ip_firewall_nat" src="https://github.com/user-attachments/assets/251850d7-32dc-40fa-a964-9908fd5f016b" />
<img width="1025" height="700" alt="ip_firewall_nat_masquerade" src="https://github.com/user-attachments/assets/c011881f-854d-4a83-b5d3-7e862bf24dfc" />

---

## Étape 5 — Tests

- Vérifier **IP → DHCP Client : Status = bound**
- `Tools → Ping` → gateway (ex. 172.16.0.1)
- Depuis un poste LAN :
  ```
  ping 8.8.8.8
  ping google.fr
  ```
- `Torch` sur vlan832 → vérifier DNS/HTTPS sortants

---

## Ancienne méthode (historique) — PPPoE sur VLAN 835

Orange utilisait PPPoE sur VLAN 835 avec identifiants `fti/...`.  
Ça fonctionnait souvent après clonage MAC et création d’une interface **PPPoE Client** sur `vlan835`.  
Mais la majorité des lignes n’acceptent plus PPPoE : **DHCP/832** devient la norme.
<img width="1800" height="1261" alt="image" src="https://github.com/user-attachments/assets/e2bd18ac-5169-44c6-8b8e-6601fee8ac48" />
<img width="707" height="745" alt="image" src="https://github.com/user-attachments/assets/1bd984cd-036d-4532-bef1-b9541b47f7b8" />



---

## 🧰 Dépannage

| Problème | Vérifications |
|-----------|----------------|
| DHCP "searching" | Interface FTTH, CoS=6, Option 90 valide, câble ONT ↔ ether1 |
| IPv6 ne monte pas | Cocher "Use Interface DUID", reboot DHCPv6 client |
| Pas de surf | NAT actif, DNS correct (`ip dns set servers=1.1.1.1,8.8.8.8`) |

---

## 🔒 Sécurité

- Ne publiez **aucune donnée sensible** (Option 90, fti, mot de passe, DUID complet).
- Masquez ces données dans Wireshark avant publication GitHub.

---

## 🧱 Config RouterOS (CLI)

```
/interface bridge add name=FTTH protocol-mode=none
/interface vlan add name=vlan832 vlan-id=832 interface=ether1
/interface bridge port add bridge=FTTH interface=vlan832

/interface bridge filter
add chain=output out-interface=vlan832 mac-protocol=ip ip-protocol=udp src-port=68 dst-port=67 action=set-priority new-priority=6 comment="DHCPv4 CoS6"
add chain=output out-interface=vlan832 mac-protocol=ipv6 ipv6-protocol=udp dst-port=546 action=set-priority new-priority=6 comment="DHCPv6 CoS6"

/ip dhcp-client option add code=60 name=vendor value=0x736167656d
/ip dhcp-client option add code=61 name=clientid value=0x0118df269e2550
/ip dhcp-client option add code=77 name=userclass value=0x2b46535644534c5f6c697665626f782e496e7465726e65742e736f66746174686f6d652e4c697665626f7835
/ip dhcp-client option add code=90 name=authsend value=0x00000000000000000000001a0900000558<FTI_HEX>

/ip dhcp-client add interface=FTTH dhcp-options=vendor,clientid,userclass,authsend add-default-route=yes use-peer-dns=yes disabled=no
/ip firewall nat add chain=srcnat out-interface=FTTH action=masquerade
/ipv6 dhcp-client add interface=FTTH add-default-route=yes request=prefix use-interface-duid=yes disabled=no
```

---

## 🖼️ Captures recommandées

- Création du bridge FTTH  
- VLAN832 sur ether1  
- Ports FTTH + vlan832  
- Bridge Filters (CoS6 v4 & v6)  
- DHCP Client Options (60/61/77/90)  
- DHCP Client bound  
- Torch vlan832  

---


MIT (ou licence de ton choix)
