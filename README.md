# Dokumentacja serwerowni V lo w Krakowie

W tym dokumencie znajduje się opis nowej infrastuktury oraz narzędzi stosowanych w sewrerownii V lo w Krakowie


* Dane kontaktowe 
	* Piotr Pyciński - 797685220 / s_vlo@lifetechguard.pl
	* Maciej Tomkiewicz - 663143804 

* 
---
### Przydatne linki:

Sieć:
*
Wirutalizatory:

---
Schemat sieci - GW

![[schemat_sieci_vlo.drawio.png]]

### Opis i adresacja sieci:

#### Adresacja i VLANy (szczegóły)

- **Trunk do CORE (bonding-sw-main, LACP 4×GE)**: przenosi wszystkie VLAN z listy w diagramie.
    
- **Gatewaye L3 na CCR**:
    
    - VLAN 10 – 10.10.0.1/16 (Zarządzanie)
        
    - VLAN 15 – 10.5.0.1/24 (WiFi mgmt)
        
    - VLAN 16 – 10.6.0.1/24 (VoiceIP)
        
    - VLAN 17 – 10.7.0.1/24 (Monitoring)
        
    - VLAN 20 – 10.20.0.1/24 (Usługi)
        
    - VLAN 70/72/77/78 – 10.4.70/72/77/78.253/24 (MChE)
        
    - VLAN 300 – 193.193.65.225/27 (publiczne) + **IPv6** 2001:6d8:1000::1/64
        
    - VLAN 302 – 192.168.2.1/24 (Administracja)
        
    - VLAN 308 – 192.168.8.1/21 (Pracownie)
        
    - VLAN 330 – 192.168.30.1/24 (INF3)
        
    - VLAN 416 – 172.16.0.1/22 (WiFi uczniowie)
        
    - VLAN 417 – 172.17.0.1/23 (WiFi nauczyciele)
        
    - VLAN 418 – 172.18.0.1/24 (WiFi goście)
        
    - VLAN 419 – 172.19.0.1/23 (WiFi staff)
        
    - VLAN 420 – 172.20.0.1/24 (Classroom)

**Publiczny prefiks IPv4:** 193.193.65.224/27 (na VLAN 300) – hosty/serwisy: `users (.227)`, `db (.226)`, `auth (.228)`, `www (.234)`, `voter (.237)`, `mssql (.238)`, `w2 (.239)`, `web (.248)`, `gielda (.249)`, itp.  
**Publiczny IPv6:** 2001:6d8:1000::/64 (na VLAN 300) + brama IPv6 na interfejsie do Cyfronetu.

_____
#### Routing i strefowanie ruchu

- **Domyślna trasa IPv4 (main):** 0.0.0.0/0 → 193.193.64.17 (Cyfronet IPv4).
    
- **Alternatywna tablica** `**ose**`**:** 0.0.0.0/0 → 192.168.10.1 (dla OSE).  
    **Rule:** cały ruch **172.16.0.0/22 (WiFi uczniowie)** lookup-only-in-table=**ose** → NAT masquerade na **ether3 (OSE)**.
    
- **IPv6 default:** ::/0 → 2001:6d8:0:1106::1.
    

---

#### DHCP

- **Relay** z wielu VLAN do serwera **10.10.0.7** (w Management): VLAN 15,16,17,302,308,416,417,418,419,420 oraz inne.
    
- CCR akceptuje **DHCP (serwer/relay)** na wszystkich interfejsach LAN (z wyjątkami dla WAN).
    

---

#### Usługi w VLAN 20 (Services) – zidentyfikowane po regułach FW

- **DC1 / DC2:** 10.20.0.231 / 10.20.0.232 – porty AD/DNS/Kerberos/LDAP, itp. wydzielone łańcuchem `dc_forward` (whitelist z Pracowni i wybranych hostów).
- **Wazuh/OSSEC:** 10.20.0.228 (TCP 1514/1515).
- **Dysk/udziały SMB:** 10.20.0.229 (TCP 445) – dostęp dla listy _staff_.
- **FOG Imaging:** 10.20.0.230 oraz 10.20.0.240 (NFS/TFTP/HTTP) – specjalne wyjątki dla _pracownie_ + multicast PIM-SM.
- **Kontroler Wi-Fi (UniFi?):** 10.10.0.127 – dostęp z pracowni i relacje z RADIUS.
- **RADIUS dla PPP:** 10.10.0.127 (porty 1818/1819) – autoryzacja OVPN.
    

