# fhem-rhasspy-gettingstarted
Kurze Anleitung für Anfänger um Rhasspy in FHEM ans Laufen zu bringen

Zum Zeitpunkt der Erstellung dieser kleinen Anleitung war vieles im Umbruch. 
SNIPS funktioniert so nicht mehr, die Anleitungen für SNIPS treffen teilweise nicht mehr zu und Rhasspy steht noch in den Anfängen.
Ich selbst hatte leider viele Stunden mit Stolpersteinen und Rumprobieren verschwendet und schreibe deswegen diese kleine Anleitung.

Es ist eine Momentaufnahme, die für mich persönlich Stand 2021-01 mit 2021-01-11-raspios-buster-armhf-lite auf zwei Raspberry Pis funktioniert hat. 

# Voraussetzungen
Ich gehe davon aus, dass Du eine funktionierende FHEM Installation hast. 
In meinem Fall ist das ein FHEM auf einem Raspberry Pi 2B (buster lite) mit Homematic CUL, einem Jeelink und u.a. ein paar Tasmota MQTT-Devices.

Weiterhin brauchst Du ein Mikrofon. Ich habe mich dabei für ein PS3eye entschieden, welches man gebraucht für 10-15 Euro bekommen kann.
Rhasspy läuft bei mir auf einem eigenen Raspberry 4 (buster lite). Ich hatte ursprünglich einmal erwogen, es auch auf meinen Raspberry 2B mit FHEM zu packen, aber die Reaktionen sind dann etwas zäh und Spracherkennung macht nicht ganz so viel Spaß. 

# Zielsetzung
Hinterher kannst Du per Wakeword und anschließender Spracheingabe Dinge in FHEM steuern und Dir ansagen lassen.


# 1. Rhasspy vorbereiten
Die Installation von Rhasspy wird ausführlich unter https://rhasspy.readthedocs.io/en/latest/installation/ beschrieben. 
Ich fasse das kurz zusammen und gehe auf die Besonderheiten ein:
<p>
Docker:<br>
curl -sSL https://get.docker.com | sh<br>
sudo usermod -a -G docker $USER<br>
sudo reboot<br>
<p>

Was die Installation und das Starten des Docker-Images angeht, so gilt folgende Besonderheit: Ich brauchte noch einen zusätzlichen Parameter <br>
      -p 12183:12183 \ <br>
damit der Rhasspy-MQTT-Server auch von außen über Port 12183 ansprechbar ist. Außerdem habe ich mich für das deutsche Profil entschieden. Somit sieht mein Aufruf so aus:
<p>

docker run -d -p 12101:12101 \
      -p 12183:12183 \
      --name rhasspy \
      --restart unless-stopped \
      -v "$HOME/.config/rhasspy/profiles:/profiles" \
      -v "/etc/localtime:/etc/localtime:ro" \
      --device /dev/snd:/dev/snd \
      rhasspy/rhasspy \
      --user-profiles /profiles \
      --profile de

Die Rhasspy-Gui ist nun unter <ip_des_rhasspy>:12101 erreichbar. Ich habe mich für folgende Settings entschieden (links das Zahnrad-Symbol):
<p>
Audio Recording: PyAudio<br>
Wake Word: Porcupine<br>
Speech to Text: Pocketsphinx<br>
Intent Recognition: Fsticuffs<br>
Text to Speech: PicoTTS<br>
Audio Playing: aplay<br>
Dialogue Management: Rhasspy<br>
<p>
Laut Tutorial sollte man unter "Audio Playing" die Devices testen können und die funktionierenden devices würden dann "working" hinten dran stehen haben.
Das war bei mir nicht der Fall und ich habe nichts geändert aber die Default-Einstellungen haben trotzdem funktioniert.

=> Save Settings.<br>
Links auf das Haus-Symbol und den blauen Download-Knopf vor "Rhasspy needs to download some files for your profile" drücken.
Wörter, bei deren Aussprache er nicht sicher ist einfach bestätigen.<br>

Unter dem Haus-Symbol bei Sentences nun diese Einträge hinzufügen:
<p>
[de.fhem:SetOnOff]<br>
schalte (die | das) (wohnzimmerlampe | stehlampe){Device} (an | ein | aus){Value}<br>
</p>
<p>
[de.fhem:GetNumeric]<br>
(wie ist die|wie warm ist es){Type:Temperatur} [temperatur] [(thermometer){Device}] [(im |in der)] [(wohnzimmer|bad|schlafzimmer|kinderzimmer|garage|draussen){Room}]<br>
</p>
<p>
[de.fhem:GetTime]<br>
wie spät ist es<br>
sag mir die uhrzeit<br>
</p>

