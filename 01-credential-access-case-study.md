# Active Directory Credential Access & Wazuh Detekce

Praktické shrnutí simulace a detekce dvou útoků z kategorie **Credential Access** (MITRE ATT&CK) v doméně `lab.local`.

---

## Přehled prostředí

* **Doménový řadič (DC01):** `192.168.10.2` (Windows Server)
* **Wazuh Manažer:** `192.168.10.10` (Centrální SIEM)
* **Útočník (Kali Linux):** `192.168.10.151` (Unmanaged stroj v síti)

---

## 1. SMB Password Spraying (MITRE T1110.003)

* **Princip útoku:** Testování jednoho běžného hesla proti mnoha uživatelům naráz. Cílem je najít slabé heslo a vyhnout se uzamčení účtu.
* **Simulace (Kali):**
  ```bash
  nxc smb 192.168.10.2 -u users.txt -p 'Heslo123!'
  ```
* **Telemetrie ve Windows (DC01):**
  * **Event ID 4625** (*Logon Failure*).
  * **Logon Type 3:** Síťové přihlášení (SMB/NTLM).
  * **SubStatus `0xC0000064`:** Kód `STATUS_NO_SUCH_USER` potvrzuje hádání neexistujících jmen.
  * **Workstation Name `-`:** Chybějící název stanice značí automatizovaný ne-Windows skript.
* **Detekce (Wazuh Rule `100100`):**
  * Sleduje korelaci neúspěšných přihlášení (rodič `60122`).
  * Podmínka: Stejná IP (`win.eventdata.ipAddress`) a různí uživatelé (`win.eventdata.targetUserName`) v okně 60 sekund.
  * Nastaveno `frequency="3"` pro spuštění alertu při 4. pokusu.
   
   -Detekční pravidla jsou uložena samostatně ve složce `detections/local_rules.xml`
* **L1 Analýza & Triage:**
  * Kontrola logů na **Event ID 4624** (úspěšné přihlášení) ze zdrojové IP. Žádná shoda nenalezena -> útok byl neúspěšný.

---

## 2. Kerberos AS-REP Roasting (MITRE T1558.004)

* **Princip útoku:** Účet má v Active Directory zapnutý příznak `DONT_REQ_PREAUTH`. Řadič (KDC) vydá šifrovaný TGT lístek komukoliv bez předchozího ověření hesla. Útočník lístek zachytí a heslo láme offline, takže řadič o hádání hesel neví a účet nezamkne.
* **Prerekvizita auditu na DC01:** Windows úspěšné žádosti o Kerberos lístky ve výchozím stavu neloguje. Bylo nutné zapnout auditování úspěchů:
  ```cmd
  auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable
  ```
* **Simulace a prolomení (Kali):**
  1. Získání hashe bez znalosti hesla:
     ```bash
     impacket-GetNPUsers lab.local/textern -dc-ip 192.168.10.2 -no-pass
     ```
     Řadič vrátil hash `$krb5asrep$23$...`.
  2. Offline cracking přes John the Ripper: Slovník `rockyou.txt` selhal kvůli vyšší komplexitě AD hesla, malý cílený slovník hash prolomil okamžitě.
* **Telemetrie ve Windows (DC01):**
  * **Event ID 4768** (*Kerberos TGT Request*).
  * **`PreAuthType: 0`:** Potvrzení absence pre-autentizace.
  * **`TicketEncryptionType: 0x17`:** Použití slabé šifry RC4-HMAC.
* **Detekce (Wazuh Rule `100200`):**
  * Navázáno na kořenový Security log (`<if_sid>60103</if_sid>`).
  * Filtruje Event ID `4768`, hodnotu `preAuthType: ^0$` a vylučuje legitimní účty počítačů končící na `$` (`targetUserName: ^.*\$`).

  -Detekční pravidla jsou uložena samostatně ve složce `detections/local_rules.xml`
* **L1 Analýza & Triage:**
  * Pravidlo **Assume Breach**: Alert na `PreAuthType: 0` znamená okamžitou kompromitaci.
  * Okamžitá náprava: Zrušit `DONT_REQ_PREAUTH` v AD, vynutit reset hesla účtu a prověřit následný pohyb v síti (Event ID 4624/4769).

---

## Hlavní poznatky pro praxi

1. **Auditní slepota:** SIEM nedetekuje útoky na Kerberos, pokud v OS není aktivní auditování úspěšných operací (`Success`).
2. **Korelační logika:** U moderních Windows logů je nutné používat přesná pole (`same_field`, `different_field`) namísto unixových syslog značek.
3. **Pravidlo reakce:** Neúspěšné SMB přihlášení vyžaduje ověření průniku (hledání 4624), zatímco úspěšný AS-REP požadavek vyžaduje okamžitý zásah a reset účtu.

