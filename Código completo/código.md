Adjuntamos código completo:

// ==== Librerías ====
#include <Servo.h>    // Permite controlar servomotores
#include <Stepper.h>  // Librería para motor paso a paso

// ==== Declaración de objetos Servo ====
Servo base;
Servo hombro;
Servo codo;
Servo muneca; 
Servo pinza;

// ==== Configuración Motor Paso a Paso (PAP) ====
// Pasos por revolución (ajusta esto a tu motor, 2048 es común para 28BYJ-48)
const int STEPS_PER_REV = 2048; 
const int motorPapPin1 = 9;     // Pin IN1 del driver ULN2003
const int motorPapPin2 = 10;    // Pin IN2 del driver ULN2003
const int motorPapPin3 = 11;    // Pin IN3 del driver ULN2003
const int motorPapPin4 = 12;    // Pin IN4 del driver ULN2003

// Inicializa el objeto Stepper (IMPORTANTE: Verifica el orden de los pines 
// para tu driver: Stepper(pasos, in1, in3, in2, in4) para una conexión común)
Stepper motorPAP(STEPS_PER_REV, motorPapPin1, motorPapPin3, motorPapPin2, motorPapPin4);

// ==== Pines de conexión ====
const int sensorProximidad = 2; // Pin digital de entrada del sensor (ej. IR o ultrasónico)
const int servoBase = 3;      // Pin PWM para servo base
const int servoHombro = 4;    // Pin PWM para servo hombro
const int servoCodo = 5;      // Pin PWM para servo codo
const int servoMuneca = 6;    // Pin PWM para servo muñeca
const int servoPinza = 7;     // Pin PWM para servo pinza


// ==== CONSTANTE DE VELOCIDAD ====
// Define la pausa (en milisegundos) por cada grado de movimiento.
// Un número MÁS ALTO = movimiento MÁS LENTO y suave.
const int VELOCIDAD_MOV = 20; // (Prueba con 15, 20 o 30)
const int VELOCIDAD_PAP = 10; // RPM del motor PAP

// ==== Variables ====
int estadoSensor = 0;           // Almacena el estado actual del sensor

// --------------------------------------------------------------
// FUNCIÓN PARA MOVIMIENTO LENTO Y SUAVE (Servos)
// --------------------------------------------------------------
/**
 * Mueve un servo gradualmente desde su posición actual a una nueva posición.
 * @param servo El objeto Servo que se va a mover (ej. 'base', 'hombro')
 * @param anguloDestino El ángulo final al que debe llegar el servo.
 * @param velocidad La pausa en ms entre cada grado (más alto = más lento).
 */
void moverServoSuave(Servo& servo, int anguloDestino, int velocidad) {
  int anguloActual = servo.read(); // Lee la posición actual del servo

  // Mover hacia el destino gradualmente
  if (anguloActual < anguloDestino) {
    for (int pos = anguloActual; pos <= anguloDestino; pos++) {
      servo.write(pos);
      delay(velocidad); // Pausa para movimiento lento
    }
  }
  else if (anguloActual > anguloDestino) {
    for (int pos = anguloActual; pos >= anguloDestino; pos--) {
      servo.write(pos);
      delay(velocidad); // Pausa para movimiento lento
    }
  }
}

// --------------------------------------------------------------
// FUNCIÓN PARA MOVER CINTA TRANSPORTADORA (Motor PAP)
// --------------------------------------------------------------
/**
 * Mueve el motor PAP una cantidad específica de pasos.
 * @param pasos Número de pasos a mover (positivo = una dirección, negativo = otra).
 */
void moverCinta(int pasos) {
  Serial.print("Moviendo cinta: ");
  Serial.print(pasos);
  Serial.println(" pasos.");
  motorPAP.step(pasos);
}


// --------------------------------------------------------------
// FUNCIONES DEL SISTEMA
// --------------------------------------------------------------

// Coloca el brazo en posición de inicio/reposo
void posicionInicial() {
  Serial.println("Moviendo a posicion de reposo (lento)...");
  // La secuencia de movimientos es crítica para evitar colisiones mecánicas.
  moverServoSuave(hombro, 0, VELOCIDAD_MOV); //4
  moverServoSuave(codo, 180, VELOCIDAD_MOV);//5
  moverServoSuave(muneca, 60, VELOCIDAD_MOV);//6
  moverServoSuave(base, 120, VELOCIDAD_MOV);//3
  moverServoSuave(pinza, 90, VELOCIDAD_MOV); //7 Pinza abierta
  }

