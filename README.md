# ansible-syslog-keepalived

Ansible projekt za namestitev visoko razpoložljivega zbiralnika oddaljenega
syslog prometa na **Red Hat Enterprise Linux 10**. Postavi `rsyslog` (UDP, TCP
in TCP/TLS) z rotacijo dnevnikov ter `keepalived` (VRRP) za samodejni preklop
plavajočega (virtualnega) IP naslova med dvema vozliščema. Zasnovan je za zagon
**kot root**, lokalno na ciljnem strežniku — z enim ukazom.

## Kaj naredi

Projekt sestavljata dve Ansible vlogi.

**Vloga `syslog`**

- preveri, da gre za RHEL družino, glavno različico 10 (sicer se ustavi);
- nastavi SELinux v način *permissive* (med izvajanjem in v `/etc/selinux/config`);
- ustavi in onemogoči `firewalld` (in `nftables`);
- namesti `rsyslog`, `rsyslog-gnutls` in `openssl`;
- ustvari samopodpisano potrdilo in ključ v `/etc/rsyslog.d/tls`;
- zapiše eno samostojno konfiguracijo `/etc/rsyslog.d/49-remote-collector.conf`
  s posluhi na UDP/514, TCP/514 in **TLS TCP/6514** (GnuTLS, samo šifriranje —
  `AuthMode anon`);
- prejete dnevnike zapisuje v `/syslog/<IP-pošiljatelja>/<zmožnost>.log`;
- konfiguracijo pred uveljavitvijo preveri z `rsyslogd -N1`;
- namesti pravilo za rotacijo dnevnikov v `/etc/logrotate.d/remote-syslog`.

**Vloga `keepalived`**

- namesti `keepalived` in zapiše `/etc/keepalived/keepalived.conf`;
- omogoča izbiro **prednostnega vozlišča** (MASTER/BACKUP) — keepalived pozna le
  stanji MASTER in BACKUP, vloge »proxy« ni;
- privzeto uporablja **nopreempt**: ko okvarjeno vozlišče znova zaživi,
  plavajočega IP ne prevzame nazaj in s tem prepreči nepotrebne prekinitve
  TCP/TLS povezav pošiljateljev;
- spremlja stanje storitve `rsyslog` in ob njeni okvari preklopi plavajoči IP na
  drugo vozlišče;
- podpira enega ali več plavajočih IP naslovov, lasten `virtual_router_id`,
  neobvezno *unicast* povezovanje in *notify* skripte.

## Zahteve

- Red Hat Enterprise Linux 10 (oz. združljiva RHEL družina, glavna različica 10).
- Dostop **root** na ciljnem strežniku.
- Za visoko razpoložljivost dve vozlišči v istem omrežnem segmentu (L2), ki si
  delita plavajoči IP.

Ansible ni potreben vnaprej — `bootstrap-and-run.sh` ga namesti sam.

## Hitra uporaba (priporočeno)

```bash
git clone https://github.com/<uporabnik>/ansible-syslog-keepalived.git
cd ansible-syslog-keepalived
sudo ./bootstrap-and-run.sh
```

Skript namesti Ansible, vpraša, ali želite tudi keepalived, nato pa zahteva
vlogo (MASTER/BACKUP), omrežni vmesnik, plavajoči IP, `virtual_router_id` in
geslo. Vse ostalo izvede sam.

## Ročni zagon prek Ansible

```bash
# vse skupaj
sudo ansible-playbook -i inventory/hosts.ini playbook.yml

# samo syslog
sudo ansible-playbook -i inventory/hosts.ini playbook.yml --tags syslog

# keepalived kot prednostno vozlišče z dvema plavajočima IP-jema
sudo ansible-playbook -i inventory/hosts.ini playbook.yml --tags keepalived \
  -e '{"keepalived_role":"MASTER","keepalived_interface":"ens192","keepalived_virtual_ips":["10.0.0.50","10.0.0.51"],"keepalived_vrid":71}'
```

## Samostojna namestitev keepalived (brez Ansible)

```bash
sudo ./keepalived-setup.sh                                  # interaktivno
sudo ./keepalived-setup.sh --role MASTER --interface ens192 --vip 10.0.0.50
```

## Visoka razpoložljivost (nopreempt)

Pri syslog prometu prek TCP/TLS gre za dolgožive povezave. Če bi obnovljeno
vozlišče plavajoči IP samodejno prevzelo nazaj (privzeto VRRP vedenje), bi se
vse povezave pošiljateljev na trenutno aktivno vozlišče prekinile. Zato je
privzeto vklopljen `nopreempt`:

