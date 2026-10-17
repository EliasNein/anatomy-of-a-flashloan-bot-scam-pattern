<div align="center">

<img src="scam-warning.png" alt="Bild im Stil eines Video-Thumbnails: &quot;How is AI doing this? +$2,629/day&quot;, mit großem rotem &quot;SCAM&quot;-Stempel" width="300">

# Anatomie eines mutmaßlichen 'Flash-Loan-Arbitrage-Bot'-Betrugsmusters

Code-Prüfung und Messung auf einem Mainnet-Fork zu einem "Arbitrage-Bot", der auf YouTube beworben wurde.

[English](README.md) · **Deutsch**

</div>

---

## Kurzfassung (TL;DR)

Dieses Repo dokumentiert die Analyse eines "Flash-Loan-Arbitrage-Bots", der in einem YouTube-Video beworben wurde. Der untersuchte Code stammt aus diesem Video, das uns persönlich bekannt ist, und wurde bewusst nicht in dieses Repo aufgenommen. Die Code-Datei enthält behauptete Gewinnbelege, die nicht echt sein können. Ihre eigenen Tests zeigen nur deshalb Gewinn, weil sie die Preislücke selbst erzeugen. Auf einem unveränderten Mainnet-Fork hat unsere Messung keinen organischen Gewinn gefunden.

> [!CAUTION]
> **Solchen Code nicht deployen. Kein ETH an einen Contract aus so einem Video schicken.**

## Im untersuchten Video beobachtetes Muster

Das Folgende beschreibt, was wir in dem einen untersuchten Video beobachtet haben. Es ist keine allgemeine Aussage über alle solchen Videos.

- Das Video verspricht passives Einkommen durch einen "Arbitrage-Bot".
- Zuschauer sollen einen Contract in eine Browser-IDE kopieren, selbst deployen und 1–2 ETH einzahlen.
- Außerdem sollen sie einen "Verification-Link" anklicken.
- In angepinnten Kommentaren stehen Kurzlinks. Die Kommentarspalte zeigt gleichförmig positive, allgemein gehaltene Kommentare.

Ein Flash Loan wird innerhalb einer einzigen Transaktion geliehen und zurückgezahlt. Er braucht außer Gas kein eigenes Kapital. Die Aufforderung, ETH in einen Bot einzuzahlen, der angeblich mit Flash Loans arbeitet, ist deshalb das zentrale Warnsignal dieses Musters.

## Der analysierte Code

Der analysierte Code stammt aus einem konkreten YouTube-Video, das uns persönlich bekannt ist. Wir haben ihn bewusst nicht in dieses Repo hochgeladen, damit es nicht zu einer weiteren Quelle wird, aus der man ihn kopieren und deployen kann.

Die Originaldatei verlangt in ihrem Kopfkommentar "1–2 ETH oder mehr". Der Contract selbst lehnt einfache ETH-Überweisungen aber ab (receive/fallback). Ein Off-Chain-Keeper-Skript in der Originaldatei verweigert den Live-Modus, solange sein eigenes Wallet weniger als 2 ETH hält. Mit den Standardeinstellungen handelt der Keeper nur über Flashbots-Bundles, weil das Senden an den öffentlichen Mempool abgeschaltet ist (Zeile 1754). Ein Bundle zahlt nur dann Gas, wenn es in einen Block aufgenommen wird. 2 ETH sind deshalb weit mehr, als der Bot braucht.

Eine Falle im Contract haben wir nicht gefunden. Unsere Prüfung war rein statisch (siehe unten).

## Red-Flag-Checkliste

- Garantiertes oder "stabiles tägliches" Einkommen durch einen Trading-Bot.
- Du sollst kopierten Code selbst deployen und aufladen.
- "Verification-Links", Kurzlinks oder Links in angepinnten Kommentaren.
- Viele fast identische Kommentare, die Gewinne loben.
- Eine einzige riesige Datei, die Code, "Audit"-Behauptungen, "Testergebnisse" und "DO NOT EDIT"-Banner bündelt.
- Behauptete Transaktionen und Statistiken, die beim Nachrechnen nicht stimmen.
- Gewinn gibt es nur in Tests, die vorher die Preise manipulieren.

## Befunde aus der Code-Prüfung

Die Prüfung war statisch: Wir haben den Code gelesen und durchsucht. Ausgeführt haben wir nur die unten beschriebenen Foundry-Tests.

Die Zeilennummern in diesem Abschnitt beziehen sich auf die Originaldatei, die nicht veröffentlicht ist. Diese Datei bündelte Contract, Tests, einen Off-Chain-Keeper und ein Simulationsskript, die beiden letzten als auskommentiertes JavaScript. Keeper und Simulation wurden nie ausgeführt.

**Behauptete Belege, die nicht echt sein können.** Die "echten Mainnet-Tests" und Statistiken im Kopf der Datei halten einer Prüfung nicht stand:

- Zwei verschiedene Trades haben denselben Transaktions-Hash (Zeilen 395 und 498).
- Mehrere Trade-Rechnungen ergeben einen Verlust, werden aber als Gewinn ausgewiesen. Ein Trade macht aus 100.000 USDC 100.412 USDC, nennt 1.040 USDC DEX-Gebühren und meldet trotzdem +255 USDC (Zeilen 381–393). Ein anderer macht aus 50.000 DAI 50.198 USDC, nennt 300 USDC Gebühren und meldet +133,50 USDC (Zeilen 339–351).
- Dieser DAI-Trade beginnt mit DAI und endet mit USDC. Der Contract lehnt jede Route ab, die nicht mit demselben Asset beginnt und endet (`RouteMustStartAndEndOnAsset`). Der Trade kann also nicht gelaufen sein.
- Einige Swaps sind unmöglich. Der Curve-3pool soll WETH liefern (Zeile 320), enthält aber kein WETH. Es werden Routen über Curve und Balancer aufgeführt, doch der Contract kann nur Router im Stil von Uniswap V2 und V3 aufrufen.
- Eine "30-Tage"-Statistik (Zeile 501) enthält eine Gewinnserie von "34 Tagen" (Zeile 514).
- Der "Mainnet Run" (Zeile 169) ist der Mock-Modus der Simulation. Er nutzt fest eingetragene, synthetische Pool-Reserven (Zeile 3769). Seine Zahlen widersprechen sich außerdem. Ein Flash Loan über 1.000.000 USDC bringt netto +2.572,64 USDC (Zeile 183), der Execution Guard meldet aber einen erwarteten Nettogewinn von +12.572,64 USDC (Zeile 192). Ein Kredit über 250.000 USDC soll netto +3.184,27 USDC bringen (Zeile 185), also mehr als der über 1.000.000 USDC.
- Der Kopf wirbt mit einem "12-hop"-Spread (Zeile 271), der Keeper sucht aber höchstens 5 Hops (Zeile 1676). Er wirbt mit 1inch-Unterstützung (Zeile 9), der Code implementiert 1inch aber nicht. Die Schlusszeilen versprechen "Guaranteed profit" und "Stable daily income" (Zeilen 1582–1583).

**Die Datei widerspricht sich selbst.** Ihr eigenes Simulationsskript stellt fest, dass auf einem gesunden Markt organische Spreads zwischen DEXs, die Gebühren und Prämie übersteigen, innerhalb eines Blocks von konkurrierenden Bots geschlossen werden (Zeilen 4337–4338).

**Irreführende Sicherheitsfunktionen im Contract.**

- Der Kopf wirbt mit einer "Emergency profit withdrawal with 24h timelock" (Zeile 630). Die Sperre gilt nur in den ersten 24 Stunden nach dem Deployment (Zeile 920) und danach nie wieder. `sweepToken` (Zeile 999) kann jeden Token abziehen, solange der Contract pausiert ist, ganz ohne Sperre.
- Der Contract speichert eine Einstellung `slippageBps` (Zeile 870), liest sie aber nie. Der Slippage-Schutz hängt komplett an den Mindestbeträgen, die der Keeper übergibt.

**Riskante Voreinstellungen im Keeper** (nur durch Lesen, der Keeper wurde nie ausgeführt).

- `DRY_RUN` steht standardmäßig auf false, `ENABLE_LIVE` auf true (Zeilen 1739, 1761). Nach 24 Stunden Laufzeit und genug bestandenen Simulationen handelt der Bot ohne weitere Bestätigung live. `LIVE_FORCE` überspringt diese Schranke (Zeile 1976). Die Schranke wertet nur Simulationen aus, keine realisierten Ergebnisse.
- `MAX_FLASH_AMOUNT` steht standardmäßig auf 0, das heißt unbegrenzt (Zeile 1735). Es gibt kein Tagesbudget für Gas oder Verluste.
- Der Private Key wird im Klartext aus der Umgebungsvariable `KEEPER_PK` gelesen (Zeile 1716). Derselbe Key signiert Trades und Flashbots-Anfragen.
- Ohne konfigurierten Startblock durchsucht die Pool-Suche nur die letzten 5.000 Blöcke, etwa 17 Stunden (Zeilen 2114, 2134). Etablierte, liquide Pools fehlen dann. Gefunden werden vor allem neu angelegte Pools und Tokens, eine Gruppe, in der Betrugs-Tokens häufig sind.

**Kein Drainer gefunden, mit Einschränkungen.** Im Contract und in den Off-Chain-Skripten haben wir keine fest eingetragenen Empfängeradressen gefunden, kein `selfdestruct` oder `delegatecall` und keinen Code, der einen Private Key irgendwohin sendet. Der Flash-Loan-Callback prüft Aufrufer und Initiator. Außerdem haben wir keine versteckten Unicode-Steuerzeichen ("Trojan Source") gefunden, kein `eval`, keinen verschleierten oder base64-kodierten Code und keine Installationsskripte.