// Movimiento de agarre y traslado del brazo robótico
void moverBrazo() {
  Serial.println("Iniciando secuencia de agarre (lento)...");

  // 1. Mover hacia la posición de agarre sobre la cinta
  Serial.println("Paso 1: Moviendo a posicion de agarre");
  moverServoSuave(base, 153, VELOCIDAD_MOV);
  moverServoSuave(hombro, 10, VELOCIDAD_MOV);
  moverServoSuave(codo, 100, VELOCIDAD_MOV);
  // 2. Cerrar pinza para agarrar objeto
  Serial.println("Paso 2: Cerrando pinza");
  moverServoSuave(pinza, 180, VELOCIDAD_MOV); // Usando 180 para cerrado (ajustar según tu pinza)
  delay(500); // Pequeña pausa para asegurar el agarre

  // 3. Levantar el objeto
  Serial.println("Paso 3: Levantando objeto");
  moverServoSuave(hombro, -20, VELOCIDAD_MOV);
  moverServoSuave(codo, 180, VELOCIDAD_MOV);

  // 4. Girar base para mover objeto a la posición de descarga
  Serial.println("Paso 4: Girando base");
  moverServoSuave(base, 90, VELOCIDAD_MOV);

  // 5. Bajar brazo para soltar objeto
  Serial.println("Paso 5: Bajando para soltar");
  moverServoSuave(hombro, 60, VELOCIDAD_MOV);
  moverServoSuave(codo, 100, VELOCIDAD_MOV);

  // 6. Abrir pinza para soltar el objeto
  Serial.println("Paso 6: Abriendo pinza");
  moverServoSuave(pinza, 90, VELOCIDAD_MOV); // Abierta (ajustar según tu pinza)
  delay(500);

  // 7. Mover la cinta para retirar el objeto clasificado
  moverCinta(STEPS_PER_REV / 2); // Mover media revolución para simular que el objeto se aleja

  // 8. Volver a posición inicial
  Serial.println("Paso 8: Volviendo a reposo");
  posicionInicial();

  Serial.println("Secuencia finalizada.");
}

// --------------------------------------------------------------
// CONFIGURACIÓN INICIAL
// --------------------------------------------------------------
void setup() {
  // Inicializar comunicación serie
  Serial.begin(9600);

  // Configurar pin del sensor
  pinMode(sensorProximidad, INPUT);
  
  // Configurar velocidad del motor PAP (ej. 10 RPM)
  motorPAP.setSpeed(VELOCIDAD_PAP);

  // Asociar servos a sus pines
  base.attach(servoBase);
  hombro.attach(servoHombro);
  codo.attach(servoCodo);
  muneca.attach(servoMuneca); 
  pinza.attach(servoPinza);

  // Posición inicial del brazo (reposo)
  posicionInicial();

  Serial.println("Sistema iniciado: Brazo en reposo. Esperando objeto...");
}

// --------------------------------------------------------------
// BUCLE PRINCIPAL
// --------------------------------------------------------------
void loop() {
  // Leer el estado del sensor
  estadoSensor = digitalRead(sensorProximidad);
  
  // Si el sensor detecta un objeto (asumiendo que detecta con LOW, como en el código original)
  if (estadoSensor == LOW) {
    // --- OBJETO DETECTADO (SENSOR ACTIVO BAJO) ---
    Serial.println("Objeto detectado. Deteniendo cinta e iniciando secuencia de brazo...");
    
    // Realizar la secuencia de agarre y traslado (la cinta se detiene implícitamente
    // ya que no se llama a motorPAP.step() hasta el final de la secuencia)
    moverBrazo();

    // Pausa breve para que el sistema se estabilice antes de buscar el siguiente objeto.
    // Esto previene detecciones falsas o re-detección inmediata del objeto movido.
    delay(3000); 
    Serial.println("Reanudando ciclo de detección.");

  } else {
    // --- NO HAY OBJETO (SENSOR ACTIVO ALTO) ---
    // El motor PAP funciona continuamente para mover la cinta
    motorPAP.step(1); // Da 1 paso (mueve la cinta lentamente)
  }
}

