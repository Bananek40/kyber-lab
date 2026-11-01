# LLMNR Poisoning a Lateral Movement

Tento zápis dokumentuje průběh simulovaného útoku v laboratoři. Zahrnuje získání administrátorských oprávnění, pokus o skrytý posun v síti, blokaci ze strany Windows Defenderu, zneužití lidské chyby (omylem povolená výjimka) a finální demonstraci toho, jak EDR úspěšně zabránilo dalším úpravám systému.

---

## 1. Získání hesla (LLMNR Poisoning)

* **Nastražená past:** V Kali Linuxu jsem spustil nástroj `Responder` naslouchající síťovým broadcastům (`sudo responder -I eth0 -dwv`).
* **Spuštění události:** Na doménovém řadiči (DC01) jsem pod účtem administrátora do průzkumníka zadal neexistující síťovou cestu.
* **Zisk:** Windows se pokusil ověřit vůči falešnému serveru Responderu, čímž odevzdal NTLMv2 hash účtu `Administrator`. Hash byl obratem prolomen pomocí nástroje John the Ripper (`john --format=netntlmv2`), čímž jsem získal heslo v čistém textu.

---

## 2. Pokus o tichý Lateral Movement (WMI)

Pokusil jsem se využít získané heslo k neviditelnému vzdálenému spouštění příkazů.

* **Útok:** Z Kali Linuxu jsem spustil nástroj `wmiexec` ze sady Impacket:
  ```bash
  impacket-wmiexec Administrator:'Heslo123'@192.168.10.2
  ```
* **Prvotní blokace EDR:** Terminál v Kali Linuxu zamrzl s hláškou `[*] SMBv3.0 dialect used`. Nástroj `wmiexec` k výpisu příkazů vytváří skryté textové soubory ve složce `ADMIN$` a přesměrovává do nich výstup. Windows Defender toto okamžitě detekoval, zablokoval proces a zařadil hrozbu jako `VirTool:Win32/Impacket.D`.

---

## 3. Lidský faktor a průnik do systému

* Při zkoumání vyskočeného alertu přímo na serveru DC01 jsem v rozhraní Windows Defenderu omylem kliknul na tlačítko **Allow (Povolit)** u zablokované hrozby. 
* Tím se pro chování Impacketu vytvořila v systému výjimka. 
* V Kali Linuxu jsem zrušil zaseknutý příkaz, spustil ho znovu a tentokrát okamžitě získal shell s právy administrátora.

---

## 4. Snaha o zahlazení stop

Pokusil jsem se z příkazové řádky z Kali smazat omylem vytvořenou výjimku v Defenderu pomocí vzdáleného PowerShellu:
```cmd
powershell -Command "Get-MpPreference | Select-Object -ExpandProperty ThreatIDDefaultAction_Ids | ForEach-Object { Remove-MpPreference -ThreatIDDefaultAction_Ids $_ }"
```
* **Obrana systému:** Příkaz selhal s chybou `0x57 (CimException)`. Zafungovala funkce **Tamper Protection**.

---

## 5. Pokus o povolení RDP a finální blokace EDR

Jelikož nešlo Defender vyřadit tiše, pokusil jsem se přes `wmiexec` shell upravit registry a povolit Vzdálenou plochu (RDP), abych mohl Defender vypnout graficky:
```cmd
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
```
* **Obrana systému:** Tento příkaz Windows Defender opět zablokoval s kritickým alertem. Defender znovu rozpoznal wrapper Impacketu (přesměrování `1> \\127.0.0.1\ADMIN$\...`).  

---  

## 6. Stopy v SIEMu (Wazuh Telemetrie) 
 Tento incident vygeneroval ucelený vzorec alertů, které jasně ukazují fáze útoku od kompromitace po skrytý posun:  
 * **Rule 92213 (Level 15 - Executable file dropped):** Kritická detekce zachycující počáteční i finální zásah Windows Defenderu proti Impacketu a pokusy o zápis payloadu do složky `ADMIN$`.
* **Rule 61640 (Level 12 - Sysmon Suspicious Process creation):** Anomálie v hierarchii procesů. Sysmon zachytil, že legitimní služba `WmiPrvSE.exe` začala nečekaně plodit instance `cmd.exe` a `reg.exe`.
* **Rule 92058 (Level 12 - Application Compatibility):** Detekce neobvyklého spouštění procesů v rámci interních Windows mechanismů (Amcache/Shimcache).
* **Rule 92205 (Level 9 - Powershell process creation):** Zachycení neúspěšného pokusu o modifikaci Defenderu přes Powershell (Tamper Protection blokace).
* **Rule 92652 (Level 6 - Successful Remote Logon):** Záplava nízkoúrovňových událostí zachycující neustálé navazování relací přes SMB (Logon Type 3).

## 7. Doporučení pro nápravu

Pro zamezení tohoto útoku je nutné zavést následující systémové změny:

* **Deaktivace LLMNR/NBT-NS (Prevence kompromitace hesla):** Fallback protokoly pro lokální překlad názvů představují kritické riziko. **LLMNR** je nutné plošně zakázat pomocí doménové politiky a zastaralý **NBT-NS** plošně deaktivovat.
* **Centrální správa EDR (Prevence lidské chyby):** Oprávnění k manipulaci s hrozbami nesmí být lokálně dostupné. Správa Windows Defenderu, nastavování výjimek a vynucení Tamper Protection musí být striktně přesunuto do rukou centrální správy kde změny budou podléhat schvalovacím procesům, čímž se eliminuje riziko chybného lokálního rozhodnutí.

**Závěr:** Scénář demonstruje, že ani špičková detekce nedokáže zabránit průniku, pokud selže lidský faktor. Zároveň ale ukazuje robustnost moderní ochrany, která útočníka nakonec uzamkla a donutila k vygenerování velkého množství kritických stop ve Wazuhu, což zajišťuje včasnou detekci bezpečnostním dohledem.