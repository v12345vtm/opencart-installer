Hieronder staat één script, `opencart.sh`, met alle fixes en de commando's voor installeren, testen en verwijderen. Ik kan het niet op jouw server draaien, dus ik heb het zorgvuldig nagelopen. Stuur me de uitvoer als er ergens iets misgaat.

Het script bevat deze fixes:
- De `config.php`- en `admin/config.php`-bestanden worden vóór de wizard aangemaakt.
- De storage staat in `/srv/opencart-storage`, buiten de webroot. Daardoor heb je geen rechten op `/var` nodig.
- De installatiemap blijft staan tot de wizard klaar is. Pas bij `finish` wordt hij verwijderd.
- Na de wizard stelt `finish` het storage-pad in, verwijdert de tijdelijke symlink en controleert of de webshop, het beheer en de storage correct reageren.
- De Twig-bibliotheek wordt gecontroleerd, en anders geïnstalleerd met Composer.
- PCRE JIT staat uit.
- Het database-wachtwoord wordt in de console getoond en opgeslagen in `/root/opencart-db-gegevens.txt`.

Maak het bestand aan met `nano opencart.sh`, plak de inhoud, sla op met Ctrl+O, Enter, en sluit af met Ctrl+X. Het oude `install-opencart.sh` kun je verwijderen met `rm install-opencart.sh`.

