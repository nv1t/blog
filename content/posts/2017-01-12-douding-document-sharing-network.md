---
title: Douding Document Sharing Network
date: 2017-01-12
layout: post
categories:
tags:
- 'reverse engineering'
slug: douding-document-sharing-network
---
*Ich werde keinen Downloader anbieten, weil mir das Coden dieses zu Muehseelig ist. Das hier ist ein Proof of Concept mit einzelnen Scripten, die fuer sich zum jetzigen Zeitpunkt funktionieren.*

Vor einiger Zeit hab ich mit dem Online Archiv Douding (http://docin.com) aus China beschaeftigt. Hierbei handelt es sich um eine Moeglichkeit Dokumente online auszutauschen. Man kann sich jedes Dokument online anschauen, aber muss sich einloggen und selber sharen, oder bezahlen um andere Dokumente herunterzuladen.

Gluecklicherweise bietet der Dienst eine Online Vorschau im eigenen Viewer. Wenn man sich das frei anschauen kann, wird es wohl auch eine Moeglichkeit geben dies auch Offline zu betrachten.

# Download der Ressource
Ein seltsames in Flash geschriebenes Teil. Mein ActionScript ist seit Flash5 ziemlich eingerostet und es hat sich sicher seitdem viel getan, aber es geht ja um das Prinzip. An sich wollen wir ja aber bloss rausfinden, wo er seine Ressourcen herkriegt und wie diese Gespeichert sind.

Nach ein wenig suchen stoesst man auf DocinLoader{1,3}.as. Hier ist aber nur die 3. spannend. Er liest von einem urlStream ein. Was er genau da einliest schauen wir uns gleich an.

Zuerst muessen wir schauen, wo er diese ueberhaupt herkriegt.

Dafuer ist die Funktion parseRequestUrl() spannend.

```actionscript
 private function parseRequestUrl():String{
    var _local1 = "";
    if (this.currentPage == 1){
        _local1 = (((VoniboManager.DocinLoadUrl + "/docin_") + VoniboManager.productId) + ".docin");
    } else {
        if ((((this.currentPage >= 20)) && ((VoniboManager.loadUrlCtrl == 1)))){
            _local1 = (((("http://221.122.117.125/docin_" + VoniboManager.productId) + "_") + this.currentPage) + ".docin");
        } else {
            _local1 = (((((VoniboManager.DocinLoadUrl + "/docin_") + VoniboManager.productId) + "_") + this.currentPage) + ".docin");
        };
    };
    trace(_local1);
    return (_local1);
}
```
Laesst sich rauslesen, dass die Ressourcen auf jeden Fall unter der DocinLoadUrl liegt, welche 221.122.117.125 ist. Hier ist mir eigentlich alles bekannt und so bekomme ich eine seltsame Binaerdatei raus.

**Beispiel:** `http://www.docin.com/p-134681478.html liegt` unter `http://221.122.117.125/docin_134681478.docin`

Wenn die Dokumente groesser sind als 20 Seiten werden sie auf mehrere Dokumente aufgeteilt. Dies sollte beim beschaffen der Ressourcen beruecksichtigt werden. Da veraendert sich der Link dann zu:

**Beispiel:** `http://221.122.117.125/docin_134681478_2.docin`

# Aufdroeseln
Diese Binaerdatei kann jetzt Anhand der Funktion LoadTimerHandle weiter aufgedroeselt werden.

```actionscript
 private function loadTimerHandle(_arg1:TimerEvent):void{
    var _local2:ByteArray;
    var _local3:int;
    var _local4:Number;
    var _local5:Number;
    if (FirstLoadCtrl){
        return;
    };
    if (((!(isFirst)) && (((isComplete) || ((this._urlStream.bytesAvailable >= 16)))))){
        isFirst = true;
        _loadTotal = 16;
        constrcutLittleHeader();
        pageWidth = _urlStream.readInt();
        pageHeight = _urlStream.readInt();
        _totalPages = _urlStream.readInt();
        headerLength = _urlStream.readInt();
        _loadTotal = (_loadTotal + headerLength);
        if (this.currentPage == 1){
            this.dispatchXml();
        };
        this.setHeaderXml();
        isSecond = false;
    } else {
        if (((!(isSecond)) && (VoniboManager.InitLoadCtrl))){
            if (loadbarCount < 100){
                loadbarCount++;
            };
            PanelManager.getLaunchPanelInstance().percentage = ((loadbarCount + ((this._urlStream.bytesAvailable / _loadTotal) * 100)) * 0.5);
            trace((((loadbarCount + ((this._urlStream.bytesAvailable / _loadTotal) * 100)) * 0.5) + "------------sssssss"));
        };
        if (((!(isSecond)) && ((this._urlStream.bytesAvailable >= _loadTotal)))){
            isSecond = true;
            headerArray = new ByteArray();
            headerArray.endian = Endian.LITTLE_ENDIAN;
            this._urlStream.readBytes(headerArray, 0, headerLength);
            headerArray.uncompress();
            headerUncompressLength = headerArray.length;
            _loadTotal = (_loadTotal + 4);
            ctrlBoolean = true;
            isThird = false;
            PanelManager.getLaunchPanelInstance().close();
        } else {
            if (((!(isThird)) && ((((this._urlStream.bytesAvailable >= _loadTotal)) || (isComplete))))){
                isThird = true;
                if (ctrlBoolean){
                    ddd++;
                    bodyLen = this._urlStream.readInt();
                    _loadTotal = (_loadTotal + bodyLen);
                    ctrlBoolean = false;
                } else {
                    _local2 = new ByteArray();
                    _local2.endian = Endian.LITTLE_ENDIAN;
                    this._urlStream.readBytes(_local2, 0, bodyLen);
                    _local2.uncompress();
                    _local3 = ((this.headerUncompressLength + _local2.length) + 8);
                    littleHeaderArray.position = 4;
                    littleHeaderArray.writeInt(_local3);
                    littleHeaderArray.position = 0;
                    LoadingPanel(_pageContainer.sourceArray[_loadCount]).oldOrNew = true;
                    LoadingPanel(_pageContainer.sourceArray[_loadCount]).DisplayBytes = _local2;
                    littleHeaderArray.readBytes(LoadingPanel(_pageContainer.sourceArray[_loadCount])._littleHeaderArray);
                    _loadCount++;
                    _loadTotal = (_loadTotal + 4);
                    ctrlBoolean = true;
                    if (_loadCount == 1){
                        if (this.currentPage == 1){
                            CustomRequest.getInstance().statisticPage(this.startLoadTime, new Date(), _loadTotal, 1);
                        };
                        ExternalManager.closeFlashBox();
                    };
                };
                if (this._loadCount >= XML(this.xmlArray[(this.currentPage - 1)]).children().length()){
                    this.loadTimer.stop();
                    _local4 = ((this.endLoadTime.getTime() - this.startLoadTime.getTime()) / 1000);
                    _local5 = (_loadTotal / 0x0400);
                    if (this.currentPage == 1){
                        CustomRequest.getInstance().statisticPage(this.startLoadTime, this.endLoadTime, _loadTotal, 2);
                    };
                    VoniboManager.averageLoadSpeed = (("" + int((_local5 / _local4))) + " k/s");
                    return;
                };
                isThird = false;
            };
        };
    };
}
```
Die Datei startet mit 4 Integer in der Reihenfolge: pageWidth, pageHeight, totalPages und headerLength. Danach kann der Header ausgelesen werden, der aber komprimiert ist. In diesem Header wiederum steht der frame count, um die Laenge des Bodys zu bestimmen. Jetzt kommt nach und nach die Einzelnen Seiten mit jeweils einem Integer dazwischen um die Laenge des Bodys zu bestimmen. Der Payload selber ist immer komprimiert.

Jetzt bleibt die Frage, was der Body Payload eigentlich ist. Wie sich rausstellt ist das in einzelnen SWF Dateien abgespeichert.

Ganz einfach rauslesbar aus der Funktion `constrcutLittleHeader()`

```java
 private function constrcutLittleHeader():void{
    littleHeaderArray = new ByteArray();
    littleHeaderArray.endian = Endian.LITTLE_ENDIAN;
    littleHeaderArray.writeByte(70);
    littleHeaderArray.writeByte(87);
    littleHeaderArray.writeByte(83);
    littleHeaderArray.writeByte(9);
    littleHeaderArray.writeInt(totalLength);
}
```
Das baut einen SWF-Header zusammen.

Das ganze kann man recht schoen in Python Code fassen (der obere Code in Python):

```python
import sys
import struct
import zlib


class DocinLoader:
    def __init__(self,file):
        try:
            self.docin = open(file,'rb').read()
            self.header = ''
            self.pages = []
            self.totalPages = 0
            self.dimensions = (0,0)
            self._decrypt()
        except:
            print 'Something went terribly wrong'

    def _decrypt(self):
        self.dimensions = struct.unpack('ii',self.docin[0:8])
        self.totalPages = struct.unpack('i',self.docin[8:12])

        headerLength = struct.unpack('i',self.docin[12:16])[0]
        self.header = zlib.decompress(self.docin[16:16+headerLength]) # get the swf header
        n = struct.unpack('h',self.header[11:13])[0] # get frame count to know how long the body is

        offset = 16+headerLength
        while offset < len(self.docin):
            bodyLen = struct.unpack('i',self.docin[offset:offset+4])[0]
            temp = zlib.decompress(self.docin[offset+4:offset+4+bodyLen])
            self.pages.append(temp)
            offset+=bodyLen+4

    def getPage(self,p):
        #if isinstance(page,int): page = [page]
        header = self.header[0:11]+struct.pack('h',len(p))+self.header[13:]
        body = ''
        for i in p: body += self.pages[i]
        return self._constructSWF(header,body)

    def getDocument(self):
        return self._constructSWF(self.header,''.join(self.pages))

    def _constructSWF(self,header,body):
        return struct.pack('bbbb',70,87,83,9)+struct.pack('<i',len(header)+len(body)+8)+header+body

o = DocinLoader(sys.argv[1])
#print o.getPage([1,2,3,4]),
print o.getDocument(),
#print o.getDocument(),
```

# Final touch
Damit haetten wir eine SWF Datei. Diese ist aber nicht schoen zu lesen, aber du kommen uns die swftools zu Hilfe. Bisher kann ich nur SWF in PNG Rendern, aber nachdem SWF eigentlich ein Vektorformat ist, muesste sich das doch auch sinnvoll in eine PDF uebersetzen lassen...kann da nicht jemand mal was schreiben?

Die einzelnen Seiten sind einzelne Frames, lassen sich aber mit einem swfextract wunderschoen aus der SWF extrahieren.

```bash
#!/bin/bash

SWF="${1}"
N=$(swfdump "${SWF}" | grep "Frame count" | awk '{print $4}')

mkdir -p pages
for ((i=0; i<$N; i++)); do
    TEMP=$(mktemp)
    swfextract -f "$i" -o "${TEMP}" "${SWF}"
    swfrender "${TEMP}" -o "pages/${i}.png"
    rm "${TEMP}"
done
```
Diese einzelnen PNGs koennen dann wieder in eine PDF zusammengebaut werden. Leider ist damit keine Volltextsuche moeglich, weil er ja kein OCR drueberlaufen hat lassen, aber hey...das geht doch sicher auch noch irgendwie.

# Fazit
War mal wieder eine schoene Fingeruebung um sich mit ActionScript ein wenig mehr auseinanderzusetzen. Vielleicht hat noch jemand eine coole Idee am Ende eine schoene PDF rauszukriegen und nicht so ein Riesending mit Bildern.

so long