Diese Ergebnisse beruhen nur auf Lesen. Die 16 Contract-Adressen in der Originaldatei wurden nicht gegen Etherscan geprüft.

**Nicht bestätigt.** Bei der Prüfung sind drei vermutete Funktionsfehler aufgefallen, die wir nicht getestet haben:

- der Uniswap-V3-Swap-Aufruf im Contract passt nicht zum voreingestellten V3-Router;
- die V3-Quoter-Schnittstelle im Keeper ist falsch;
- die Signatur der Flashbots-Anfragen ist fehlerhaft.

Das sind Hypothesen.

## Unser Experiment

Aufbau: Foundry-Tests auf einem Mainnet-Fork bei Block 25733035 (Details unter "So haben wir gemessen").

**Die Gewinn-Tests erzeugen ihre Gelegenheit selbst.** Die Tests PROOF1 bis PROOF3 lassen zuerst einen simulierten "Wal" (finanziert per Foundry-`vm.deal`) WETH auf Uniswap V2 verkaufen. Danach handelt der Bot gegen die Lücke, die dadurch entsteht. Die geloggten Gewinne sind die Änderung des USDC-Guthabens im Contract, gemessen nach Rückzahlung der Aave-Prämie und vor Gas:

- 3.530,15 USDC: 20.000 USDC Flash Loan, der Wal verkauft 1.000 WETH.
- 41,02 USDC: 2.000 USDC Flash Loan, der Wal verkauft 100 WETH.
- 26.848,25 USDC: 50.000 USDC Flash Loan, der Wal verkauft 5.000 WETH.

**Die "honest organic baseline" prüft nichts.** Test T4 fragt einen Rundlauf ab, enthält aber kein Assert. Eine seiner Variablen wird nie verwendet.

**Unsere eigene Messung (T4b).** Unser Test `test_T4b_organicRoundTrip` setzt keinen Wal ein und führt nichts aus. Der Test fragt nur die Router nach Quotes, deshalb fehlen Aave-Gebühr und Gas. Ein Rundlauf USDC → WETH → USDC verliert bei jedem getesteten Einsatz (auf zwei Nachkommastellen gerundet):

| Einsatz (USDC) | Uniswap → Sushi | Sushi → Uniswap |
|---:|---:|---:|
| 100 | −0,49 % | −0,88 % |
| 1.000 | −1,23 % | −1,61 % |
| 5.000 | −4,38 % | −4,75 % |
| 20.000 | −14,60 % | −14,94 % |

> [!NOTE]
> **Grenzen der Messung.** Sie deckt einen Block und ein Paar ab (USDC/WETH auf Uniswap V2 und SushiSwap). Sie nutzt nur Quotes, ohne Gas und ohne konkurrierende Bots. Sie ist keine Aussage über alle Märkte. Sie zeigt aber: In diesem Aufbau gab es den beworbenen Gewinn ohne künstliche Preisbewegung nicht.

## So haben wir gemessen

- Werkzeug: Foundry 1.8.5.
- Umgebung: ein lokaler Mainnet-Fork, festgelegt auf Block 25733035. Es wurden keine Transaktionen an das echte Netzwerk gesendet.
- Methode für T4b: Quotes über `getAmountsOut` der Router für den Rundlauf USDC → WETH → USDC, in beiden Router-Reihenfolgen (Uniswap V2 und SushiSwap), für Einsätze von 100, 1.000, 5.000 und 20.000 USDC. Das Ergebnis ist die Differenz zwischen ausgezahlten und eingesetzten USDC, bezogen auf den Einsatz.
- Nicht enthalten: Aave-Flash-Loan-Gebühr und Gas.

Der analysierte Code wird hier nicht veröffentlicht.

## Wenn du schon etwas deployt oder eingezahlt hast

- Token-Freigaben deines Wallets prüfen und widerrufen, zum Beispiel mit revoke.cash. Tippe die Adresse selbst in den Browser ein. Nutze nie einen Link aus einem Video oder seinen Kommentaren.
- Verbleibendes Guthaben auf ein neues Wallet verschieben. Wenn du jemals eine Seed Phrase oder einen Private Key auf einer Seite eingegeben hast, die aus so einem Video verlinkt war, behandle das alte Wallet als kompromittiert.
- Das Video bei YouTube melden.

## Über dieses Repo

Dieses Repo enthält nur diese Dokumentation, auf Englisch und Deutsch. Der analysierte Code ist bewusst nicht enthalten.

## Haftungshinweis

Dieses Repo dient ausschließlich Bildungs- und Forschungszwecken. Es wird ohne Mängelgewähr ("as is") und ohne jede Gewährleistung bereitgestellt. Es ist keine Finanzberatung. Es beschreibt ein einzelnes Code-Beispiel und ein mutmaßliches Betrugsmuster, das in einem Video beobachtet wurde. Es trifft keine Aussage über einzelne Personen.
