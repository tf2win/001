# 001
Fjarlægðamælir og vefsíða 

Það sem þarf að gera fyrst er að ná í þennan pakka 

sudo apt-get update
sudo apt-get install python3-flask
 Síðan kemur forritið 

# Flytjum inn nauðsynleg forritasöfn:
from flask import Flask, jsonify, render_template_string  # Flask fyrir vefinn, jsonify fyrir JSON gögn
import RPi.GPIO as GPIO  # Notað til að stjórna GPIO pinnum á Raspberry Pi
import time              # Til að mæla tíma í fjarlægðarútreikningum

# Búum til Flask vef-forrit
app = Flask(__name__)

# Skilgreinum hvaða GPIO-pinna eru notaðir fyrir ultrasonic skynjara
TRIG, ECHO = 23, 24

GPIO.setwarnings(False)           # Slökkvum á viðvörunum ef pinnar eru þegar í notkun
GPIO.setmode(GPIO.BCM)            # Notum BCM skíráningu fyrir pinnana (frekar en physical order)
GPIO.setup(TRIG, GPIO.OUT)        # TRIG (sendingarpinni) er stilltur sem útgangur
GPIO.setup(ECHO, GPIO.IN)         # ECHO (móttökupinni) er stilltur sem inngangur

# Fall sem mælir fjarlægð með ultrasonic skynjara
def get_distance():
    # Sendum mjög stuttan háspennu-púls á TRIG
    GPIO.output(TRIG, True)
    time.sleep(0.00001)  # 10 míkrósekúndur
    GPIO.output(TRIG, False)

    # Mælum hvenær við fáum endurkast frá hljóðbylgju (byrjun og endir)
    while GPIO.input(ECHO) == 0:
        start = time.time()
    while GPIO.input(ECHO) == 1:
        end = time.time()

    # Reiknum tíma sem bylgjan tók og umbreytum því í fjarlægð í cm
    return round((end - start) * 17150, 2)  # hraði hljóðs í lofti: ~343 m/s → 17150 cm/s (frem og til baka)

# Fall sem breytir fjarlægð yfir í lítra
def calculate_liters(distance):
    max_height = 100  # Hámarks hæð t.d. á vatnstank í cm
    # Reiknar hversu mikið tankurinn er fullur hlutfallslega og margfaldar með 100 lítrum
    return round(max(0, (1 - distance / max_height)) * 100, 1)

# Aðalsíða: HTML + JavaScript sem birtir mælingar og uppfærir þær sjálfkrafa
@app.route('/')
def index():
    return render_template_string(
        # HTML og JavaScript í einni línu fyrir einfaldan vef
        '<h1>📏 <span id=d>...</span></h1>'  # Fjarlægð verður birt hér
        '<h2>💧 <span id=l>...</span></h2>'  # Rúmmál verður birt hér
        '<script>'
        'setInterval(()=>'  # Keyrir á 1000ms (1 sek fresti)
        'fetch("/data")'    # Sækir nýjustu mæligögn frá /data route
        '.then(r=>r.json())'
        '.then(g=>{'        # Þegar gögnin eru móttekin...
        'd.innerText=g.distance+" cm";'  # ...birtum fjarlægð
        'l.innerText=g.liters+" lítrar"})'
        ',1000)'            # Endurtaka á hverri sekúndu
        '</script>'
    )

# Þessi route skilar mæligögnum sem JSON – sem JS á síðunni les
@app.route('/data')
def data():
    d = get_distance()  # Fáum fjarlægð
    l = calculate_liters(d)  # Reiknum lítra út frá fjarlægð
    return jsonify({'distance': d, 'liters': l})  # Skilum JSON svari

# Keyrir Flask vefþjón og lokar GPIO-pinnum þegar hætt er
if __name__ == '__main__':
    try:
        app.run(host='0.0.0.0', port=5000)  # Keyrir vefinn á öllum IP tölum Raspberry Pi (getur séð úr vafra á netinu)
    finally:
        GPIO.cleanup()  # Lætur GPIO-pinna hætta notkun og forðast vandamál við næstu keyrslu

