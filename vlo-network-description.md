# VLO – Opis sieci (na bazie konfiguracji MikroTik CCR1036)

_Plik przygotowany do wpięcia w Git / Obsidian. Źródło: export RouterOS 7.18.2._

> **Uwaga**: Diagram draw.io w oddzielnym pliku: `vlo-network-diagram.drawio`.

---

## 2) Adresacja i VLAN-y
- **Trunk do CORE (`bonding-sw-main`, LACP 4×GE)** – przenosi VLAN: 10,15,16,17,20,70,72,77,78,300,302,308,330,416–420.
- **Bramy L3 na CCR**:
  - VLAN 10 — `10.10.0.1/16` (**Management**)
  - VLAN 15 — `10.5.0.1/24` (**WiFi Management**)
  - VLAN 16 — `10.6.0.1/24` (**VoiceIP**)
  - VLAN 17 — `10.7.0.1/24` (**Monitoring**)
  - VLAN 20 — `10.20.0.1/24` (**Services**)
  - VLAN 70/72/77/78 — `10.4.70/72/77/78.253/24` (**MChE**)
  - VLAN 300 — `193.193.65.225/27` (**Publiczne IPv4**) + `2001:6d8:1000::1/64` (**IPv6**)
  - VLAN 302 — `192.168.2.1/24` (**Administracja**)
  - VLAN 308 — `192.168.8.1/21` (**Pracownie**)
  - VLAN 330 — `192.168.30.1/24` (**INF3**) + `2001:6d8:1000:33::1/64` (IPv6)
  - VLAN 416 — `172.16.0.1/22` (**WiFi uczniowie**)
  - VLAN 417 — `172.17.0.1/23` (**WiFi nauczyciele**)
  - VLAN 418 — `172.18.0.1/24` (**WiFi goście**)
  - VLAN 419 — `172.19.0.1/23` (**WiFi staff**)
  - VLAN 420 — `172.20.0.1/24` (**Classroom.VLO**)

**Publiczny IPv4 (VLAN 300):** `193.193.65.224/27` – m.in. `users (.227)`, `db (.226)`, `auth (.228)`, `www (.234)`, `voter (.237)`, `mssql (.238)`, `w2 (.239)`, `web (.248)`, `gielda (.249)`.
  
**Publiczny IPv6:** `2001:6d8:1000::/64` (VLAN 300) + trasa domyślna przez `2001:6d8:0:1106::1`.

---

## 3) Routing i strefowanie ruchu
- **Default IPv4 (`main`)**: `0.0.0.0/0 → 193.193.64.17` (Cyfronet).
- **Tablica `ose`**: `0.0.0.0/0 → 192.168.10.1` (dla łącza OSE).
- **Reguła PBR**: źródło `172.16.0.0/22` (**WiFi uczniowie**) → lookup only in table **`ose`**. NAT `masquerade` na `ether3-Internet_OSE`.
- **Default IPv6**: `::/0 → 2001:6d8:0:1106::1`.

---

## 4) DHCP
- **DHCP Relay** z VLAN: 15,16,17,302,308,416,417,418,419,420 → **serwer 10.10.0.7** (w Management).
- Router akceptuje role **DHCP Relay/Client** na interfejsach LAN (z wyjątkami WAN).

---

## 5) Usługi w VLAN 20 (Services)
- **DC1 / DC2** – `10.20.0.231 / 10.20.0.232` – porty AD/DNS/Kerberos/LDAP w łańcuchu `dc_forward` (whitelistowane źródła, m.in. *pracownie* i wybrane hosty publiczne).
- **Wazuh/OSSEC** – `10.20.0.228` (TCP `1514,1515`).
- **Dysk SMB** – `10.20.0.229` (TCP `445`) – dostęp: lista `staff`.
- **FOG Imaging** – `10.20.0.230` i `10.20.0.240` (HTTP/HTTPS/TFTP/NFS + multicast).
- **Kontroler Wi‑Fi** – `10.10.0.127` – powiązania z RADIUS (PPP), wyjątki FW dla AP i Pracowni.
- **RADIUS dla PPP** – `10.10.0.127` (`1818/1819`).

---

## 6) VPN i dostęp administracyjny
- **WireGuard (admin)** – UDP/13231; adresacja: `10.10.104.0/24`, `172.31.0.0/24`; peers: *pyton*, *maciej*. Lista `VPN-wireguard` dopuszczona w `input`.
- **OpenVPN (TCP/1194)** – TLS 1.2, profile `openvpn_frosty`.
- **IPsec IKEv2** – pula: `192.168.100.100–199`, split-include `0.0.0.0/0`.
- **Dostęp admin (SSH/Winbox/WWW-SSL)** – ograniczony do `address-list: management`.

---

