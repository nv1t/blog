---
title: 'CSRF in Http Basierten Binaer Protokollen'
date: 2023-12-07
categories:
- tranquilpeak
- features
tags:
- html elements
- markdown
---
# Einleitung: Was ist eine CSRF?
HTTP ist im Grunde ein stateless Protokoll. Man stellt eine Anfrage an einen Server und bekommt darauf die Antwort. Der Server schliesst nun die Verbindung und das Spiel beginnt von neuem. Damit der Server weiss wer man ist, muss zwingend eine Art Token mitgeschickt werden. Dies geschieht traditionell mit Cookies. Mit Cookies kann der Server auf dem Client Informationen speichern. Ein Cookie kann fuer einen bestimmten Pfad fuer einen gewissen Zeitraum vom Server beim Client gesetzt werden. Diese Informationen werden auch bei jeder Anfrage automatisch mitgeschickt. Da diese Cookies einen State in HTTP implementieren werden sie auch verwendet um Authentifizierungen zu Regeln. Meist wird nach einem Login ein Session Cookie generiert mit einem zufaelligen Token um dem Client die Moeglichkeit zu bieten ohne weiteren Login die Seite zu verwenden.

Man kann also davon ausgehen, dass, wenn eine Seite aufgerufen wird und ein gueltiges Session Cookie existiert der Benutzer automatisch eingeloggt ist.
Eine Cross-Site-Request-Forgery Attacke beruht also darauf ein Opfer eine vom Angreifer praepierte Seite zu locken, von der ein Request zu der betreffenden Seite gestartet wird, welcher, da der Benutzer durch das Session Cookie eingeloggt ist, in seinem Kontext ausgefuehrt wird.

Betrachten wir ein einfaches Beispiel: 
Eine Bank fuehrt ihre Ueberweisungen mit einem einfachen Aufruf aus der wie folgt aussieht:

```language-html
<form action="transferMoney.php" method="GET">
    <input type="text" name="ktnr" value="">
    <input type="text" name="amount" value="">
    <input type="submit" name="submit" value="Let it rain money">
</form>
```

Wenn man sich den Aufruf naeher Betrachtet wird dieser Request gemacht:

```
GET /transferMoney.php?ktnr=123456&amount=100 HTTP/1.1
Host: bank.me
Cookie: SessionID=<totallyRandomSessionID>
```

Der gleiche Request laesst sich durch den Aufruf der URL: "http://bank.me/transferMoney.php?ktnr=123456&amount=100" erzielen. Dies hat zur Folge, dass der Angreifer nur diese URL im Browser des Opfers aufrufen muss um Geld auf das Konto 123456 zu transferieren.
Am einfachsten eine URL aufzurufen ist ein iframe.
Da der Session Cookie jedes mal mitgeschickt wird, wenn das Opfer die Domain bank.me aufruft, praepiere ich eine Seite mit folgendem Code:

```language-html
<iframe src="http://bank.me/transferMoney.php?ktnr=evilKtnr&blz=77050000&amount=100000"></iframe>
```

Bei ansurfen der Seite wird die Seite bank.me aufgerufen und Geldtransfer von 100.000 Euro auf das Konto "evilKtnr" veranlasst.

Man koennte jetzt natuerlich die Parameter per POST uebertragen und nicht an die URL anhaengen. Gehen wir davon aus und die Bank veranlasst dieses um ihre Applikation weiter abzusichern. Der Aufruf reicht also nichtmehr da der Aufruf nun so aussieht:

```
POST /transferMoney.php HTTP/1.1
Host: bank.me
Cookie: SessionID=<totallyRandomSessionID>
Content-Length:22

ktnr=123456&amount=100
```

Die einfachste Moeglichkeit einen POST Request an dieses PHP Skript "transferMoney.php" zu schicken ist auch ein Formular zu bauen oder das der Bank mit einer kleinen Aenderung zu Uebernehmen.

```language-html
<form action="http://bank.me/transferMoney.php" method="POST">
    <input type="text" name="ktnr" value="evilKtnr">
    <input type="text" name="amount" value="100000">
</form>
<script>
document.getElementByTagName("form")[0].submit();
</script>
```

Das beigefuegte PHP Script bewirkt, dass das Formular ohne Interaktion des Opfers abgeschickt wird.
So kann nun alles uebertragen werden.

# Fuck You! Ich benutze JSON!
Wir ueberlegen uns was passieren wuerde, wenn wir ein Formular mit einer Textarea mit dem Namen "x" bauen einem JSON als Value.
Nach dem oben genannten Beispiel sollte klar sein, dass der Request aussieht wie Folgt:

```
POST /somewhere HTTP/1.1
Cookie: SessionID=<totallyRandomSessionID>
Content-Length:

x={'kant':'x**2+y**2=z**2'}
```

