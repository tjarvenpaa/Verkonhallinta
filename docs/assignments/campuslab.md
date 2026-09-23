# Labra: Cisco-verkon automaatio Ansiblella

## Tavoite

Laboratorion tavoitteena on opetella:

- Verkkolaitteiden hallintaa SSH-yhteydellä
- Ansible-inventaarion käyttöä
- Cisco IOS -laitteiden automaattista konfigurointia
- Konfiguraatioiden varmistamista (backup)
- Konfiguraatioiden versionhallintaa Gitillä
- Muutosten palauttamista aiempaan tilaan

```mermaid
flowchart TD
    server["Ubuntu Server<br/>Ansible Control Node<br/>Git Repository<br/>192.168.100.10"]
    switch1["Cisco SW1<br/>Management Switch<br/>192.168.100.21"]
    switch2["Cisco SW2<br/>Access Switch<br/>(k&auml;ytt&auml;j&auml;verkko)"]
    r1["R1<br/>Cisco Router<br/>192.168.100.11 / 172.16.0.1"]
    r2["R2<br/>Cisco Router<br/>172.16.0.2 / 10.10.10.1"]
    workstation["Workstation<br/>Windows/Linux<br/>10.10.10.100"]

    server --- switch1
    switch1 --- r1
    r1 --- r2
    r2 --- switch2
    switch2 --- workstation
```

---

# Fyysiset laitteet

## Hallintapalvelin

- Ubuntu Server 24.04
- Ansible
- Git
- VS Code (valinnainen)
- Python3

## Verkkolaitteet

- Cisco Router R1
- Cisco Router R2
- Cisco Switch SW1
- Cisco Switch SW2

## Työasema

- Windows 11 tai Linux
- SSH-client
- selain

---

# IP-suunnitelma

## Hallintaverkko

Verkon tarkoitus on mahdollistaa Ansible-hallinta.

| Laite | Osoite |
|---------|---------|
| Ansible Server | 192.168.100.10/24 |
| SW1 | 192.168.100.21/24 |
| R1 (G0/0) | 192.168.100.11/24 |
| Gateway | R1 (192.168.100.11) |

> **Huom:** SW2 ja R2 eivät ole fyysisesti kiinni tässä verkossa (ks. topologia), joten niille ei anneta osoitetta 192.168.100.0/24-verkosta. R2 hallitaan reititetysti oman pisteestä-pisteeseen-osoitteensa kautta (172.16.0.2), koska Cisco IOS reitittää oletuksena suoraan kytkettyjen verkkojen välillä. SW2 ei ole mukana Ansible-inventaariossa, joten sen ei tarvitse olla saavutettavissa hallintaverkosta.

---

## Reitittimien välinen yhteys

| Laite | Interface | Osoite |
|---------|---------|---------|
| R1 | G0/1 | 172.16.0.1/30 |
| R2 | G0/1 | 172.16.0.2/30 |

---

## Käyttäjäverkko

| Laite | Interface | Osoite |
|---------|---------|---------|
| R2 | G0/0 | 10.10.10.1/24 |
| Workstation | NIC | 10.10.10.100/24 |

> Työasema on kiinni SW2:n kautta R2:n G0/0-interfaceen, mikä vastaa topologiakaaviota (R2 --- SW2 --- Workstation).

---

# Topologian looginen rakenne

```mermaid
flowchart LR
    subgraph MGMT["Management Network 192.168.100.0/24"]
        ansible["Ansible Server<br/>192.168.100.10"]
        sw1["SW1"]
        ansible --- sw1
    end

    subgraph LINK["Router-to-Router 172.16.0.0/30"]
        direction LR
        r1if["R1 G0/1<br/>172.16.0.1"] --- r2if["R2 G0/1<br/>172.16.0.2"]
    end

    subgraph DATA["Data Network 10.10.10.0/24"]
        sw2["SW2"]
        ws["Workstation<br/>10.10.10.100"]
        sw2 --- ws
    end

    sw1 --- r1["R1<br/>G0/0: 192.168.100.11"]
    r1 --- r1if
    r2if --- r2["R2<br/>G0/0: 10.10.10.1"]
    r2 --- sw2
```

R1 on ainoa laite, joka on fyysisesti kiinni hallintaverkossa. R2 saavutetaan Ansiblesta reititetysti R1:n kautta osoitteessa 172.16.0.2, koska R1 reitittää oletusarvoisesti suoraan kytkettyjen verkkojensa (192.168.100.0/24 ja 172.16.0.0/30) välillä.

---

# Cisco-laitteiden peruskonfiguraatio

## SSH:n käyttöönotto

Suoritetaan molemmille reitittimille.

```cisco
hostname R1

ip domain-name lab.local

crypto key generate rsa modulus 2048

username admin privilege 15 secret Salainen123

line vty 0 4
 login local
 transport input ssh

ip ssh version 2
```

---

## Hallintaosoitteet

### R1

```cisco
! G0/0 kohti SW1:tä / hallintaverkkoa
interface g0/0
 ip address 192.168.100.11 255.255.255.0
 no shutdown

! G0/1 reititysverkkoon R2:lle
interface g0/1
 ip address 172.16.0.1 255.255.255.252
 no shutdown
```

### R2

```cisco
! G0/0 kohti SW2:ta / käyttäjäverkkoa
interface g0/0
 ip address 10.10.10.1 255.255.255.0
 no shutdown

! G0/1 reititysverkkoon R1:lle
interface g0/1
 ip address 172.16.0.2 255.255.255.252
 no shutdown
```