```bash
#!/usr/bin/env bash
# =============================================================================
#  OpenCart 4 beheerscript voor Ubuntu Server 26.04
#  Apache + MariaDB + PHP, storage buiten de webroot
#
#  sudo bash opencart.sh install           Alles installeren, daarna de wizard
#  sudo bash opencart.sh finish            Na de wizard: opruimen, storage instellen
#  sudo bash opencart.sh password          Database-gegevens tonen
#  sudo bash opencart.sh info              IP-adres en URL's tonen
#  sudo bash opencart.sh uninstall         OpenCart verwijderen (vraagt bevestiging)
#  sudo bash opencart.sh uninstall --yes   Verwijderen zonder bevestiging
# =============================================================================

set -euo pipefail

WEB_DIR="/var/www/opencart"
STORAGE_DIR="/srv/opencart-storage"
DB_NAME="opencart"
DB_USER="ocuser"
CREDS_FILE="/root/opencart-db-gegevens.txt"
APACHE_CONF="/etc/apache2/sites-available/opencart.conf"
OC_TAG="${OC_TAG:-}"   # leeg = nieuwste 4.x-release, of bv. OC_TAG=4.1.0.4

# ----- hulpfuncties -----------------------------------------------------------
step() { echo; echo ">> $*"; }
good() { echo "   OK: $*"; }
die()  { echo; echo "FOUT: $*" >&2; exit 1; }

need_root() {
  [[ $EUID -eq 0 ]] || die "Start dit commando met sudo."
}

server_ip() {
  hostname -I | awk '{print $1}'
}

# Geeft de HTTP-statuscode van een pad op de server terug.
check_url() {
  curl -s -o /dev/null -w '%{http_code}' "http://localhost$1" || true
}

# Zet DIR_STORAGE op de echte storage-map. Alleen de regel define('DIR_STORAGE'...)
# wordt aangepast; DIR_CACHE, DIR_LOGS enzovoort blijven gelijk.
set_storage_path() {
  local f="$1"
  sed -i "/^[[:space:]]*define('DIR_STORAGE'/c\\define('DIR_STORAGE', '${STORAGE_DIR}/');" "$f"
  grep -q "define('DIR_STORAGE', '${STORAGE_DIR}/');" "$f" \
    || die "Kon DIR_STORAGE niet aanpassen in $f"
  chown www-data:www-data "$f"
}

# ----- install ----------------------------------------------------------------
cmd_install() {
  need_root
  if [[ -e "$WEB_DIR" || -e "$STORAGE_DIR" ]]; then
    die "$WEB_DIR of $STORAGE_DIR bestaat al. Draai eerst: sudo bash opencart.sh uninstall"
  fi

  local DB_PASS PHPV SRC
  DB_PASS="$(openssl rand -hex 16)"

  step "1/8 Pakketten installeren"
  export DEBIAN_FRONTEND=noninteractive
  apt-get update -qq
  apt-get install -y -qq apache2 mariadb-server \
    php libapache2-mod-php php-mysql php-curl php-gd php-mbstring \
    php-zip php-xml php-intl php-bcmath unzip curl python3 openssl
  good "pakketten geïnstalleerd"

  step "2/8 PHP instellen (PCRE JIT uit)"
  PHPV=$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')
  echo "pcre.jit=0" > "/etc/php/${PHPV}/mods-available/pcrejit.ini"
  phpenmod pcrejit
  good "PHP ${PHPV}, pcre.jit=0"

  step "3/8 OpenCart-versie bepalen"
  if [[ -z "$OC_TAG" ]]; then
    OC_TAG=$(curl -fsSL "https://api.github.com/repos/opencart/opencart/releases?per_page=30" \
      | python3 -c '
import sys, json
for r in json.load(sys.stdin):
    t = r["tag_name"]
    if t.lstrip("v").startswith("4.") and not r.get("prerelease") and not r.get("draft"):
        print(t)
        break
') || true
  fi
  [[ -n "$OC_TAG" ]] || die "Geen OpenCart 4-release gevonden. Probeer: sudo OC_TAG=4.1.0.4 bash opencart.sh install"
  good "versie ${OC_TAG}"

  step "4/8 OpenCart downloaden en uitpakken"
  TMP_DIR=$(mktemp -d)
  trap 'rm -rf "${TMP_DIR:-}"' EXIT
  curl -fsSL -o "$TMP_DIR/oc.zip" "https://github.com/opencart/opencart/archive/refs/tags/${OC_TAG}.zip"
  unzip -q "$TMP_DIR/oc.zip" -d "$TMP_DIR/src"
  SRC=$(find "$TMP_DIR/src" -mindepth 2 -maxdepth 2 -type d -name upload -print -quit)
  [[ -n "$SRC" ]] || die "Map 'upload' niet gevonden in de download."
  good "download uitgepakt"

  step "5/8 Bestanden plaatsen, configuratie maken, storage verplaatsen"
  mkdir -p "$WEB_DIR"
  cp -a "$SRC/." "$WEB_DIR/"
  find "$WEB_DIR" -name '.htaccess.txt' | while read -r f; do mv "$f" "${f%.txt}"; done

  [[ -f "$WEB_DIR/config-dist.php" && -f "$WEB_DIR/admin/config-dist.php" ]] \
    || die "config-dist.php ontbreekt in de download."
  cp "$WEB_DIR/config-dist.php" "$WEB_DIR/config.php"
  cp "$WEB_DIR/admin/config-dist.php" "$WEB_DIR/admin/config.php"

  mkdir -p "$STORAGE_DIR"
  cp -a "$WEB_DIR/system/storage/." "$STORAGE_DIR/"
  rm -rf "$WEB_DIR/system/storage"
  ln -s "$STORAGE_DIR" "$WEB_DIR/system/storage"
  for d in cache download logs session upload marketplace modification; do
    mkdir -p "$STORAGE_DIR/$d"
  done
  good "storage in ${STORAGE_DIR}, tijdelijk gekoppeld voor de installer"

  step "6/8 Twig-bibliotheek (vendor) controleren"
  if [[ -d "$STORAGE_DIR/vendor/twig" ]]; then
    good "vendor/ is aanwezig"
  elif [[ -f "$STORAGE_DIR/composer.json" ]]; then
    command -v composer >/dev/null || apt-get install -y -qq composer
    (cd "$STORAGE_DIR" && COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader --no-interaction)
    good "vendor/ geïnstalleerd met Composer"
  else
    die "Geen vendor/twig en geen composer.json in $STORAGE_DIR."
  fi

  chown -R www-data:www-data "$WEB_DIR" "$STORAGE_DIR"
  find "$WEB_DIR" -type d -exec chmod 755 {} +
  find "$WEB_DIR" -type f -exec chmod 644 {} +
  find "$STORAGE_DIR" -type d -exec chmod 755 {} +
  find "$STORAGE_DIR" -type f -exec chmod 644 {} +
  for f in "$WEB_DIR/config.php" "$WEB_DIR/admin/config.php" "$STORAGE_DIR/logs"; do
    runuser -u www-data -- test -w "$f" || die "www-data kan niet schrijven naar $f"
  done
  good "rechten ingesteld"

  step "7/8 Database aanmaken"
  systemctl enable --now mariadb >/dev/null
  mariadb <<SQL
CREATE DATABASE IF NOT EXISTS \`${DB_NAME}\` CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER IF NOT EXISTS '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PASS}';
ALTER USER '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PASS}';
GRANT ALL PRIVILEGES ON \`${DB_NAME}\`.* TO '${DB_USER}'@'localhost';
FLUSH PRIVILEGES;
SQL
  cat > "$CREDS_FILE" <<EOF
Database host:       localhost
Database naam:       ${DB_NAME}
Database gebruiker:  ${DB_USER}
Database wachtwoord: ${DB_PASS}
Tabel-prefix:        oc_
EOF
  chmod 600 "$CREDS_FILE"
  good "database '${DB_NAME}' en gebruiker '${DB_USER}' klaar"

  step "8/8 Apache instellen en starten"
  cat > "$APACHE_CONF" <<EOF
<VirtualHost *:80>
    DocumentRoot ${WEB_DIR}
    <Directory ${WEB_DIR}>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    # Storage (logs, cache, uploads, vendor) is niet via de browser bereikbaar
    <Directory ${WEB_DIR}/system/storage>
        Require all denied
    </Directory>
    <Directory ${STORAGE_DIR}>
        Require all denied
    </Directory>
    ErrorLog \${APACHE_LOG_DIR}/opencart_error.log
    CustomLog \${APACHE_LOG_DIR}/opencart_access.log combined
</VirtualHost>
EOF
  a2enmod rewrite >/dev/null
  a2dissite 000-default.conf >/dev/null 2>&1 || true
  a2ensite opencart.conf >/dev/null
  systemctl enable --now apache2 >/dev/null
  systemctl restart apache2
  good "Apache draait"

  local IP
  IP=$(server_ip)
  echo
  echo "======================================================================"
  echo " OpenCart is geïnstalleerd. Doe nu de wizard in je browser:"
  echo
  echo "   http://${IP}/install/"
  echo
  echo " Database-gegevens voor de wizard:"
  echo "   Host:          localhost"
  echo "   Gebruiker:     ${DB_USER}"
  echo "   Wachtwoord:    ${DB_PASS}"
  echo "   Database:      ${DB_NAME}"
  echo "   Tabel-prefix:  oc_"
  echo
  echo " Klik in de wizard niet op 'Move' of 'Delete' in een storage-melding."
  echo " Na de wizard, op de server:   sudo bash opencart.sh finish"
  echo " Wachtwoord opnieuw tonen:     sudo bash opencart.sh password"
  echo "======================================================================"
}

# ----- finish (na de wizard) --------------------------------------------------
cmd_finish() {
  need_root
  [[ -f "$WEB_DIR/config.php" ]] || die "OpenCart is niet geïnstalleerd (geen $WEB_DIR/config.php)."
  grep -q "'${DB_NAME}'" "$WEB_DIR/config.php" \
    || die "De wizard lijkt niet afgerond: de database staat niet in config.php. Doe eerst de wizard in de browser."

  step "1/4 Installatiemap verwijderen"
  rm -rf "$WEB_DIR/install"
  good "install/ verwijderd"

  step "2/4 Tijdelijke storage-koppeling verwijderen"
  if [[ -L "$WEB_DIR/system/storage" ]]; then
    unlink "$WEB_DIR/system/storage"
    good "symlink verwijderd, de echte storage blijft in ${STORAGE_DIR}"
  elif [[ -d "$WEB_DIR/system/storage" ]]; then
    die "$WEB_DIR/system/storage is een echte map en geen symlink. Controleer dit eerst handmatig."
  else
    good "geen koppeling aanwezig"
  fi
  [[ -d "$STORAGE_DIR" ]] || die "Storage-map ${STORAGE_DIR} ontbreekt."

  step "3/4 Storage-pad in de configuratie instellen"
  set_storage_path "$WEB_DIR/config.php"
  set_storage_path "$WEB_DIR/admin/config.php"
  good "beide configuraties wijzen naar ${STORAGE_DIR}"

  step "4/4 Apache herladen en controleren"
  systemctl reload apache2
  echo "   Webshop         /                 -> $(check_url /)                  (verwacht 200)"
  echo "   Beheer          /admin/           -> $(check_url /admin/)            (verwacht 200 of 302)"
  echo "   Storage via web /system/storage/  -> $(check_url /system/storage/)   (verwacht 403 of 404)"
  echo
  echo "Klaar. Webshop: http://$(server_ip)/   Beheer: http://$(server_ip)/admin"
  echo "Sluit een eventuele storage-melding in het beheer met het kruisje."
}

# ----- password ---------------------------------------------------------------
cmd_password() {
  need_root
  [[ -f "$CREDS_FILE" ]] || die "Geen database-gegevens gevonden in ${CREDS_FILE}."
  echo
  cat "$CREDS_FILE"
  echo
}

# ----- info -------------------------------------------------------------------
cmd_info() {
  local IP
  IP=$(server_ip)
  echo "Server-IP:  ${IP}"
  echo "Wizard:     http://${IP}/install/   (alleen zolang de installatie nog bezig is)"
  echo "Webshop:    http://${IP}/"
  echo "Beheer:     http://${IP}/admin"
  if [[ -f "$WEB_DIR/config.php" ]]; then
    echo "Storage:    $(grep -h "define('DIR_STORAGE'" "$WEB_DIR/config.php" || echo onbekend)"
  else
    echo "Storage:    OpenCart is nog niet geïnstalleerd"
  fi
}

# ----- uninstall --------------------------------------------------------------
cmd_uninstall() {
  need_root
  if [[ "${1:-}" != "--yes" ]]; then
    echo "Dit verwijdert OpenCart volledig:"
    echo "  - ${WEB_DIR}"
    echo "  - ${STORAGE_DIR} (inclusief eventuele webshopdata)"
    echo "  - database '${DB_NAME}' en gebruiker '${DB_USER}'"
    echo "  - de Apache-site 'opencart' en ${CREDS_FILE}"
    echo "Apache, MariaDB en PHP blijven geïnstalleerd."
    echo
    read -r -p "Typ JA om door te gaan: " antwoord
    [[ "$antwoord" == "JA" ]] || die "Afgebroken, er is niets verwijderd."
  fi

  step "1/4 Apache-site verwijderen"
  a2dissite opencart.conf >/dev/null 2>&1 || true
  rm -f "$APACHE_CONF"
  a2ensite 000-default.conf >/dev/null 2>&1 || true
  systemctl reload apache2 >/dev/null 2>&1 || true
  good "Apache-site verwijderd, standaardsite hersteld"

  step "2/4 Bestanden verwijderen"
  rm -rf "$WEB_DIR" "$STORAGE_DIR"
  rm -f "$CREDS_FILE"
  good "${WEB_DIR} en ${STORAGE_DIR} verwijderd"

  step "3/4 Database verwijderen"
  if command -v mariadb >/dev/null && systemctl is-active --quiet mariadb; then
    mariadb -e "DROP DATABASE IF EXISTS \`${DB_NAME}\`; DROP USER IF EXISTS '${DB_USER}'@'localhost';"
    good "database en gebruiker verwijderd"
  else
    good "MariaDB draait niet, niets te verwijderen"
  fi

  step "4/4 Klaar"
  good "OpenCart is verwijderd. Opnieuw testen met: sudo bash opencart.sh install"
}

# ----- help -------------------------------------------------------------------
cmd_help() {
  cat <<'EOF'
Gebruik: sudo bash opencart.sh <commando>

  install            OpenCart installeren (daarna de wizard in de browser doen)
  finish             Na de wizard: installatiemap verwijderen en storage instellen
  password           Database-gegevens tonen
  info               IP-adres, URL's en storage-pad tonen
  uninstall          OpenCart volledig verwijderen (vraagt bevestiging)
  uninstall --yes    Hetzelfde, zonder bevestiging
EOF
}

# ----- hoofdprogramma ---------------------------------------------------------
case "${1:-help}" in
  install)   cmd_install ;;
  finish)    cmd_finish ;;
  password)  cmd_password ;;
  info)      cmd_info ;;
  uninstall) cmd_uninstall "${2:-}" ;;
  *)         cmd_help ;;
esac
```

**CLI-commando's op de server**

Dit zijn de commando's die je nodig hebt. Ze werken allemaal vanuit de map waar het script staat.

```bash
sudo bash opencart.sh install           # alles installeren, toont het wachtwoord
sudo bash opencart.sh finish            # na de wizard: opruimen en storage instellen
sudo bash opencart.sh password          # database-gegevens opnieuw tonen
sudo bash opencart.sh info              # IP-adres, URL's en storage-pad
sudo bash opencart.sh uninstall         # alles verwijderen, met bevestiging
sudo bash opencart.sh uninstall --yes   # alles verwijderen, zonder bevestiging
```

Om meermaals te testen, doe je dit:

```bash
sudo bash opencart.sh uninstall --yes
sudo bash opencart.sh install
# wizard doorlopen in de browser op http://IP-ADRES/install/
sudo bash opencart.sh finish
```

Let op: `uninstall` verwijdert ook je huidige webshop en de database. Apache, MariaDB en PHP blijven staan, zodat een nieuwe installatie sneller gaat.

Het script configureert geen wifi. Dat doe je met netplan, zoals eerder beschreven. HTTPS en het hernoemen van de admin-map laat ik er bewust uit, omdat dat voor een testomgeving niet nodig is.
