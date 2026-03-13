
# Installation och konfiguration för Drupal & grundläggande härdning på Ubuntu 24.04 LTS
Denna guide går igenom installationen av Drupal (och dess beroenden) på Ubuntu 24.04, följt av grundläggande systemhärdning enligt CIS-riktlinjer.
## 1. Uppdatera systemet

Som alla nya miljöer börjar vi med att uppdatera paketlistan och uppgradera till senaste versionen av paketen:

```bash
sudo apt update && sudo apt upgrade -y

```

## 2. Nätverk och Brandvägg (UFW)

Webbservern kommer behöva port 80 (HTTP) samt port 443 (HTTPS). Vi kommer även att använda oss av ssh så vi öppnar alla dessa i brandväggen med:

```bash
sudo ufw allow 22,80,443/tcp

```

Vi aktiverar brandväggen med:

```bash
sudo ufw default deny incoming
sudo ufw enable

```

## 3. Installera beroenden

Drupal har en del beroenden, de installera vi via:

```bash
sudo apt install mysql-server apache2 php8.3 libapache2-mod-php8.3 php8.3-mysql php8.3-gd php8.3-xml php8.3-mbstring

```

## 4. Ladda ned och förbered Drupal

Ladda ned Drupal och packa upp den med:

```bash
sudo wget https://www.drupal.org/download-latest/tar.gz -O drupal.tar.gz
sudo tar -xzvf drupal.tar.gz
sudo mv drupal-* /var/www/html/drupal

```

## 5. Databaskonfiguration (MySQL)

Vi skapar en databas genom att först logga in med `sudo mysql`:

```bash
sudo mysql

```

Väl inne i databasen skapar vi databasen för Drupal:

```sql
CREATE DATABASE drupal;

```

Vi skapar en användare med följande kommando (ett starkt lösenord genereras med fördel genom pwgen):

```sql
CREATE USER 'drupaladmin'@'localhost' IDENTIFIED BY '[lösenord här]';

```
Det är möjligt att Drupal inte kommer kunna kommunicera med MySql-servern då nyare versioner använder sig av `caching_sha2_password` för autentisering, något som kan ställa till med strul i konfigurationen i webbläsaren. I så fall, skapa databasanvändaren med denna parameter istället:
```sql
CREATE USER 'drupaladmin'@'localhost' IDENTIFIED WITH mysql_native_password BY '[lösenord här]';
```
Slutligen ger vi den nya användaren rättigheter för databasen och uppdaterar dessa med:

```sql
GRANT ALL PRIVILEGES ON drupal.* TO 'drupaladmin'@'localhost';
FLUSH PRIVILEGES;
EXIT;

```

## 6. Konfigurera Webbservern (Apache2)

Vi skapar en konfigurationsfil för Drupal med:

```bash
sudo nano /etc/apache2/sites-available/drupal.conf

```

Där fyller vi i följande information:

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/drupal

    <Directory /var/www/html/drupal>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/drupal_error.log
    CustomLog ${APACHE_LOG_DIR}/drupal_access.log combined
</VirtualHost>

```

Vi aktiverar sidan med Apache2 Enable Site & inaktiverar standard-välkomstsidan med:

```bash
sudo a2ensite drupal.conf
sudo a2dissite 000-default.conf

```

För att Drupal ska fungera med Apache2 krävs det att vi aktiverar två moduler, rewrite & php8.3:

```bash
sudo a2enmod rewrite php8.3
sudo systemctl restart apache2

```
**Det är nu möjligt att slutföra konfigurationen för Drupal genom en webbläsare.**


## 7. Härdning av systemet & minska attackyta
### 7.1 Säkra SSH

**CIS 5.1.20 - Ensure sshd PermitRootLogin is disabled**

```bash
sudo nano /etc/ssh/sshd_config

```

Se till att följande rad finns och är satt till `no`:

```text
PermitRootLogin no

```

**CIS 5.1.4 - Ensure sshd access is configured**

Skapa en grupp och lägg till din användare:

```bash
sudo groupadd ssh_users
sudo usermod -aG ssh_users [din användare]

```

Ändra `sshd_config` och lägg till `AllowGroups ssh_users`:

```bash
sudo nano /etc/ssh/sshd_config

```

*(Starta sedan om SSH-tjänsten med `sudo systemctl restart ssh`)*

### 7.2 Säkra /tmp

**CIS 1.1.2.x (1.1.2.1.1-1.1.2.1.4) - Configure /tmp** 

Mappen `/tmp` är ofta ett mål för angripare eftersom alla användare och processer som standard har skrivrättigheter där. Genom att montera `/tmp` som **tmpfs** (ett filsystem i RAM-minnet) kan vi begränsa attackytan med följande monteringsalternativ:

* **noexec** - inga körbara programfiler / binärer


* **nodev** - den kan inte innehålla speciella enhetsfiler


* **nosuid** - det går inte att köra program som filens ägare 



Vi kopierar `tmp.mount`, då den kan förändras av en OS-uppdatering, till vår mapp för lokala overrides `/etc/systemd/system/`:

```bash
sudo cp /usr/share/systemd/tmp.mount /etc/systemd/system/tmp.mount

```

Vi redigerar innehållet med:

```bash
sudo nano /etc/systemd/system/tmp.mount

```

Under sektionen `[Mount]` hittar vi raden “Options=” och lägger till `nosuid,nodev,noexec`. Spara och stäng filen.

Vi startar om systemd och aktiverar vår nya `tmp.mount`:

```bash
sudo systemctl daemon-reload 
sudo systemctl restart tmp.mount 
sudo systemctl enable tmp.mount

