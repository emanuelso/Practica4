César Roldan Moros
Grupo 13

# PRÁCTICA 4: Sistemas operativos en tiempo real
___
##### Objetivo 
El objetivo de la práctica es comprender el funcionamiento de un sistema operativo en tiempo real. Para ello generaremos varias tareas simultáneamente y veremos cómo se ejecutan dividiendo el tiempo de uso de la CPU.
___
##### Creando tareas en el ESP32 con Arduino i FreeRTOS

###### Parpadear un LED encendido y apagado continuamente
Primero definimos el PIN al que el LED está conectado y establecemos el modo OUTPUT:
```
const int led1 = 2; // Pin of the LED
void setup(){
pinMode(led1, OUTPUT);
}
``` 
Ahora creamos la función principal. Utilizamos digitalWrite() para encender y apagar el LED y también vTaskDelay para pausar la tarea 500ms entre ON y OFF:
```
void toggleLED(void * parameter){
  for(;;){ // infinite loop
    digitalWrite(led1, HIGH); // Turn the LED on
    vTaskDelay(500 / portTICK_PERIOD_MS); // Turn the LED off
    digitalWrite(led1, LOW); // Pause the task again for 500ms
    vTaskDelay(500 / portTICK_PERIOD_MS);
  }
}
```
Por último, debemos informar al planificador de la tarea. En el setup() hacemos:
```
void setup() {
  xTaskCreate(
  toggleLED, // Function that should be called
  "Toggle LED", // Name of the task (for debugging)
  1000, // Stack size (bytes)
  NULL, // Parameter to pass
  1, // Task priority
  NULL // Task handle
  );
}
```
##### Parar tareas
```
void anotherTask(void * parameter){
// Kill task1 if it's running
  if(toggleLED != NULL) {
    vTaskDelete(NULL);
  }
}
```
___
### Primer ejercicio
##### Codigo
```
#include <Arduino.h>
void anotherTask( void * parameter ){
  for(;;){
    Serial.println("this is another Task");
    delay(1000);
  }  
}
void setup()
{
  Serial.begin(112500);
  xTaskCreate(
  anotherTask, 
  "another Task", 
  10000, 
  NULL, 
  1, 
  NULL); 
}
void loop()
{
  Serial.println("this is ESP32 Task");
  delay(1000);
}
```
##### Funcionamiento
Primero creamos la función de otra tarea donde es un bucle que va imprimiendo por pantalla cada segundo "This ins antother task".
En el void setup() inicializamos el monitor. A continuación le informamos sobre la tarea con un xTaskCreate donde le pasamos:
-Nombre de función de la tarea a llamar (anotherTask)
-Nombre de la tarea ("another Task")
-Tamaño de la tarea (1000 bytes)
-Parametro a pasar (NULL)
-Prioridad de la tarea (1)
-El handle (puntero) de la tarea (NULL)
En el loop hacemos la primera función (tarea) que le decimos que por pantalla imprima "This is ESP32 task"
Por tanto una vez ejecutado el programa deberá ejecutar las dos tareas a la vez
##### Salida
Una vez ejecutado el programa, vemos que se ejecutan las dos tareas a la vez y va imprimiendo por pantalla los dos mensajes.
![](sortida_ejer1.png)

___
### Segundo ejercicio
Hacer un programa que utilice dos tareas: una que encienda un LED y la otra que lo apague, es decir, hacer un semáforo.
##### Codigo
```
#include <Arduino.h>
long debouncing_time = 150; 
volatile unsigned long last_micros;
 
SemaphoreHandle_t interruptleds;
 void TaskLed(void * pvParameters)
{
  (void) pvParameters;
  pinMode(8, OUTPUT);
  for (;;) {
    if (xSemaphoreTake(interruptleds, portMAX_DELAY) == pdPASS) {
      digitalWrite(8, !digitalRead(8));
    }
  }
}
void TaskBlink(void *pvParameters)
{
  (void) pvParameters;
  pinMode(7, OUTPUT);
  for (;;) {
      digitalWrite(7, HIGH);
      vTaskDelay(200 / portTICK_PERIOD_MS);
      digitalWrite(7, LOW);
      vTaskDelay(200 / portTICK_PERIOD_MS);
  }
}
void interruptHandler() {
  xSemaphoreGiveFromISR(interruptleds, NULL);
}
void debounceInterrupt() {
  if((long)(micros() - last_micros) >= debouncing_time * 1000) {
    interruptHandler();
    last_micros = micros();
  }
}
void setup() {
  pinMode(2, INPUT_PULLUP);
  xTaskCreate(TaskLed,  "Led", 128, NULL, 0, NULL );
  xTaskCreate(TaskBlink,  "LedBlink", 128, NULL, 0, NULL );
  interruptleds = xSemaphoreCreateBinary();
  if (interruptleds != NULL) {
    attachInterrupt(digitalPinToInterrupt(2), debounceInterrupt, LOW);
  }
}
 
void loop() {}
```