=> Save Sentences + retrain

In der aktuellen Rhasspy Version  2.5.9 ist es nun auch möglich, ohne größere Klimmzüge ein anderes Wakeword als Porcupine zu verwenden. Hierzu einfach auf "Wakewords" und im Dropdown das gewünschte Keyword aussuchen. Ich bin bei "Alexa" hängen geblieben, weil es kurz ist und sehr gut erkannt wird. Meiner Erfahrung nach sollte man das erste A etwas länger aussprechen, also eher Aalexa. 

Zwischenstand: Wenn man nun in Rhasspy auf das Häuschen klickt, "Alexa" sagt, kurz wartet und dann "Schalte die Stehlampe an" sagt, dann sollte er alles korrekt erkennen, aber natürlich noch nichts tun.

# 2. FHEM einrichten
Einloggen auf die FHEM-Maschine:
<p>
cd /opt/fhem/FHEM/<br>
sudo wget "https://raw.githubusercontent.com/drhirn/fhem-rhasspy/master/10_RHASSPY.pm"<br>
sudo chown fhem:dialout 10_RHASSPY.pm<br>
<p>
In Fhem ein shutdown restart machen und im fhem.log prüfen, ob es Meldungen gibt. Bei mir gab es noch zwei fehlende Dependencies für DateTime und Pluggable, welche ich durch
<p>
sudo apt-get install libdatetime-perl<br>
sudo apt-get install libmodule-pluggable-perl<br>
<p>
behoben habe. 

In FHEM folgendes angelegt:
<p>
define RhasspyMQTT MQTT [ip_des_rhasspy]:12183<br>
define Rhasspy RHASSPY RhasspyMQTT Wohnzimmer<br>
attr RhasspyMQTT room Rhasspy<br>
attr RhasspyMQTT rhasspyRoom Wohnzimmer<br>
attr Rhasspy room Rhasspy<br>
attr Rhasspy rhasspyRoom Wohnzimmer<br>
<p>

=> Save config

Damit ich nun meine Gosund SP111 MQTT-Steckdose mit dem Namen MQTT2_SP111a auch ansprechen und schalten kann:
<p>

attr MQTT2_SP111a rhasspyName stehlampe<br>
attr MQTT2_SP111a room Rhasspy,MQTT2_DEVICE<br>
attr MQTT2_SP111a rhasspyRoom Wohnzimmer<br>
attr MQTT2_SP111a rhasspyMapping SetOnOff:cmdOn=on,cmdOff=off<br>
<p>

=> Save config

Nun sollte eine Schaltung per "Alexa [pause] Schalte die Stehlampe an " funktionieren.
<p>

Ansage Temperatur:
In Rhasspy diesen Sentence anlegen:<br>
[de.fhem:GetNumeric]<br>
(wie ist die|wie warm ist es){Type:Temperatur} [temperatur] [(thermometer){Device}] im [(wohnzimmer|bad|schlafzimmer|kinderzimmer){Room}]<br>
<p>
In FHEM:<br>
attr Temp_Schlafzimmer rhasspyName Temperatur<br>
attr Temp_Schlafzimmer room Rhasspy,CUL_HM<br>
attr Temp_Schlafzimmer rhasspyRoom schlafzimmer<br>
attr Temp_Schlafzimmer rhasspyMapping GetNumeric:currentVal=temperature,type=Temperatur<br>
<p>
Und für einen anderen Temperatursensor - in meinem Fall einen Jeelink im Wohnzimmer: <br> 
attr TempHum_Wohnzimmer rhasspyName Temperatur<br>
attr TempHum_Wohnzimmer room Rhasspy,Lacrosse<br>
attr TempHum_Wohnzimmer rhasspyRoom wohnzimmer<br>
attr TempHum_Wohnzimmer rhasspyMapping GetNumeric:currentVal=temperature,type=Temperatur<br>
<p>
Die Basis ist hiermit gelegt und weiteren persönlichen Experimenten steht nichts im Wege. Viel Spaß damit. 