Wenn man versucht `"x={'kant':'x**2+y**2=z**2'}"` durch JSON parsen zu lassen wird es einen Fehler produzieren.

## Die Formular Variante
Betrachten wir aber, was da eigentlich geschickt wird, faellt uns auf, dass es sich um Strings handeln koennte.

An sich ist der Name "x" auch nur Ballast, aber selbst, wenn wir es leer setzen kriegen wir das "=" nicht weg. Wenn wir weiterhin Formulare verwenden wollen ist es wohl die einfachste Moeglichkeit das = in das JSON einzugliedern. Im oben genannten Beispiel saehe das Formularfeld dazu so aus:

```language-html
<input type="{'kant':'x**2+y**2" value="z**2'}">
```

Damit wuerde das = in das JSON eingebunden werden und ein korrektes JSON produzieren.

## XHR Requests
Um das ganze zu umgehen, und da wir eh JavaScript einsetzen muessen um das Formular ohne Interaktion abzusenden, kann, durch die Moeglichkeit der neuere Browser, auch direkt auf reines JavaScript mit XHR Requests zurueckgegriffen werden.
XHR bezeichnet das im Umgangssprachlich besser bekannten AJAX Requests.

Durch einen einfachen Request (Hier mit Jquery vereinfacht), kann sehr einfach ein JSON-Request gemacht werden.

```language-javascript
$.ajax({
   url: '%your_service_url%',
   type: 'POST',
   contentType: 'text/json',  
   data: {"a": true,"b": "some string"},
   processData: false
});
```

# Man kann nicht wirklich Binaer schicken?
Das ist das schoene an XHR Requests. Die einfache Antwort ist: Doch!
Man kann JavaScript XHR Requests Typed Arrays uebergeben, was nicht nur die JSON Angriffsvariante deutlich erleichtert, sondern es auch Moeglich macht, Binaerdaten in einem HTTP Request unterzubringen.

```language-javascript
var bytesToSend = [253, 0, 128, 1],
    bytesToSendCount = bytesToSend.length;

var bytesArray = new Uint8Array(bytesToSendCount);
for (var i = 0, l = bytesToSendCount; i < l; i++) {
  bytesArray[i] = bytesToSend[i];
}

$.ajax({
   url: '%your_service_url%',
   type: 'POST',
   contentType: 'application/octet-stream',  
   data: bytesArray,
   processData: false
});
```

Dabei werden die einzelnen Bytes in ein Typed Array gebracht, welches man mit einem normalen XHR Request (Hier mit JQuery vereinfacht), abgeschickt wird.
Der Content-Type kann dabei aber 3 Werte annehmen, ohne, dass ein Pre-Flight geschickt wird. Auch koennen die Header nicht bearbeitet werden, was einen Eingriff Massiv einschraenkt.
Trotzdem sollte hier beachtet werden, dass dieser Pre-Flight spezifiziert ist, aber in einzelnen Faellen von Plugins umgangen werden kann.
So gab es eine Luecke in Flash, bei der man eine Anfrage an einen dritten Server (Server eines Angreifers) stellte. Diese Seite gab den Pre-Flight, der alles erlaubte und leitet mit einem 307er weiter auf die eigentlich anzugreifende Seite. Die Folge war, dass alle Header mitgeschickt wurden und kein neuer Pre-Flight gesendet wurde.

Dies macht es nun moeglich CSRF Angriffe auf alle Protokoll-Arten durchzufuehren, die auf HTTP basieren.

# Einschraenkungen
Ich erwaehnte bereits Einschraenkungen, mit denen in neueren Browsern zu Rechnen ist.

1. Vorhandenes Session Cookie im Browser: Die erste grosse Einschraenkung waere wohl das zur Verfuegung stehende Session Cookie. Eine CSRF Attacke findet standardgemaess im Browser statt. Dementsprechend muss auch hier das Session Cookie vorhanden sein. Bei einer reinen Desktop Applikation ist dies eher unwahrscheinlich.
* Veraenderung der Header: eine Veraenderung der Header hat in aktuellen Browser einen Pre-Flight zur Folge. Dies fuehrt zu keiner korrekten Ausfuehrung der CSRF Attacke.
* Verwendung eines vernuenftigen CSRF Token: Well...dann ist alles richtig implementiert und man kann einpacken mit CSRF.

# Fazit
Es ist also nicht leicht diese Art des Angriffes auszufuehren, weil es doch recht viele Einschraenkungen gibt. Sollte man aber alle Bedingungen erfuellen, kann der Angriff sehr effektiv sein.
Es sollte also bei jedem Request, der irgendwie an eine API geht, ein CSRF-Token mitgeschickt werden, damit so ein Angriff erkannt werden kann.

so long

