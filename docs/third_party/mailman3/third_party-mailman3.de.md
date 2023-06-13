# Installation von mailcow und Mailman 3 auf der Basis von dockerisierten Versionen

!!! info
    Diese Anleitung ist eine Kopie von [dockerized-mailcow-mailman](https://github.com/g4rf/dockerized-mailcow-mailman). Bitte posten Sie Probleme, Fragen und Verbesserungen in den [issue tracker](https://github.com/g4rf/dockerized-mailcow-mailman/issues) dort.

!!! warning "Warnung"
    mailcow ist nicht verantwortlich für Datenverlust, Hardwareschäden oder kaputte Tastaturen. Diese Anleitung kommt ohne jegliche Garantie. Macht Backups bevor ihr anfangt, **Kein Backup kein Mitleid!**

## Einleitung

Diese Anleitung zielt darauf ab, [mailcow-dockerized](https://github.com/mailcow/mailcow-dockerized) mit [docker-mailman] (https://github.com/maxking/docker-mailman) zu installieren und zu konfigurieren und einige nützliche Skripte bereitzustellen. Eine wesentliche Bedingung ist, dass _mailcow_ und _Mailman_ in ihren eigenen Installationen für unabhängige Updates erhalten bleiben.

Es gibt einige Anleitungen und Projekte im Internet, aber sie sind nicht auf dem neuesten Stand und/oder unvollständig in der Dokumentation oder Konfiguration. Diese Anleitung basiert auf der Arbeit von:

- [mailcow-mailman3-dockerized](https://github.com/Shadowghost/mailcow-mailman3-dockerized) von [Shadowghost](https://github.com/Shadowghost)
- [mailman-mailcow-integration](https://gitbucket.pgollor.de/docker/mailman-mailcow-integration)

Nach Beendigung dieser Anleitung werden [mailcow-dockerized](https://github.com/mailcow/mailcow-dockerized) und [docker-mailman](https://github.com/maxking/docker-mailman) laufen und _Apache_ als Reverse-Proxy wird die Web-Frontends bedienen.

Das verwendete Betriebssystem ist ein _Ubuntu 20.04 LTS_.

## Installation

Diese Anleitung basiert auf verschiedenen Schritten:

1. DNS-Einrichtung
1. Installieren Sie _Apache_ als Reverse Proxy
1. Beziehen Sie SSL-Zertifikate mit _Let's Encrypt_.
1. Installieren Sie _mailcow_ mit _Mailman_ Integration
1. Installieren Sie _Mailman_.
1. Ausführen

### DNS-Einrichtung

Der größte Teil der Konfiguration ist in *mailcow*s [DNS Konfiguration](../../prerequisite/prerequisite-dns.de.md) enthalten. Nachdem diese Einrichtung abgeschlossen ist, fügen Sie eine weitere Subdomain für _Mailman_ hinzu, z.B. `lists.example.org`, die auf denselben Server zeigt:

```
# Name Typ Wert
lists IN A 1.2.3.4
lists IN AAAA dead:beef
```

### Installieren Sie einen Reverse Proxy

Folgen Sie der [Reverse Proxy Anleitung](../../post_installation/reverse-proxy/r_p.md).

### Installieren Sie _mailcow_ mit _Mailman_ Integration

#### Installieren Sie mailcow

Folgen Sie der [mailcow installation](../../i_u_m/i_u_m_install.de.md). **Schritt 5 auslassen und nicht mit starten!**

#### mailcow konfigurieren

Dies ist auch **Schritt 4** in der offiziellen _mailcow-Installation_ (`nano mailcow.conf`). Passen Sie also Ihre Bedürfnisse an und ändern Sie die folgenden Variablen:

```
HTTP_PORT=18080 # verwenden Sie nicht 8080, da mailman es braucht
HTTP_BIND=127.0.0.1 #
HTTPS_PORT=18443 # Sie können 8443 verwenden
HTTPS_BIND=127.0.0.1 # # HTTPS_BIND=127.0.0.1

SKIP_LETS_ENCRYPT=y # Der Reverse Proxy wird die SSL-Verifizierung durchführen

SNAT_TO_SOURCE=1.2.3.4 # ändern Sie dies in Ihre IPv4
SNAT6_TO_SOURCE=dead:beef # Ändern Sie dies in Ihre globale IPv6
```

#### Mailman-Integration hinzufügen

Erstelle die Datei `/opt/mailcow-dockerized/docker-compose.override.yml` (z.B. mit `nano`) und füge die folgenden Zeilen hinzu:

```
version: '2.1'

services:
  postfix-mailcow:
    volumes:
      - /opt/mailman:/opt/mailman
    networks:
      - docker-mailman_mailman

networks:
  docker-mailman_mailman:
    external: true
```

Das zusätzliche Volume wird von _Mailman_ verwendet, um zusätzliche Konfigurationsdateien für _mailcow postfix_ zu generieren. Das externe Netzwerk wird von _Mailman_ erstellt und verwendet. _mailcow_ benötigt es, um eingehende Listenmails an _Mailman_ zu liefern.

Erstellen Sie die Datei `/opt/mailcow-dockerized/data/conf/postfix/extra.cf` (z.B. mit `nano`) und fügen Sie die folgenden Zeilen hinzu:

```
# mailman

recipient_delimiter = +
unknown_local_recipient_reject_code = 550
owner_request_special = no

local_recipient_maps =
  regexp:/opt/mailman/core/var/data/postfix_lmtp,
  proxy:unix:passwd.byname,
  $alias_maps
virtual_mailbox_maps =
  proxy:mysql:/opt/postfix/conf/sql/mysql_virtual_mailbox_maps.cf,
  regexp:/opt/mailman/core/var/data/postfix_lmtp
transport_maps =
  pcre:/opt/postfix/conf/custom_transport.pcre,
  pcre:/opt/postfix/conf/local_transport,
  proxy:mysql:/opt/postfix/conf/sql/mysql_relay_ne.cf,
  proxy:mysql:/opt/postfix/conf/sql/mysql_transport_maps.cf,
  regexp:/opt/mailman/core/var/data/postfix_lmtp
relay_domains =
  proxy:mysql:/opt/postfix/conf/sql/mysql_virtual_relay_domain_maps.cf,
  regexp:/opt/mailman/core/var/data/postfix_domains
relay_recipient_maps =
  proxy:mysql:/opt/postfix/conf/sql/mysql_relay_recipient_maps.cf,
  regexp:/opt/mailman/core/var/data/postfix_lmtp
```

Da wir hier die _mailcow postfix_ Konfiguration überschreiben, kann dieser Schritt Ihre normalen Mailtransporte unterbrechen. Überprüfen Sie die [originalen Konfigurationsdateien](https://github.com/mailcow/mailcow-dockerized/tree/master/data/conf/postfix), wenn sich etwas geändert hat.

#### SSL-Zertifikate

Da wir _mailcow_ als Proxy verwenden, müssen wir die SSL-Zertifikate in die _mailcow_-Dateistruktur kopieren. Diese Aufgabe wird das Skript [renew-ssl.sh](https://github.com/g4rf/dockerized-mailcow-mailman/tree/master/scripts/renew-ssl.sh) für uns erledigen:

- Kopieren Sie die Datei nach `/opt/mailcow-dockerized`
- Ändere **mailcow_HOSTNAME** in deinen _mailcow_ Hostnamen
- Machen Sie es ausführbar (`chmod a+x renew-ssl.sh`)
- **Noch nicht ausführen, da wir zuerst Mailman benötigen**

Sie müssen einen _cronjob_ erstellen, so dass neue Zertifikate kopiert werden. Führen Sie ihn als _root_ oder _sudo_ aus:

```
crontab -e
```

Um das Skript jeden Tag um 5 Uhr morgens laufen zu lassen, fügen Sie hinzu:

```
0 5 * * * /opt/mailcow-dockerized/renew-ssl.sh
```

### Installieren Sie _Mailman_.

Befolgen Sie im Wesentlichen die Anweisungen unter [docker-mailman](https://github.com/maxking/docker-mailman). Da sie sehr umfangreich sind, ist hier in aller Kürze beschrieben, was zu tun ist:

Als _root_ oder _sudo_:

```
cd /opt
mkdir -p mailman/core
mkdir -p mailman/web
git clone https://github.com/maxking/docker-mailman
cd docker-mailman
```

#### Mailman konfigurieren

Erstellen Sie einen langen Schlüssel für _Hyperkitty_, z.B. mit dem Linux-Befehl `cat /dev/urandom | tr -dc a-zA-Z0-9 | head -c30; echo`. Speichern Sie diesen Schlüssel vorerst als HYPERKITTY_KEY.

Erstellen Sie ein langes Passwort für die Datenbank, z. B. mit dem Linux-Befehl `cat /dev/urandom | tr -dc a-zA-Z0-9 | head -c30; echo`. Speichern Sie dieses Passwort zunächst als DBPASS.

Erstellen Sie einen langen Schlüssel für _Django_, z. B. mit dem Linux-Befehl `cat /dev/urandom | tr -dc a-zA-Z0-9 | head -c30; echo`. Speichern Sie diesen Schlüssel für einen Moment als DJANGO_KEY.

Erstellen Sie die Datei `/opt/docker-mailman/docker compose.override.yaml` und ersetzen Sie `HYPERKITTY_KEY`, `DBPASS` und `DJANGO_KEY` durch die generierten Werte:

```
version: '2'

services:
  mailman-core:
    environment:
    - DATABASE_URL=postgres://mailman:DBPASS@database/mailmandb
    - HYPERKITTY_API_KEY=HYPERKITTY_KEY
    - TZ=Europe/Berlin
    - MTA=postfix
    restart: always
    networks:
      - mailman

  mailman-web:
    environment:
    - DATABASE_URL=postgres://mailman:DBPASS@database/mailmandb
    - HYPERKITTY_API_KEY=HYPERKITTY_KEY
    - TZ=Europe/Berlin
    - SECRET_KEY=DJANGO_KEY
    - SERVE_FROM_DOMAIN=MAILMAN_DOMAIN # e.g. lists.example.org
    - MAILMAN_ADMIN_USER=admin # the admin user
    - MAILMAN_ADMIN_EMAIL=admin@example.org # the admin mail address
    - UWSGI_STATIC_MAP=/static=/opt/mailman-web-data/static
    restart: always

  database:
    environment:
    - POSTGRES_PASSWORD=DBPASS
    restart: always
```

Bei `mailman-web` geben Sie die korrekten Werte für `SERVE_FROM_DOMAIN` (z.B. `lists.example.org`), `MAILMAN_ADMIN_USER` und `MAILMAN_ADMIN_EMAIL` ein. Sie benötigen die Admin-Zugangsdaten, um sich in der Web-Oberfläche (_Pistorius_) anzumelden. Um **das Passwort zum ersten Mal** zu setzen, verwenden Sie die Funktion _Passwort vergessen_ im Webinterface.

Über andere Konfigurationsoptionen lesen Sie die Dokumentationen [Mailman-web](https://github.com/maxking/docker-mailman#mailman-web-1) und [Mailman-core](https://github.com/maxking/docker-mailman#mailman-core-1).

#### Konfigurieren Sie Mailman core und Mailman web

Erstellen Sie die Datei `/opt/mailman/core/mailman-extra.cfg` mit dem folgenden Inhalt. `mailman@example.org` sollte auf ein gültiges Postfach oder eine Umleitung verweisen.

```
[mailman]
default_language: de
site_owner: mailman@example.org
```

Erstellen Sie die Datei `/opt/mailman/web/settings_local.py` mit dem folgenden Inhalt. `mailman@example.org` sollte auf ein gültiges Postfach oder eine Umleitung verweisen.

```
# Gebietsschema
LANGUAGE_CODE = 'de-de'

# soziale Authentifizierung deaktivieren
MAILMAN_WEB_SOCIAL_AUTH = []

# ändern
DEFAULT_FROM_EMAIL = 'mailman@example.org'

DEBUG = False
```

Sie können `LANGUAGE_CODE` und `SOCIALACCOUNT_PROVIDERS` an Ihre Bedürfnisse anpassen.

### Ausführen

Ausführen (als _root_ oder _sudo_)

=== "docker compose (Plugin)"

    ``` bash
    a2ensite mailcow.conf
    a2ensite mailman.conf
    systemctl restart apache2

    cd /opt/docker-mailman
    docker compose pull
    docker compose up -d

    cd /opt/mailcow-dockerized/
    docker compose pull
    ./renew-ssl.sh
    ```

=== "docker-compose (Standalone)"

    ``` bash
    a2ensite mailcow.conf
    a2ensite mailman.conf
    systemctl restart apache2

    cd /opt/docker-mailman
    docker-compose pull
    docker-compose up -d

    cd /opt/mailcow-dockerized/
    docker-compose pull
    ./renew-ssl.sh
    ```

**Warten Sie ein paar Minuten!** Die Container müssen ihre Datenbanken und Konfigurationsdateien erstellen. Dies kann bis zu 1 Minute und mehr dauern.

## Bemerkungen

### Neue Listen werden von Postfix nicht sofort erkannt

Wenn man eine neue Liste anlegt und versucht, sofort eine E-Mail zu versenden, antwortet _postfix_ mit `Benutzer existiert nicht`, weil _postfix_ die Liste noch nicht an _Mailman_ übergeben hat. Die Konfiguration unter `/opt/mailman/core/var/data/postfix_lmtp` wird nicht sofort aktualisiert. Wenn Sie die Liste sofort benötigen, starten Sie _postifx_ manuell neu:

=== "docker compose (Plugin)"

    ``` bash
    cd /opt/mailcow-dockerized
    docker compose restart postfix-mailcow
    ```

=== "docker-compose (Standalone)"

    ``` bash
    cd /opt/mailcow-dockerized
    docker-compose restart postfix-mailcow
    ```

## Update

**mailcow** hat sein eigenes Update-Skript in `/opt/mailcow-dockerized/update.sh`, [siehe die Dokumentation](../../i_u_m/i_u_m_update.de.md).

Für **Mailman** holen Sie sich einfach die neueste Version aus dem [github repository](https://github.com/maxking/docker-mailman).

## Sicherung

**mailcow** hat ein eigenes Backup-Skript. [Lies die Docs](../../backup_restore/b_n_r-backup.de.md) für weitere Informationen.

**Mailman** gibt keine Backup-Anweisungen in der README.md an. Im [gitbucket von pgollor](https://gitbucket.pgollor.de/docker/mailman-mailcow-integration/blob/master/mailman-backup.sh) befindet sich ein Skript, das hilfreich sein könnte.

## ToDo

### Skript installieren

Schreiben Sie ein Skript wie in [mailman-mailcow-integration/mailman-install.sh](https://gitbucket.pgollor.de/docker/mailman-mailcow-integration/blob/master/mailman-install.sh), da viele der Schritte automatisierbar sind.

1. Fragen Sie alle Konfigurationsvariablen ab und erstellen Sie Passwörter und Schlüssel.
2. Führen Sie eine (halb)automatische Installation durch.
3. Viel Spaß!
