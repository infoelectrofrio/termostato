#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int botonBlancoPin = 11;
const int botonAzulPin = 6;
const int botonRojoPin = 9;
const int relePin = 4;
const int releTermostatoPin = 5;

enum EstadoTermostato {
  APAGADO,
  ENCENDIDO
};

EstadoTermostato estadoActual = APAGADO;
unsigned long tiempoUltimaPulsacion = 0;
unsigned long tiempoEncendido = 0;
unsigned long tiempoMostrarNuevoValor = 0;
const unsigned long tiempoDebounce = 200;
const unsigned long tiempoApagarLuzFondo = 4400000;
const unsigned long tiempoMostrarNuevoValorLCD = 1000;
const unsigned long tiempoVolverPantallaPrincipal = 20000;
const unsigned long tiempoEncenderTermostato = 20000;
float temperaturaObjetivo = 22.0;
const float histereisis = 1.0;

#define DHTPIN 3
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);
float temperature = 0.0;
unsigned long tiempoUltimaActualizacion = 0;
bool mostrarTemperatura = false;
bool ajustarTemperatura = false;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void mostrarMensajeLCD(const char* mensaje, int duracion) {
  lcd.clear();
  lcd.backlight();
  lcd.print(mensaje);
  delay(duracion);
  if (estadoActual == ENCENDIDO) {
    lcd.backlight();
  } else {
    lcd.noBacklight();
  }
}

void activarRele(int pin, unsigned long duracion) {
  digitalWrite(pin, LOW);
  delay(duracion);
  digitalWrite(pin, HIGH);
}

void setup() {
  Wire.begin();
  lcd.begin(16, 2);
  lcd.noBacklight();
  pinMode(botonBlancoPin, INPUT_PULLUP);
  pinMode(botonAzulPin, INPUT_PULLUP);
  pinMode(botonRojoPin, INPUT_PULLUP);
  pinMode(relePin, OUTPUT);
  pinMode(releTermostatoPin, OUTPUT);
  digitalWrite(relePin, HIGH);
  digitalWrite(releTermostatoPin, HIGH);
  dht.begin();
}

void loop() {
  unsigned long tiempoActual = millis();

  temperature = dht.readTemperature();

  if (ajustarTemperatura) {
    if (digitalRead(botonRojoPin) == LOW) {
      temperaturaObjetivo += 0.5;
      mostrarMensajeLCD(("Temp: " + String(temperaturaObjetivo) + " C").c_str(), tiempoMostrarNuevoValorLCD);
      tiempoMostrarNuevoValor = tiempoActual + tiempoMostrarNuevoValorLCD;
      tiempoUltimaPulsacion = tiempoActual; // Restablecer el tiempo de última pulsación
    } else if (digitalRead(botonAzulPin) == LOW) {
      temperaturaObjetivo -= 0.5;
      mostrarMensajeLCD(("Temp: " + String(temperaturaObjetivo) + " C").c_str(), tiempoMostrarNuevoValorLCD);
      tiempoMostrarNuevoValor = tiempoActual + tiempoMostrarNuevoValorLCD;
      tiempoUltimaPulsacion = tiempoActual; // Restablecer el tiempo de última pulsación
    }

    while (digitalRead(botonRojoPin) == LOW || digitalRead(botonAzulPin) == LOW) {
      delay(10);
    }

    ajustarTemperatura = false;
  }

  switch (estadoActual) {
    case APAGADO:
      if (digitalRead(botonBlancoPin) == LOW) {
        estadoActual = ENCENDIDO;
        activarRele(relePin, 250);
        lcd.clear();
        lcd.backlight();
        lcd.print("Bungalow 102");
        mostrarTemperatura = true;
        tiempoEncendido = tiempoActual;
        tiempoUltimaActualizacion = tiempoActual;
        tiempoMostrarNuevoValor = tiempoActual + tiempoVolverPantallaPrincipal;
        digitalWrite(releTermostatoPin, HIGH);
        delay(tiempoEncenderTermostato);
        digitalWrite(releTermostatoPin, LOW);
        delay(500);
      }
      break;

    case ENCENDIDO:
      if (tiempoActual - tiempoEncendido >= tiempoApagarLuzFondo) {
        lcd.noBacklight();
      } else {
        lcd.backlight();

        if (mostrarTemperatura && tiempoActual - tiempoUltimaActualizacion >= 1000) {
          lcd.setCursor(0, 1);
          lcd.print("Temp: ");
          lcd.print(temperature, 1);
          lcd.print(" C");
          tiempoUltimaActualizacion = tiempoActual;
        }

        // Ajustar la temperatura objetivo a 22°C cuando el termostato se encienda
        temperaturaObjetivo = 22.0;

        if (temperature > temperaturaObjetivo + histereisis) {
          digitalWrite(releTermostatoPin, LOW);
        } else if (temperature < temperaturaObjetivo - histereisis) {
          digitalWrite(releTermostatoPin, HIGH);
        }

        if (tiempoActual >= tiempoMostrarNuevoValor) {
          lcd.clear();
          lcd.print("Bungalow 102 ");
          lcd.print(temperature, 1);
          lcd.print(" C");
          tiempoMostrarNuevoValor = tiempoActual + tiempoVolverPantallaPrincipal;
        }
      }

      if (digitalRead(botonBlancoPin) == LOW) {
        estadoActual = APAGADO;
        activarRele(relePin, 250);
        delay(1000);
        activarRele(relePin, 250);
        digitalWrite(releTermostatoPin, HIGH);
        lcd.clear();
        lcd.noBacklight();
        delay(500);
      }

      if (digitalRead(botonRojoPin) == LOW || digitalRead(botonAzulPin) == LOW) {
        ajustarTemperatura = true;

        // Regresar a la pantalla principal después de 20 segundos
        if (tiempoActual - tiempoUltimaPulsacion >= tiempoVolverPantallaPrincipal) {
          lcd.clear();
          lcd.backlight();
          lcd.print("Bungalow 1");
          lcd.setCursor(0, 1);
          lcd.print("Temp: ");
          lcd.print(temperature, 1);
          lcd.print(" C");
          tiempoUltimaPulsacion = tiempoActual;
        }
      }
      break;
  }
}