---

#### VPN i dostęp administracyjny

- **WireGuard (admin):** port 13231/UDP, adresacja: 10.10.104.0/24 i 172.31.0.0/24;
    
    - Peers: `pyton (172.31.0.2/32)`, `maciej (172.31.0.3/32)`.
        
    - Lista `VPN-wireguard` dopuszczona do chain=input oraz do zarządzania.
        
- **OpenVPN (TCP/1194)** – certyfikaty, TLS 1.2, profil `openvpn_frosty`.
    
- **IKEv2 (IPsec):** profil strong crypto (x25519/aes-256), puli **192.168.100.100-199**; split-include 0.0.0.0/0.
    
- **Dostęp admin (SSH/Winbox/WWW-SSL)** ograniczony do listy **management (10.10.0.0/16, 10.5.0.0/24, 192.168.100.0/24)**.
    

---

#### Firewall (skrót polityki)

- **Domyślnie DROP** niepożądany ruch (INPUT/ FORWARD) po zaakceptowaniu RELATED,ESTABLISHED.
    
- **Listy adresowe i blacklisting:** automatyczne pobieranie _FireHOL Level1_, _SSH brute force_, _AbuseIPDB_ → osobne łańcuchy `blacklist_chain_input/forward` + _portscan_ (PSD) z dodawaniem IP do blacklisty.
    
- **Limity ICMP** na INPUT/FORWARD (anti-flood) + drop rzadkich typów (timestamp).
    
- **Wyjątki publikacyjne (VLAN 300):** precyzyjne allow na usługi (HTTP/HTTPS/SSH/DNS/QUIC/ICMP) dla poszczególnych hostów (`web`, `voter`, `gielda`, `users`, `auth`, `dns` itd.). Wejście na publiczny zakres bez matchu → DROP.
    
- **Segmentowe reguły wyjścia:**
    
    - _pracownie_ → Internet (NAT na puli 193.193.65.245–251) + specjalne dostępy do FOG i kontrolera Wi‑Fi.
        
    - _staff_ → Internet (NAT 193.193.65.252) + wewnętrzne wzajemne.
        
    - _teacher_ → Internet (NAT 193.193.65.251) + drukarki w Admin.
        
    - _voiceip/monitoring/management_ → Internet (NAT 193.193.65.253).
        
    - _reszta_ (catch‑all) → NAT 193.193.65.254.
        
    - _WiFi Uczniowie_ (172.16.0.0/22) → wyłącznie OSE (ether3), inne kierunki DROP.
        

---

#### NAT (mapowanie źródeł)

- **Pracownie:** `193.193.65.245–251`    
- **Management/VoiceIP/Monitoring:** `193.193.65.253`
- **Staff:** `193.193.65.252`
- **Teacher:** `193.193.65.251`
- **Pozostałe (catch‑all):** `193.193.65.254`
- **WiFi Uczniowie → OSE:** masquerade na `ether3-Internet_OSE` (osobna tablica routingu).
    

---

#### Multicast (PIM-SM, FOG/klonowanie)

- **Instance:** `fog-multicast` (VRF main).
- **Interfejsy PIM-SM:** VLAN 20 (Services) + VLAN 330 (INF3).
- **Static RP:** 192.168.30.1 (gateway VLAN 330/INF3).
- Ruch do/z **10.20.0.240** do listy _pracownie_ jest dozwolony (IGMP/PIM) – wsparcie dla multicastowego deployu obrazów.
    

---

#### IPv6

- **Adresy na routerze:**
    - WANv6 (bridge-Cyfronet_IPv6): 2001:6d8:0:1106::2
    - VLAN 300 (public): 2001:6d8:1000::1/64    
    - VLAN 330 (INF3): 2001:6d8:1000:33::1/64
    
