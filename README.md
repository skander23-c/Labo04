#include <LCD_I2C.h>
#include <HCSR04.h>
#include <AccelStepper.h>


#define TYPE_MOTEUR 4
#define BROCHE_TRIG 9
#define BROCHE_ECHO 10
#define PIN_1 31
#define PIN_2 33
#define PIN_3 35
#define PIN_4 37

#define ANGLE_MIN 10
#define ANGLE_MAX 170


LCD_I2C ecran(0x27, 16, 2);
HCSR04 capteur(BROCHE_TRIG, BROCHE_ECHO);
AccelStepper moteur(TYPE_MOTEUR, PIN_1, PIN_3, PIN_2, PIN_4);

enum PositionObjet { OBJET_PROCHE, OBJET_LOIN, OBJET_CIBLE };
PositionObjet position = OBJET_LOIN;

float mesureDistance = 0;
int angleActuel = 0;
int ancienAngle = -1;

unsigned long tempsPrecedentDistance = 0;
unsigned long tempsPrecedentLCD = 0;
unsigned long tempsPrecedentSerie = 0;
const int TEMPS_MESURE = 50;
const int DISTANCE_MIN = 30;
const int DISTANCE_MAX = 60;

void initialiserSysteme() {
  Serial.begin(115200);
  ecran.begin();
  ecran.backlight();
  ecran.setCursor(0, 0);
  ecran.print("2344779");
  ecran.setCursor(0, 1);
  ecran.print("Labo 4b");
  delay(2000);
  ecran.clear();

  moteur.setMaxSpeed(500);
  moteur.setAcceleration(100);
}


void tacheMesureDistance() {
  if (millis() - tempsPrecedentDistance >= TEMPS_MESURE) {
    mesureDistance = capteur.dist();

    if (mesureDistance < DISTANCE_MIN) {
      position = OBJET_PROCHE;
      angleActuel = ANGLE_MIN;
    } else if (mesureDistance > DISTANCE_MAX) {
      position = OBJET_LOIN;
      angleActuel = ANGLE_MAX;
    } else {
      position = OBJET_CIBLE;
      angleActuel = map(mesureDistance, 30, 60, ANGLE_MIN, ANGLE_MAX);
    }

    tempsPrecedentDistance = millis();
  }
}

void tacheAffichageLCD() {
  if (millis() - tempsPrecedentLCD >= 100) {
    ecran.clear();
    ecran.setCursor(0, 0);
    ecran.print("Dist:");
    ecran.print((int)mesureDistance);
    ecran.print("cm");

    ecran.setCursor(0, 1);
    switch (position) {
      case OBJET_PROCHE:
        ecran.print("obj :Trop pres");
        break;
      case OBJET_LOIN:
        ecran.print("obj :Trop loin");
        break;
      case OBJET_CIBLE:
        ecran.print("obj:");
        ecran.print(angleActuel);
        ecran.print("deg");
        break;
    }

    tempsPrecedentLCD = millis();
  }
}

void tacheSerie() {
  if (millis() - tempsPrecedentSerie >= 100) {
    Serial.print("etd:");
    Serial.print("2344779");
    Serial.print(",dist:");
    Serial.print((int)mesureDistance);
    Serial.print(",deg:");
    Serial.println(angleActuel);
    tempsPrecedentSerie = millis();
  }
}

void tacheControleMoteur() {
  if (position == OBJET_CIBLE) {
    if (angleActuel != ancienAngle) {
      int cibleStep = map(angleActuel, ANGLE_MIN, ANGLE_MAX, 57, 967);
      moteur.enableOutputs();
      moteur.moveTo(cibleStep);
      ancienAngle = angleActuel;
    }
    moteur.run();
  } else {
    moteur.disableOutputs();
  }
}

// === Setup & Loop ===
void setup() {
  initialiserSysteme();
}

void loop() {
  tacheMesureDistance();
  tacheAffichageLCD();
  tacheSerie();
  tacheControleMoteur();
}