- obe vozlišči se zaženeta kot **BACKUP**;
- vozlišče z višjo prioriteto (privzeto MASTER = 150, BACKUP = 100) zmaga na
  volitvah in drži plavajoči IP;
- ko to vozlišče odpove, IP prevzame drugo;
- ko prvotno vozlišče znova zaživi, IP **ostane** na trenutnem nosilcu (brez
  prekinitev).

Teža kontrolne skripte (`keepalived_track_weight`, privzeto `-60`) mora biti
večja od razlike prioritet med vozliščema, sicer se ob okvari `rsyslog`
plavajoči IP ne preseli. S privzetimi vrednostmi (razlika 50, teža -60) to
deluje pravilno.

Če želite klasično vedenje (prednostno vozlišče plavajoči IP vedno prevzame
nazaj), nastavite `keepalived_nopreempt: false`.

## Najpomembnejše spremenljivke

**syslog** (`roles/syslog/defaults/main.yml`)

| Spremenljivka | Privzeto | Pomen |
| --- | --- | --- |
| `syslog_enable_udp` / `_tcp` / `_tls` | `true` | vklop posameznih posluhov |
| `syslog_udp_port` / `_tcp_port` | `514` | vrata za navadni sprejem |
| `syslog_tls_port` | `6514` | vrata za TLS |
| `syslog_storage_dir` | `/syslog` | korenski imenik za prejete dnevnike |
| `syslog_logrotate_rotate` | `7` | število obdržanih rotacij |
| `syslog_disable_firewall` | `true` | ustavi in onemogoči požarni zid |
| `syslog_selinux_permissive` | `true` | SELinux v način permissive |

**keepalived** (`roles/keepalived/defaults/main.yml`)

| Spremenljivka | Privzeto | Pomen |
| --- | --- | --- |
| `keepalived_role` | `BACKUP` | prednostno vozlišče (določa prioriteto) |
| `keepalived_nopreempt` | `true` | brez prevzema nazaj ob obnovitvi |
| `keepalived_interface` | `eth0` | vmesnik za plavajoči IP |
| `keepalived_virtual_ip` | `192.168.1.100` | en plavajoči IP |
| `keepalived_virtual_ips` | `[]` | seznam plavajočih IP-jev (ima prednost) |
| `keepalived_vrid` | `51` | VRRP `virtual_router_id` |
| `keepalived_auth_pass` | `changeMe` | geslo VRRP (največ 8 znakov) |
| `keepalived_track_service` | `rsyslog` | nadzorovana storitev za preklop |
| `keepalived_track_weight` | `-60` | sprememba prioritete ob okvari storitve |

## Opomba o TLS

Privzeto je TLS v načinu `anon` — promet je **šifriran**, vendar se potrdila ne
preverjajo (niti strežniško niti odjemalsko). To zadošča za zaupnost prenosa.
Za medsebojno overjanje (`x509/name`) potrebujete lasten CA, odjemalska
potrdila, `DefaultNetstreamDriverCAFile` in seznam `permittedPeer`.

## Opozorila in omejitve

- **SELinux je namerno permissive.** Če pozneje preklopite na *enforcing*, morate
  označiti TLS vrata in pot za shranjevanje, npr.
  `semanage port -a -t syslogd_port_t -p tcp 6514` ter ustrezen kontekst na
  `/syslog`.
- `rsyslog` na RHEL teče kot **root** (uporabnika `syslog` in skupine `adm` kot
  na Debianu ni), zato so prejete datoteke v lasti uporabnika root.
- Geslo `keepalived_auth_pass` se pri VRRPv2 skrajša na **8 znakov**.
- Vozlišči zapisujeta v **ločena lokalna** shrambna imenika `/syslog`; vsako
  hrani le dnevnike, ki jih je prejelo, ko je bilo aktivno. Če podatke
  posredujete naprej (npr. v Splunk), na vsakem vozlišču zaženite svoj Universal
  Forwarder, sicer dnevniki niso zbrani na enem mestu.

## Struktura projekta

```
ansible-syslog-keepalived/
├── ansible.cfg
├── bootstrap-and-run.sh        # namestitev Ansible + interaktivni zagon
├── keepalived-setup.sh         # samostojna namestitev keepalived (brez Ansible)
├── playbook.yml
├── inventory/
│   └── hosts.ini
└── roles/
    ├── syslog/
    │   ├── defaults/main.yml
    │   ├── handlers/main.yml
    │   ├── tasks/main.yml
    │   └── templates/
    │       ├── remote-collector.conf.j2
    │       └── logrotate.j2
    └── keepalived/
        ├── defaults/main.yml
        ├── handlers/main.yml
        ├── tasks/main.yml
        └── templates/keepalived.conf.j2
```
