# kafka-config

Ansible-Playbook zum gezielten Überschreiben einzelner Konfigurationswerte in der
bereits auf den Kafka-VMs vorhandenen `/etc/kafka/server.properties`, je nach
Environment (`entw`, `test`, `atu`, `prod`).

**Wichtig:** Die Datei auf der VM wird **nicht** komplett ersetzt. Es werden nur
die Zeilen der in `kafka_config_overrides` gepflegten Keys geändert bzw. ergänzt.
Alles andere in der Datei — inklusive Secrets und SSL-Konfiguration — bleibt
unangetastet. Es werden außerdem keine Owner/Group/Rechte (`chmod`/`chown`) der
Datei verändert, nur der Inhalt.

## Struktur

```
ansible.cfg
playbook.yml
inventory/
  entw/hosts.ini  + group_vars/all.yml  → kafka_config_overrides (Werte für entw)
  test/hosts.ini  + group_vars/all.yml  → kafka_config_overrides (Werte für test)
  atu/hosts.ini   + group_vars/all.yml  → kafka_config_overrides (Werte für atu)
  prod/hosts.ini  + group_vars/all.yml  → kafka_config_overrides (Werte für prod)
```

## Ausführung

Pro Environment das passende Inventory angeben:

```bash
ansible-playbook -i inventory/entw/hosts.ini playbook.yml
ansible-playbook -i inventory/test/hosts.ini playbook.yml
ansible-playbook -i inventory/atu/hosts.ini playbook.yml
ansible-playbook -i inventory/prod/hosts.ini playbook.yml
```

## Was das Playbook macht

1. Für jeden Key/Value aus `kafka_config_overrides` (definiert in `group_vars/all.yml`
   des jeweiligen Inventory-Ordners) sucht `ansible.builtin.lineinfile` die Zeile
   `key=...` in `/etc/kafka/server.properties` und ersetzt nur deren Wert. Existiert
   der Key noch nicht in der Datei, wird die Zeile ergänzt.
2. Vor jeder Änderung wird automatisch ein Backup der Datei angelegt (`backup: true`).
3. Die Datei muss auf der VM bereits existieren — das Playbook legt sie nicht neu an
   (kein `create: true`), damit nie versehentlich eine leere Datei ohne Secrets/SSL
   entsteht.
4. Owner, Group und Dateirechte werden nicht angefasst.
5. Der Handler startet `kafka.service` per `systemd`-Modul neu — aber **nur wenn
   sich mindestens ein Wert tatsächlich geändert hat**, nicht bei jedem Lauf
   (idempotent). Es wird kein `become`/sudo verwendet; der `ansible_user` muss auf
   den Ziel-VMs selbst genug Rechte haben, um `/etc/kafka/server.properties` zu
   editieren und `kafka.service` neu zu starten.

## Ablauf für einen Config-Change

1. In `inventory/<env>/group_vars/all.yml` das Dict `kafka_config_overrides` anpassen
   — nur die Keys eintragen, die geändert werden sollen (Wert für Wert).
2. Änderung committen und reviewen (Pull Request), bevor sie ausgerollt wird.
3. Playbook gegen das passende Inventory laufen lassen:
   ```bash
   ansible-playbook -i inventory/<env>/hosts.ini playbook.yml
   ```
4. Bei Bedarf vorher mit `--check --diff` einen Trockenlauf machen:
   ```bash
   ansible-playbook -i inventory/<env>/hosts.ini playbook.yml --check --diff
   ```
5. Der Kafka-Service wird automatisch neu gestartet, sobald sich mindestens ein Wert
   in `/etc/kafka/server.properties` geändert hat — kein manueller Restart nötig.
6. Rollout in `prod` erst nach erfolgreichem Test in `entw`/`test`/`atu` durchführen.

## Einmalige Vorbereitung / Anpassungen vor dem ersten Lauf

- Platzhalter-Hostnamen in den vier `inventory/*/hosts.ini` durch die echten
  VM-Namen/IPs ersetzen.
- `ansible_user` in den `hosts.ini`-Dateien anpassen, falls nicht `ansible`.
- Service-Name `kafka.service` in `playbook.yml` (`kafka_service_name`) anpassen,
  falls abweichend.
- `kafka_config_overrides` je Environment in `inventory/<env>/group_vars/all.yml`
  mit den tatsächlich gewünschten Werten befüllen (aktuell nur Beispielwerte).
