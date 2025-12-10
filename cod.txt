#include <IRremote.h>

// --- Configuración de Pines ---
#define IR_RECEIVE_PIN 25     // Receptor IR
#define LED_RED_PIN 23        // LED Rojo (Luz 1)
#define LED_GREEN_PIN 22      // LED Verde (Luz 2)
#define BUZZER_PIN 4          // Pin D4 para el buzzer (PWM)

// --- Códigos de tu Mando ---
#define BOTON_ENCENDER 0x18 
#define BOTON_APAGAR   0x0C 

// --- Configuración LEDC (Buzzer) ---
#define LEDC_CHANNEL 0      // Canal LEDC para PWM
#define LEDC_TIMER_BIT 13

// --- Estados del Sistema ---
enum SystemState {
  OFF,
  ON_PLAYING
};
SystemState currentState = OFF;

// --- Variables para control de la Melodía (Sin Delay) ---
int melody[] = {659, 659, 659, 659, 659, 659, 659, 784, 523, 587, 659};
int duration[] = {250, 250, 500, 250, 250, 500, 250, 250, 250, 250, 500};
int notes = sizeof(melody) / sizeof(melody[0]);
int noteIndex = 0;
unsigned long noteStartTime = 0;

// --------------------------- CONTROL DE TONO SIN BLOQUEO ---------------------------

void updateMelody() {
  if (currentState != ON_PLAYING) {
    ledcWriteTone(LEDC_CHANNEL, 0);
    return;
  }
  
  unsigned long currentTime = millis();
  
  // 1. Comprobar si es hora de pasar a la siguiente nota (o pausa)
  if (currentTime - noteStartTime >= duration[noteIndex] + 30) { // Duración + 30ms de pausa
    
    // Avanzar al siguiente índice de nota
    noteIndex++;
    if (noteIndex >= notes) {
      noteIndex = 0; // Reiniciar la melodía (bucle infinito)
    }
    
    // Iniciar la nueva nota
    noteStartTime = currentTime;
    ledcWriteTone(LEDC_CHANNEL, melody[noteIndex]);
  }
}

// --------------------------- SETUP ---------------------------
void setup() {
  Serial.begin(115200);
  
  // Inicialización de Pines como Salida
  pinMode(LED_RED_PIN, OUTPUT);
  pinMode(LED_GREEN_PIN, OUTPUT); 
  
  // Inicialización del Buzzer (LEDC)
  ledcSetup(LEDC_CHANNEL, 0, LEDC_TIMER_BIT);
  ledcAttachPin(BUZZER_PIN, LEDC_CHANNEL);
  
  // Iniciar el receptor IR
  IrReceiver.begin(IR_RECEIVE_PIN, ENABLE_LED_FEEDBACK); 
  
  // Estado inicial: Todo apagado
  digitalWrite(LED_RED_PIN, LOW);
  digitalWrite(LED_GREEN_PIN, LOW);
  ledcWriteTone(LEDC_CHANNEL, 0); 

  Serial.println("--- Control IR Instantáneo Listo ---");
}

// --------------------------- LOOP ---------------------------
void loop() {
  
  // 1. Lógica de Recepción IR (Instantánea)
  if (IrReceiver.decode()) {
    
    if (!(IrReceiver.decodedIRData.flags & IRDATA_FLAGS_IS_REPEAT)) {

        if (IrReceiver.decodedIRData.command == BOTON_ENCENDER) {
          // ENCENDER
          currentState = ON_PLAYING;
          digitalWrite(LED_RED_PIN, HIGH);
          digitalWrite(LED_GREEN_PIN, HIGH);
          
          // Reiniciar el índice de la melodía para comenzar
          noteIndex = 0;
          noteStartTime = millis(); 
          ledcWriteTone(LEDC_CHANNEL, melody[noteIndex]); // Tocar la primera nota
          
          Serial.println(">>> ¡ENCENDIDO! (Instantáneo)");
        } 
        else if (IrReceiver.decodedIRData.command == BOTON_APAGAR) {
          // APAGAR (Instantáneo)
          currentState = OFF;
          digitalWrite(LED_RED_PIN, LOW);
          digitalWrite(LED_GREEN_PIN, LOW);
          ledcWriteTone(LEDC_CHANNEL, 0); // Silencia el buzzer instantáneamente
          
          Serial.println(">>> ¡APAGADO! (Instantáneo)");
        }
    }
    IrReceiver.resume(); 
  }
  
  // 2. Lógica de Reproducción de Melodía (Sin Bloqueo)
  updateMelody();
  
}