```

Vi kan enkelt kontrollera om förändringen har skett med:

```bash
mount | grep "/tmp"

```

### 7.3 Aktivera Audit Log

**CIS 6.2.1.x - Configure auditd Service** 

Genom att installera och aktivera auditd (och specificera vad som ska loggas) får vi en logg över systemkritiska funktioner:

```bash
sudo apt install auditd

```

Vi definierar reglerna i:

```bash
sudo nano /etc/audit/rules.d/audit.rules

```

Klistra in följande:

```text
## First rule - delete all
-D

## Increase the buffers to survive stress events.
## Make this bigger for busy systems
-b 8192

## This determine how long to wait in burst of events
--backlog_wait_time 60000

## Set failure mode to syslog
-f 1

# Monitor system call adjtimex (kernel clock) and settimeofday
-a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time-change
-a always,exit -F arch=b32 -S adjtimex -S settimeofday -S stime -k time-change

# Monitor clock_settimepr
-a always,exit -F arch=b64 -S clock_settime -k time-change
-a always,exit -F arch=b32 -S clock_settime -k time-change

# Monitor changes to the timezone file
-w /etc/localtime -p wa -k time-change

-w /etc/group -p wa -k identity
-w /etc/passwd -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/security/opasswd -p wa -k identity

-a always,exit -F arch=b64 -S sethostname -S setdomainname -k system-locale
-a always,exit -F arch=b32 -S sethostname -S setdomainname -k system-locale
-w /etc/issue -p wa -k system-locale
-w /etc/issue.net -p wa -k system-locale
-w /etc/hosts -p wa -k system-locale
-w /etc/network -p wa -k system-locale

# Monitoring the 'sudo' and 'su' commands specifically
-w /usr/bin/sudo -p x -k privileged
-w /usr/bin/su -p x -k privileged

# Monitoring any use of the mount command
-a always,exit -F arch=b64 -S mount -k mounts
-a always,exit -F arch=b32 -S mount -k mounts

```

Enligt vår `audit.rules` så loggar vi följande:

* Loggar förändringar av inställningar relaterade till tid; adjtimex, settimeofday etc. samt filen `/etc/localtime` 


* Loggar förändring av användare/grupp i `/etc/group`, `/etc/passwd`, `/etc/shadow` etc. 


* Loggar förändringar i `/etc/hosts` och `/etc/issue` - för kontroll av eventuella omdirigeringar


* Loggar användingen av “su” och “sudo” samt all använding av “mount” 



### 7.4 Begränsa Core Dumps / minnesutskrift
**CIS 1.5.3 - Ensure core dumps are restricted**

När ett program kraschar kan Linux skapa en fil med innehållet av arbetsminnet (en dump) vid exakt den tidpunkten. Detta kan även inkludera känslig information, så vi begränsar möjligheten att ta dessa "dumps".

Öppna filen:
```bash
sudo nano /etc/security/limits.conf

```  
 och lägg till  `* hard core 0` för att förhindra användare att ta en core dump.
```bash
sudo nano /etc/sysctl.d/99-sysctl.conf

```  
 lägg till raden `fs.suid_dumpable = 0` för att motverka att program som körs genom SUID kan skapa en dump.

### 7.5 Aktivera TCP SYN cookies
**CIS 3.3.10 - Ensure tcp syn cookies is enabled** 

En stor risk för just webbservrar är överbelastningsattacker (DoS/DDoS). En variant av detta är en så kallad "SYN-flood", där angripare skickar mängder med TCP SYN-paket för att överbelasta servern med oavslutade handskakningar. Genom att aktivera TCP SYN cookies motverkar man att servern reserverar minne för falska anslutningar.

Det gör vi genom att öppna:
```bash
sudo nano /etc/sysctl.d/99-sysctl.conf
``` 
Där lägger vi till raden `net.ipv4.tcp_syncookies = 1`

### 7.6 Inaktivera oanvända filsystem
**CIS 1.1.1.x - Ensure unused filesystems kernel modules are not available**

Föråldrade (legacy) filsystem som inte används av servern är en onödig attackyta. Vi kan "lura" systemet att peka dessa filsystem mot /bin/true, som alltid returnerar ett "lyckades"-svar; `0`.

Vi skapar filen:
```bash
sudo nano /etc/modprobe.d/disablefilesystems.conf

```
och lägger till:
```text
install cramfs /bin/true
install freevxfs /bin/true
install jffs2 /bin/true
install hfs /bin/true
install hfsplus /bin/true
install udf /bin/true
```

### 7.7 Härda rättigheter för mappar & filer i /var/www/html/drupal/

Sist men inte minst; för tillfället har vi inte begränsat rättigheterna för skriv/läs/exekvera för mapparna och filerna i vår webbserver.
Vi börjar med att byta ägare av `/var/www/html/drupal/` så att Apache2 får tillgång endast genom grupprättigheter:
```bash
sudo chown -R [din användare]:www-data /var/www/html/drupal/

```
Vi begränsar rättigheterna för mappar och filer med:
```bash
sudo find /var/www/html/drupal/ -type d -exec chmod 750 {} +;
sudo find /var/www/html/drupal/ -type f -exec chmod 640 {} +;

```
Den första raden begränsar mappar till 7 (rwx för ägaren) 5 (r-x för gruppen) och 0 för resterande.
Den andra raden begränsar filer till 6 (rw- för ägaren) 4 (r-- för gruppen) och 0 för resterande.

