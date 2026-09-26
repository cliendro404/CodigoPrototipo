# CodigoPrototipo

#include <Servo.h> 

#include <EEPROM.h> 

 
 

/* Locking mechanism definitions */ 

#define SERVO_PIN 6 

#define SERVO_LOCK_POS   20 

#define SERVO_UNLOCK_POS 90 

Servo lockServo; 

 
 

/* Buzzer and LED definitions */ 

#define BUZZER_PIN A4 

#define LED_PIN A5 

 
 

/* EEPROM address for storing the code */ 

#define EEPROM_CODE_ADDR 0 

#define CODE_LENGTH 4 

#define CODE_VALID_FLAG 0xAA 

 
 

/*  

 * IMPORTANTE: En Wokwi no existe el módulo Bluetooth HC-05, 

 * así que usamos el Monitor Serie como interfaz virtual. 

 * En un proyecto real, reemplaza 'Serial' por tu SoftwareSerial 

 * o HardwareSerial conectado al módulo Bluetooth. 

 */ 

#define bluetooth Serial 

 
 

/* Estado del sistema */ 

bool isLocked = true; 

String inputBuffer = ""; 

unsigned long lastActivityTime = 0; 

const unsigned long AUTO_LOCK_TIMEOUT = 30000; // 30 segundos 

 
 

/* Funciones de Buzzer */ 

void beepSuccess() { 

  tone(BUZZER_PIN, 1000, 200); 

  delay(300); 

  tone(BUZZER_PIN, 1500, 200); 

} 

 
 

void beepError() { 

  tone(BUZZER_PIN, 200, 300); 

  delay(400); 

  tone(BUZZER_PIN, 200, 300); 

} 

 
 

void beepKeyPress() { 

  tone(BUZZER_PIN, 500, 50); 

} 

 
 

void beepWarning() { 

  tone(BUZZER_PIN, 800, 100); 

  delay(150); 

  tone(BUZZER_PIN, 800, 100); 

} 

 
 

/* Funciones de LED */ 

void ledSuccess() { 

  digitalWrite(LED_PIN, HIGH); 

  delay(500); 

  digitalWrite(LED_PIN, LOW); 

  delay(200); 

  digitalWrite(LED_PIN, HIGH); 

  delay(500); 

  digitalWrite(LED_PIN, LOW); 

} 

 
 

void ledError() { 

  for (int i = 0; i < 3; i++) { 

    digitalWrite(LED_PIN, HIGH); 

    delay(200); 

    digitalWrite(LED_PIN, LOW); 

    delay(200); 

  } 

} 

 
 

/* Enviar mensaje al Monitor Serie (reemplaza lcd.print) */ 

void sendMessage(String msg) { 

  bluetooth.println(msg); 

} 

 
 

/* Limpiar el buffer de entrada */ 

void clearBuffer() { 

  inputBuffer = ""; 

} 

 
 

/* Leer el código desde EEPROM */ 

String readCodeFromEEPROM() { 

  String code = ""; 

  if (EEPROM.read(EEPROM_CODE_ADDR) == CODE_VALID_FLAG) { 

    for (int i = 0; i < CODE_LENGTH; i++) { 

      code += char(EEPROM.read(EEPROM_CODE_ADDR + 1 + i)); 

    } 

  } 

  return code; 

} 

 
 

/* Guardar el código en EEPROM */ 

void saveCodeToEEPROM(String code) { 

  EEPROM.write(EEPROM_CODE_ADDR, CODE_VALID_FLAG); 

  for (int i = 0; i < CODE_LENGTH; i++) { 

    EEPROM.write(EEPROM_CODE_ADDR + 1 + i, code[i]); 

  } 

} 

 
 

/* Verificar si hay un código guardado */ 

bool hasCode() { 

  return (EEPROM.read(EEPROM_CODE_ADDR) == CODE_VALID_FLAG); 

} 

 
 

/* Verificar si un String contiene solo dígitos */ 

bool isNumeric(String str) { 

  if (str.length() == 0) return false; 

  for (int i = 0; i < str.length(); i++) { 

    if (!isDigit(str[i])) return false; 

  } 

  return true; 

} 

 
 

/* Bloquear la caja fuerte */ 

void lock() { 

  lockServo.write(SERVO_LOCK_POS); 

  isLocked = true; 

  digitalWrite(LED_PIN, LOW); 

  beepKeyPress(); 

  sendMessage("LOCKED"); 

  sendMessage("La caja esta CERRADA"); 

} 

 
 

/* Desbloquear la caja fuerte */ 

void unlock() { 

  lockServo.write(SERVO_UNLOCK_POS); 

  isLocked = false; 

  digitalWrite(LED_PIN, HIGH); 

  beepSuccess(); 

  ledSuccess(); 

  lastActivityTime = millis(); 

  sendMessage("UNLOCKED"); 

  sendMessage("La caja esta ABIERTA"); 

  sendMessage("Escriba '#' para cerrar"); 

  sendMessage("Escriba 'A' para nueva clave"); 

} 

 
 

/* Esperar a recibir un código de 4 dígitos */ 

String waitForCode(unsigned long timeoutMs) { 

  String result = ""; 

  unsigned long startTime = millis(); 

 
 

  while (result.length() < CODE_LENGTH) { 

    if (millis() - startTime > timeoutMs) { 

      return ""; 

    } 

 
 

    if (bluetooth.available()) { 

      char c = bluetooth.read(); 

      if (c >= '0' && c <= '9') { 

        result += c; 

        beepKeyPress(); 

        sendMessage("*"); 

      } 

    } 

  } 

  return result; 

} 

 
 