- **FW IPv6:** zasady analogiczne do IPv4 dla hostów publicznych (WWW/SSH/QUIC/ICMPv6).
- **Do Internetu (IPv6):** allow z VLAN 300 i z listy `Pracownie` (INF3 v6).
    

---

####  Porty fizyczne i mosty

- **bridge-Cyfronet_IPv4:** łączy `ether1-Internet_Cyfronet_IPv4` z `ether4-MChE_Router` (porty dodane do bridge).    
- **bridge-Cyfronet_IPv6:** port `ether2-Internet_Cyfronet_IPv6`.
- **bonding-sw-main (802.3ad):** `ether5–ether8` do **sw-main** (trunk wszystkie VLAN).
    

---

#### Rekomendacje (szybkie)

1. **Ujednolicenie DROP dla ICMP** – obecnie pełny DROP ICMP w FORWARD po limicie może psuć PMTUD oraz niektóre aplikacje; rozważyć pozostawienie `destination-unreachable (type 3)` i `time-exceeded (11)` w FORWARD.
    
2. **OSE jako osobny VRF** – obecnie wykorzystywana `routing-table=ose` z rule dla 172.16.0.0/22 działa, ale warto rozważyć pełny **VRF** dla izolacji (PBR → VRF), by uniknąć przypadkowych przecieków polityk.
    
3. **Firewall publiczny (VLAN 300)** – dodać jawne `DROP with log` po wyjątkach dla każdego hosta/portu (częściowo już jest) i rozważyć **address-list** dla uproszczenia publikacji.
    
4. **DHCP Relay – failover** – pojedynczy serwer `10.10.0.7` jest SPoF. Warto dodać drugi (helper) lub HA.
    
5. **Monitoring PIM/IGMP** – sprawdzić querier na VLANach z multicastem (czy CCR pełni rolę queriera) i stale weryfikować listy grup.
    
6. **VPN spójność** – OVPN TCP/1194 bywa mniej wydajny; jeśli możliwe, preferować **WireGuard/IKEv2** dla wszystkich nowych dostępów.
    
7. **Lista** `**pracownie**` – ma wiele wyjątków per-drukarki; rozważyć drukowanie przez serwer **pull-print** w VLAN 20 i uproszczenie FW.
    
8. **NTP/DNS** – router korzysta z DNS Google i lokalnego `193.193.65.228`; upewnić się, że polityka DNS-over-HTTPS/QUIC jest świadoma (port 443/UDP dopuszczony na wybrane hosty publiczne).
    
9. **Logowanie** – włączyć zewnętrzny syslog (aktualnie `action=memory/disk`), zwłaszcza dla logów z blacklist/portscan/DC1-DC2_DROP.
    
10. **Backup konfiguracji + Let’s Encrypt** – zautomatyzować odnowienia cert (już obecny cert _letsencrypt-autogen_) i kopie `/export hide-sensitive` na bezpieczny zdalny zasób.
    

---

#### Załącznik: mapowanie stref → NAT (tabela szybka)

| Strefa                        | Zakres                                              | Wyjście          | Translacja         |
| ----------------------------- | --------------------------------------------------- | ---------------- | ------------------ |
| Pracownie                     | 192.168.8.0/21, 192.168.30.0/24, 172.20.0.0/24      | Cyfronet IPv4    | 193.193.65.245–251 |
| Management/VoiceIP/Monitoring | 10.10.0.0/16, 10.6.0.0/24, 10.7.0.0/24, 10.5.0.0/24 | Cyfronet IPv4    | 193.193.65.253     |
| Staff                         | 192.168.2.0/24, 172.19.0.0/23                       | Cyfronet IPv4    | 193.193.65.252     |
| Teacher                       | 172.17.0.0/23, 172.18.0.0/24                        | Cyfronet IPv4    | 193.193.65.251     |
| Classroom                     | 172.20.0.0/24                                       | Cyfronet IPv4    | 193.193.65.254     |
| WiFi Uczniowie                | 172.16.0.0/22                                       | **OSE (ether3)** | masquerade         |
| Publiczne (VLAN 300)          | 193.193.65.224/27, 2001:6d8:1000::/64               | Cyfronet\|       | bez NAT            |
|                               |                                                     |                  |                    |
|                               |                                                     |                  |                    |