> Routerit eivät ole L3-kytkimiä, joten `interface vlan 1` ei ole niissä pätevä konfiguraatio - hallinta- ja käyttäjäverkot konfiguroidaan suoraan fyysisille G0/x-interfaceille. `ip routing` on Cisco IOS -reitittimillä päällä oletusarvoisesti, joten R1 reitittää automaattisesti hallintaverkon (192.168.100.0/24) ja reititysverkon (172.16.0.0/30) välillä - erillisiä staattisia reittejä ei tarvita R2:n saavuttamiseksi Ansiblesta. Staattiset verkot kannattaa kuitenkin lisätä reitittimille, jotta palvelimet ja työasemat pystyvät liikennöimään keskenään.

```cisco
!reitti R1
ip route 10.10.10.0 255.255.255.0 172.16.0.2

!reitti R2
ip route 192.168.100.0 255.255.255.0 172.16.0.1
```

---

# Ansible-palvelimen valmistelu

Tehdään labran käyttöön uusi / uudet wsl distrot. Ennen aloittamista, vaihda wsl verkon tila muotoon mirrored. Tämä onnistuu yksinkertaisimmin windows sovelluksella wsl settings (löytyy start-valikosta)

## WSL 

```powershell
wsl -l -o
# valitaan sopiva distro, esim. ubuntu-24.04
wsl --install Ubuntu-24.04 --name lab1-ubuntu

# voit kirjautua uudelle distrolle jollei automaattisesti siirry

wsl -d lab1-ubuntu

```
Ota luokan koneelle käyttöön uusi distro käyttäjätunnuksella tllabra ja salasanalla tllabra


## Tarvittavat paketit

```bash
sudo apt update

sudo apt install -y git python3-pip python3-paramiko

sudo apt install ansible

# vain jos asennus pip kautta.
ansible-galaxy collection install cisco.ios
```

---

# Projektirakenne

```text
network-automation/

├── inventory
│   └── hosts.yml
│
├── playbooks
│   ├── backup.yml
│   ├── hostname.yml
│   ├── facts.yml
│   └── restore.yml
│
├── backups
│
└── README.md
```

---

# Inventaario

Tiedosto:

```yaml
# inventory/hosts.yml

all:
  children:
    routers:
      hosts:
        r1:
          ansible_host: 192.168.100.11
        r2:
          # R2 ei ole kiinni hallintaverkossa, mutta on reititetysti
          # saavutettavissa oman G0/1-osoitteensa kautta R1:n läpi
          ansible_host: 172.16.0.2

      vars:
        ansible_connection: network_cli
        ansible_network_os: cisco.ios.ios
        ansible_user: admin
        ansible_password: Salainen123
        ansible_ssh_common_args:
            -o HostKeyAlgorithms=+ssh-rsa
            -o StrictHostKeyChecking=no
            -o UserKnownHostsFile=/dev/null
            -o PubkeyAcceptedAlgorithms=+ssh-rsa
            -o KexAlgorithms=+diffie-hellman-group14-sha1
```

---

# Harjoitus 1: Yhteystestin suorittaminen

Komento:

```bash
ansible routers -i inventory/hosts.yml -m ping
```

Tavoite:

- Ansible saa SSH-yhteyden laitteisiin
- Molemmat laitteet vastaavat

---

# Harjoitus 2: Laitetietojen kerääminen

Playbook:

```bash
ansible-playbook \
 playbooks/facts.yml \
 -i inventory/hosts.yml
```

Kerättävät tiedot:

- hostname
- IOS-versio
- sarjanumero
- interfacet

Tavoite:

Ymmärtää inventointiprosessi.

---

# Harjoitus 3: Konfiguraation varmistus

Suorita:

```bash
ansible-playbook \
 playbooks/backup.yml \
 -i inventory/hosts.yml
```

Tuloksena:

```text
backups/

├── r1.cfg
└── r2.cfg
```

---

# Harjoitus 4: Konfiguraation tallennus Git-repositorioon

Luo repository.

```bash
git init
```

Lisää tiedostot.

```bash
git add .
git commit -m "Initial configuration"
```

Tavoite:

Verkon ensimmäinen dokumentoitu tila.

---

# Harjoitus 5: Hostname-muutos Ansiblella

Luo playbook.

Muuta:

```text
R1 -> Branch-R1
R2 -> Branch-R2
```

Aja playbook.

```bash
ansible-playbook hostname.yml
```

Varmista SSH:lla että muutokset toteutuivat.

---

# Harjoitus 6: Muutoksen dokumentointi

Ota uudet varmistukset.

```bash
ansible-playbook backup.yml
```

Tee Git commit.

```bash
git add .
git commit -m "Hostname update"
```

Tarkastele muutoksia.

```bash
git diff
```

---

# Harjoitus 7: Virhetilanteen palautus

Muuta reitittimen hostname käsin CLI:stä.

```cisco
hostname VIRHE
```

Suorita palautus.

```bash
ansible-playbook restore.yml
```

Tarkista että laite palautuu aiemmin tallennettuun tilaan.

---

# Oppimistavoitteet

Harjoituksen jälkeen opiskelija osaa:

- muodostaa SSH-yhteyden Cisco-laitteisiin
- rakentaa Ansible-inventaarion
- käyttää Cisco IOS -moduuleja
- kerätä tietoja verkkolaitteista
- varmistaa konfiguraatioita
- käyttää Git-versionhallintaa verkkolaitteiden konfiguraatioiden hallintaan
- palauttaa verkkolaitteen tunnettuun konfiguraatiotilaan
- ymmärtää Infrastructure as Code -ajattelun perusteet

---