/* Solicitar y validar una nueva clave */ 

bool setNewCode() { 

  sendMessage("NEW_CODE"); 

  sendMessage("Ingrese NUEVA clave (4 digitos):"); 

 
 

  String newCode = waitForCode(15000); 

 
 

  if (newCode.length() != CODE_LENGTH) { 

    sendMessage("TIMEOUT"); 

    sendMessage("No se recibio la clave"); 

    return false; 

  } 

 
 

  sendMessage("CONFIRM_CODE"); 

  sendMessage("Confirme la clave (4 digitos):"); 

 
 

  String confirmCode = waitForCode(15000); 

 
 

  if (newCode == confirmCode) { 

    saveCodeToEEPROM(newCode); 

    sendMessage("CODE_SAVED"); 

    sendMessage("Clave guardada con exito!"); 

    beepSuccess(); 

    return true; 

  } else { 

    sendMessage("CODE_MISMATCH"); 

    sendMessage("Las claves no coinciden"); 

    beepError(); 

    ledError(); 

    return false; 

  } 

} 

 
 

/* Manejar comandos recibidos */ 

void handleCommand(String cmd) { 

  cmd.trim(); 

  cmd.toUpperCase(); 

 
 

  if (cmd.length() == 0) return; 

 
 

  // Comando para cerrar la caja 

  if (cmd == "#") { 

    if (!isLocked) { 

      sendMessage("Cerrando..."); 

      delay(500); 

      lock(); 

    } else { 

      sendMessage("La caja ya esta cerrada"); 

    } 

    return; 

  } 

 
 

  // Comando para cambiar la clave 

  if (cmd == "A") { 

    if (!isLocked) { 

      if (setNewCode()) { 

        sendMessage("Ahora puede cerrar la caja con #"); 

      } 

    } else { 

      sendMessage("Debe abrir la caja primero"); 

    } 

    return; 

  } 

 
 

  // Si la caja está cerrada, el comando es un intento de clave 

  if (isLocked) { 

    if (cmd.length() == CODE_LENGTH && isNumeric(cmd)) { 

      if (!hasCode()) { 

        sendMessage("NO_CODE_SET"); 

        sendMessage("No hay clave configurada"); 

        sendMessage("Use 'A' despues de abrir para crear una"); 

        return; 

      } 

 
 

      String storedCode = readCodeFromEEPROM(); 

      sendMessage("Verificando..."); 

      delay(1000); 

 
 

      if (cmd == storedCode) { 

        sendMessage("ACCESS_OK"); 

        sendMessage("Acceso concedido"); 

        unlock(); 

      } else { 

        sendMessage("ACCESS_DENIED"); 

        sendMessage("Acceso denegado"); 

        beepError(); 

        ledError(); 

      } 

    } else { 

      sendMessage("INVALID"); 

      sendMessage("Ingrese 4 digitos numericos"); 

    } 

  } else { 

    sendMessage("INFO"); 

    sendMessage("Caja abierta. Use # o A"); 

  } 

} 

 
 

/* Estado inicial */ 

void setup() { 

  Serial.begin(9600); 

 
 

  lockServo.attach(SERVO_PIN); 

 
 

  pinMode(BUZZER_PIN, OUTPUT); 

  pinMode(LED_PIN, OUTPUT); 

 
 

  digitalWrite(LED_PIN, LOW); 

  noTone(BUZZER_PIN); 

 
 

  // Sincronizar el servo con el estado guardado 

  if (hasCode()) { 

    lockServo.write(SERVO_LOCK_POS); 

    isLocked = true; 

  } else { 

    lockServo.write(SERVO_UNLOCK_POS); 

    isLocked = false; 

  } 

 
 

  delay(500); 

 
 

  // Mensaje de bienvenida 

  sendMessage("========================="); 

  sendMessage("   CAJA FUERTE IoT"); 

  sendMessage("   Silent Hill"); 

  sendMessage("========================="); 

  sendMessage("Interfaz: Monitor Serie"); 

  sendMessage(""); 

 
 

  if (isLocked) { 

    sendMessage("Estado: CERRADA"); 

    sendMessage("Ingrese la clave de 4 digitos"); 

  } else { 

    sendMessage("Estado: ABIERTA"); 

    sendMessage("No hay clave. Use 'A' para crear una"); 

  } 

 
 

  lastActivityTime = millis(); 

} 

 
 

/* Bucle principal */ 

void loop() { 

  // Leer datos entrantes desde el Monitor Serie 

  if (bluetooth.available()) { 

    char c = bluetooth.read(); 

 
 

    if (c == '\n' || c == '\r') { 

      if (inputBuffer.length() > 0) { 

        handleCommand(inputBuffer); 

        clearBuffer(); 

        lastActivityTime = millis(); 

      } 

    } else { 

      inputBuffer += c; 

 
 

      // Si ya tenemos 4 dígitos y la caja está cerrada, procesamos directamente 

      if (inputBuffer.length() >= CODE_LENGTH && isLocked && isNumeric(inputBuffer)) { 

        handleCommand(inputBuffer); 

        clearBuffer(); 

        lastActivityTime = millis(); 

      } 

    } 

  } 

 
 

  // Auto-bloqueo después de 30 segundos sin actividad 

  if (!isLocked && (millis() - lastActivityTime > AUTO_LOCK_TIMEOUT)) { 

    sendMessage("AUTO_LOCK"); 

    sendMessage("Cerrando por inactividad..."); 

    beepWarning(); 

    delay(2000); 

    lock(); 

  } 

} 