## 7) Firewall – skrót polityki
- **Domyślnie DROP** po `established,related` (INPUT/FORWARD).
- **Blacklisting** – automatyczne listy: FireHOL Level1, SSH brute‑force, AbuseIPDB; łańcuchy `blacklist_chain_input/forward` + detekcja portscan (`psd`) i dodawanie do BL.
- **ICMP limity** – na INPUT/FORWARD (anty‑flood), drop rzadkich typów (timestamp).
- **Publika (VLAN 300)** – precyzyjne allow per host/usługa: HTTP/HTTPS/QUIC/DNS/SSH/ICMP, reszta DROP.
- **Routingi segmentowe** (wybrane):
  - `pracownie` → Internet (NAT 193.193.65.245–251) + dostęp do FOG i kontrolera Wi‑Fi.
  - `staff` → Internet (NAT 193.193.65.252) + ruch wewnętrzny `staff↔staff` allow.
  - `teacher` → Internet (NAT 193.193.65.251) + drukarki w Admin.
  - `voiceip`/`monitoring`/`management` → Internet (NAT 193.193.65.253).
  - `reszta` (catch‑all) → NAT 193.193.65.254.
  - `WiFi uczniowie` (`172.16.0.0/22`) → **tylko OSE**; wszystko inne DROP.

---

## 8) NAT – mapowanie źródeł
- **Pracownie** → `193.193.65.245–251`
- **Staff** → `193.193.65.252`
- **Teacher** → `193.193.65.251`
- **Mgmt/VoIP/Monitoring** → `193.193.65.253`
- **Pozostałe** → `193.193.65.254`
- **WiFi uczniowie** → **OSE** (`ether3-Internet_OSE`, `masquerade`).

---

## 9) Multicast (PIM-SM, FOG)
- **Instance**: `fog-multicast`.
- **Interfejsy**: VLAN 20 (**Services**), VLAN 330 (**INF3**).
- **Static RP**: `192.168.30.1` (GW VLAN 330).
- IGMP/PIM dopuszczone między `10.20.0.240` ↔ `pracownie` (deploy obrazów).

---

## 10) IPv6
- **Adresy na routerze**:
  - WANv6 (`bridge-Cyfronet_IPv6`): `2001:6d8:0:1106::2`
  - VLAN 300: `2001:6d8:1000::1/64`
  - VLAN 330: `2001:6d8:1000:33::1/64`
- **FW IPv6** – zasady analogiczne: allow dla publikowanych hostów (`WEB`, `VOTER`, `GIELDA`), ICMPv6, QUIC; dostęp do Internetu dla VLAN 300 i listy `Pracownie` (INF3 v6).

---

## 11) Porty fizyczne i mosty
- **bridge-Cyfronet_IPv4** – porty: `ether1-Internet_Cyfronet_IPv4`, `ether4-MChE_Router`.
- **bridge-Cyfronet_IPv6** – port: `ether2-Internet_Cyfronet_IPv6`.
- **bonding-sw-main (802.3ad)** – `ether5–ether8` do przełącznika głównego (trunk wszystkich VLAN).

---

## 12) Rekomendacje
1. **ICMP w FORWARD** – pozostawić `dest-unreachable` i `time-exceeded` (PMTUD) zamiast pełnego DROP po limicie.
2. **OSE → VRF** – rozważyć pełny VRF dla łącza OSE (obecnie PBR do tablicy `ose`).
3. **FW dla VLAN 300** – dodać jawne `DROP with log` po wyjątkach oraz standaryzować publikacje przez `address-list`.
4. **DHCP Relay HA** – dodać drugi serwer/helper dla `10.10.0.7` (SPoF).
5. **Multicast** – potwierdzić querier/IGMP snooping na VLAN-ach FOG.
6. **VPN** – preferować WireGuard/IKEv2 (OVPN/TCP zostawić dla zgodności).
7. **Drukowanie** – scentralizować przez serwer w VLAN 20 (pull‑print) i uprościć reguły FW.
8. **DNS/NTP** – weryfikacja polityk dla DoH/QUIC; potwierdzić priorytety `193.193.65.228` vs 8.8.8.8.
9. **Logi** – wysyłka do zewnętrznego sysloga (logi z BL/portscan/DC1‑DC2_DROP).
10. **Certy/backup** – automatyzacja odnowień Let's Encrypt i cykliczne `/export hide-sensitive` off-box.

---

## 13) Mapowanie stref → NAT (tabela)
| Strefa | Zakres | Wyjście | Translacja |
|---|---|---|---|
| Pracownie | 192.168.8.0/21, 192.168.30.0/24, 172.20.0.0/24 | Cyfronet IPv4 | 193.193.65.245–251 |
| Mgmt/VoiceIP/Monitoring | 10.10.0.0/16, 10.6.0.0/24, 10.7.0.0/24, 10.5.0.0/24 | Cyfronet IPv4 | 193.193.65.253 |
| Staff | 192.168.2.0/24, 172.19.0.0/23 | Cyfronet IPv4 | 193.193.65.252 |
| Teacher | 172.17.0.0/23, 172.18.0.0/24 | Cyfronet IPv4 | 193.193.65.251 |
| Classroom | 172.20.0.0/24 | Cyfronet IPv4 | 193.193.65.254 |
| WiFi uczniowie | 172.16.0.0/22 | **OSE (ether3)** | masquerade |
| Publiczne | 193.193.65.224/27, 2001:6d8:1000::/64 | Cyfronet | brak NAT |
