---
title: freeing Elmo for Linux
date: 2017-01-12
layout: post
slug: freeing-elmo
published: true
tags:
- 'reverse engineering'
- 'usb'
- 'camera'
---
(Source: https://github.com/nv1t/freeElmo/)

Vor ein paar Wochen ist mir eine Document Camera in der Schule aufgefallen. Genauer gesagt handelt es sich im eine L-12 von Elmo. Zu diesem Zeitpunkt wussten wir bloss, dass es keine normale Webcam ist und eigene Software fuer Windows und MacOSX bereitgestellt wurde. Also musste es ein eigenes Protokoll sein. Ich moechte hier nicht so tief reingehen in die ganzen Ueberlegungen die wir hatten, sondern nur darstellen, wie das Teil aufgebaut ist und wie man ungefaehr damit kommuniziert.

Um zu den einzelnen Punkten zu gelangen hatten wir viele Fehler und lange Ueberlegungen, da wir den Elmo nur alle 2-3 Wochen fuer maximal 3-4h hatten. Wir haben also viel Blind geschrieben und dann erst getestet. Wiederliche Produktentwicklung...

Leider blieb Linux mal wieder aussen vor, was wir nicht auf uns sitzen lassen konnten.

Um an das Protokoll zu kommen, starteten wir erstmal den Originalen Windows Client und machten einen Mitschnitt der Kommuikation. Stellt sich raus, dass er ein paar Pakete schickt

```
 (endpoints: (IN 01:00:81, OUT 01:00:02))
OUT 0000 0000 1800 0000 108B 0000 0000 0000
OUT 0000 0000 0000 0000 0000 0000 0000 0000
IN  0100 0000 1800 0000 0000 0000 4C2D 3132  ............L-12
IN  0000 0000 0000 0000 0000 0000 0000 0000  ................
OUT 0000 0000 1800 0000 118B 0000 0000 0000
OUT 0000 0000 0000 0000 0000 0000 0000 0000
IN  0100 0000 1800 0000 0000 0000 5741 2E31  ............WA.1
IN  2E30 3034 0000 0000 0000 0000 0000 0000  .004............
OUT 0000 0000 1800 0000 118B 0000 0100 0000
OUT 0000 0000 0000 0000 0000 0000 0000 0000
IN  0100 0000 1800 0000 0000 0000 5741 2E31  ............WA.1
IN  2E31 3931 0000 0000 0000 0000 0000 0000  .191............
OUT 0000 0000 1800 0000 118B 0000 0200 0000
OUT 0000 0000 0000 0000 0000 0000 0000 0000
IN  0100 0000 1800 0000 0000 0000 5741 2E31  ............WA.1
IN  2E30 3630 0000 0000 0000 0000 0000 0000  .060............
OUT 0000 0000 1800 0000 108B 0000 0000 0000
OUT 0000 0000 0000 0000 0000 0000 0000 0000
IN  0100 0000 1800 0000 0000 0000 4C2D 3132  ............L-12
IN  0000 0000 0000 0000 0000 0000 0000 0000  ................

(endpoints: (IN 01:00:83, OUT 01:00:04))
OUT 0000 0000 1800 0000 8E80 0000 5000 0000
OUT 0000 0000 0000 0000 0000 0000 0000 0000
IN  0000 0000 1800 0000 XXXX 0000 5000 0000
IN  0000 0000 0000 0000 0000 0000 0000 0000
IN  0200 0000 F8FE 0000 PPPPPPPPPPPPPPPPPPP
IN  PPPP ....
IN  0200 0000 F8FE 0000 PPPPPPPPPPPPPPPPPPP
IN  PPPP ....
```
Er schickt immer 32 Bytes raus und kriegt 32 Bytes rein. Das Bild unterscheidet sich leicht, weil er nach der 32 Byte Antwort noch Pakete schickt die beliebig bzw. max. 65280 Bytes gross sind. Zudem wird das Bild auf anderen Endpoints abgehandelt.

Wir konnten also relativ schnell feststellen wie das Protokoll aufgebaut ist um Bilder zu bekommen. Wir schicken ein Datenpaket hin, kriegen eine Antwort (in der sogar die Groesse des Bildes kodiert ist) und lesen dann Pakete der Groesse 65280 Bytes. In Wirklichkeit sind es nicht immer 65280 Bytes. Das letzte Paket ist immer ein wenig kleiner, weil natuerlich die Groesse eines Bildes nicht immer ein vielfaches von 65280 ist.

Damit lesen wir das 4 und 5 Byte und rechnen es in die korrekte Groesse um, die wir vom USB-Endpoint lesen muessen.

Geliefert wird ein reiner Bytestream eines JPEG Bildes den wir theoretisch so in eine Datei pipen koennen.

Damit haetten wir eines der ersten Bilder:

![](/img/2013-06-28-101747_1024x768_scrot.png)

Am Anfang hatten wir noch leichte Pixelfehler, da wir vergessen haben die Header aus den Paketen rauszuparsen. Was uns natuerlich die Pixel auf interessante Art und Weise zerschossen hat.

Aber wir konnten endlich Bilder anzeigen!

Uns fehlten nach wir vor vitale Funktionen wie Zoom, Kontrast und Fokus gefehlt, die in der Originalsoftware auch vorhanden waren.

Also zurueck an den Windows Rechner und Mitschnitte machen. Wir konnten ja jetzt uns bekannte Pakete rausfiltern und nur noch die unbekannten anschauen.

Nach ein wenig geschneide und gerate hatten wir endlich eine "vollstaendige" (Ich glaube nicht an die Vollstaendigkeit) Liste der Funktionen:

```python
 self.msg = {
     'version':         [0,0,0,0,0x18,0,0,0,0x10,0x8B,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'picture':         [0,0,0,0,0x18,0,0,0,0x8e,0x80,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'buttons':         [0,0,0,0,0x18,0,0,0,0x00,0x0f,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'zoom_stop':       [0,0,0,0,0x18,0,0,0,0xE0,0,0,0,0x00,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'zoom_in':         [0,0,0,0,0x18,0,0,0,0xE0,0,0,0,0x01,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'zoom_out':        [0,0,0,0,0x18,0,0,0,0xE0,0,0,0,0x02,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'brightness_light':[0,0,0,0,0x18,0,0,0,0xE2,0,0,0,0x02,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'brightness_dark': [0,0,0,0,0x18,0,0,0,0xE2,0,0,0,0x03,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'brightness_stop': [0,0,0,0,0x18,0,0,0,0xE2,0,0,0,0x04,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'brightness_auto': [0,0,0,0,0x18,0,0,0,0xE2,0,0,0,0x05,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'focus_wide':      [0,0,0,0,0x18,0,0,0,0xEA,0,0,0,0x00,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'focus_near':      [0,0,0,0,0x18,0,0,0,0xEA,0,0,0,0x01,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'focus_stop':      [0,0,0,0,0x18,0,0,0,0xEA,0,0,0,0x02,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ,'focus_auto':      [0,0,0,0,0x18,0,0,0,0xE1,0,0,0,0x00,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]

}
```
Leider sind sie hier ein wenig inkonsistent. Es sieht sehr danach aus, dass sie auf dem 8. Byte eine Art Menu fuer Funktionen haben und dann auf dem 12. Byte ein Sub-Menu. Da bricht aber der Autofocus deutlich mit dem Autobrightness.

Zudem liegen die Stop Funktionen seltsam verteilt in den Untermenus.

Dazu waere noch zu sagen, dass die Stop Funktionen gebraucht werden. Man beginnt eine Aktion und diese wird solange ausgefuehrt, bis ein Stop Signal kommt. Im Prinzip kann man es sich vorstellen wie ein Motor den man anschaltet und wieder ausschalten muss.

Dies hat den Vorteil der stufenlosen Einstellung, aber man kann etwas nicht wieder auf die exakt gleiche Einstellung bringen. Man koennte es ueber einen Timer loesen, aber bisher muss man aus dem Frontend ein Start und Stop schicken :)

Wer Source lesen will kann sich freeElmo anschauen. elmo.py ist die zustaendige Klasse um den Elmo anzusteuern und `elmo-display.py` ist ein Viewer, der die Klasse einbindet. `elmo.py` ist nur abhaengig von pyusb, waehrend der Viewer derzeit in pygame geschrieben ist. Bild Manipulationen werden dort ueber PIL geloest. Es ist noch nicht unter Windows oder MacOSX getestet.

Die Linux User wollen die udev-Rules Datei sicher haben (diese gehoert nach `/etc/udev/rules.d/`) und aendert die Berechtigung des Devices auf die Gruppe video, damit man auf den Elmo ohne Root Rechte zugreifen kann

Der Source ist natuerlich frei zugaenglich.

Fals Leute Elmos besitzen waeren wir dankbar ueber Rueckmeldungen. Ich schreibe auch gerne kleine Tests um die Elmos auf Kompatibilitaet zu testen und gebe natuerlich Hilfestellung.

so long
