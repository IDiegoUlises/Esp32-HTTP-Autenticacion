# Esp32 HTTP Autenticacion

<img src="https://github.com/IDiegoUlises/Esp32-HTTP-Autenticacion/blob/main/images/IMG_20230606_214023.jpg" width="400" height="800" />

* Para acceder solicita un nombre de usuario y una contraseña

### Codigo
```c++
//Carga la libreria WiFi
#include <WiFi.h>

//Agregar las credenciales Wifi
const char* ssid = "SSID";
const char* password = "PASSWORD";

//Establece la web server en el puerto 80
WiFiServer server(80);

//Variable para almacenar la solicitud HTTP
String header;

//Variables auxiliares para almacenar el estado de salida actual
String output15State = "off";
String output4State = "off";

//Asigne variables de salida a los pines GPIO
const int output15 = 15;
const int output4 = 4;

//Tiempo actual
unsigned long currentTime = millis();

//Momento anterior
unsigned long previousTime = 0;

//Defina el tiempo de espera en milisegundos (ejemplo: 2000ms = 2s)
const long timeoutTime = 2000;

//Defina la autenticacion
const char* base64Encoding = "dXN1YXJpbzpjbGF2ZQ==";  //esta en base64encoding en este caso user:clave


void setup()
{
  //Inicia el puerto serial
  Serial.begin(115200);

  //Inicializar las variables de salida como salidas
  pinMode(output15, OUTPUT);
  pinMode(output4, OUTPUT);

  //Establece los GPIO de salida como LOW
  digitalWrite(output15, LOW);
  digitalWrite(output4, LOW);

  //Conecta a la red WiFi con la SSID y el password
  Serial.print("Conectando ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }
  //Imprime la direccion IP local y inicia la web server
  Serial.println("");
  Serial.println("WiFi conectado.");
  Serial.println("IP direccion: ");
  Serial.println(WiFi.localIP());
  server.begin();
}

void loop() {
  WiFiClient client = server.available();   // Listen for incoming clients

  if (client)
  { //Si un nuevo cliente se a conectado
    currentTime = millis();
    previousTime = currentTime;
    Serial.println("New Client.");          // Imprime un mennsaje de salida en el puerto serial
    String currentLine = "";                //Hace una cadena para contener los datos entrantes del cliente

    while (client.connected() && currentTime - previousTime <= timeoutTime)  //loop while mientras el cliente esta conectado
    {
      currentTime = millis();
      if (client.available()) {             //Si hay bytes para leer del cliente
        char c = client.read();             //leer un byte, entonces
        Serial.write(c);                    //imprime en el monitor serie
        header += c;
        if (c == '\n') {                    //Si el byte es un carácter de nueva línea
          //si la línea actual está en blanco, tienes dos caracteres de nueva línea seguidos.
          //ese es el final de la solicitud HTTP del cliente, así que envíe una respuesta:
          if (currentLine.length() == 0) {
            //verifique la codificación base64 para la autenticación
            //Encontrar las credenciales correctas
            if (header.indexOf(base64Encoding) >= 0)
            {

              //Los encabezados HTTP siempre comienzan con un código de respuesta (e.g. HTTP/1.1 200 OK)
              //y un tipo de contenido para que el cliente sepa lo que viene, luego una línea en blanco:
              client.println("HTTP/1.1 200 OK");
              client.println("Content-type:text/html");
              client.println("Connection: close");
              client.println();

              //enciende y apaga los GPIO
              if (header.indexOf("GET /15/on") >= 0) {
                Serial.println("GPIO 15 on");
                output15State = "on";
                digitalWrite(output15, HIGH);
              } else if (header.indexOf("GET /15/off") >= 0) {
                Serial.println("GPIO 15 off");
                output15State = "off";
                digitalWrite(output15, LOW);
              } else if (header.indexOf("GET /4/on") >= 0) {
                Serial.println("GPIO 4 on");
                output4State = "on";
                digitalWrite(output4, HIGH);
              } else if (header.indexOf("GET /4/off") >= 0) {
                Serial.println("GPIO 4 off");
                output4State = "off";
                digitalWrite(output4, LOW);
              }

              //Mostrar la página web HTML
              client.println("<!DOCTYPE html><html>");
              client.println("<head><meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">");
              client.println("<link rel=\"icon\" href=\"data:,\">");
              //CSS para dar estilo a los botones de encendido/apagado
              //Siéntase libre de cambiar los atributos de color de fondo y tamaño de fuente para que se ajusten a sus preferencias
              client.println("<style>html { font-family: Helvetica; display: inline-block; margin: 0px auto; text-align: center;}");
              client.println(".button { background-color: #4CAF50; border: none; color: white; padding: 16px 40px;");
              client.println("text-decoration: none; font-size: 30px; margin: 2px; cursor: pointer;}");
              client.println(".button2 {background-color: #555555;}</style></head>");

              //Encabezado de página web
              client.println("<body><h1>ESP32 Web Server</h1>");

              //Muestra el estado actual y los botones ON/OFF para GPIO 15
              client.println("<p>GPIO 15 - Estado " + output15State + "</p>");

              //Si el estado de salida 15 está apagado, muestra el botón ENCENDIDO
              if (output15State == "off")
              {
                client.println("<p><a href=\"/15/on\"><button class=\"button\">ON</button></a></p>");
              }
              else
              {
                client.println("<p><a href=\"/15/off\"><button class=\"button button2\">OFF</button></a></p>");
              }

              //Muestra el estado actual y los botones ON/OFF para GPIO 4
              client.println("<p>GPIO 4 - Estado " + output4State + "</p>");

              //Si el estado de la salida 4 está apagado, muestra el botón ON
              if (output4State == "off")
              {
                client.println("<p><a href=\"/4/on\"><button class=\"button\">ON</button></a></p>");
              }
              else
              {
                client.println("<p><a href=\"/4/off\"><button class=\"button button2\">OFF</button></a></p>");
              }
              client.println("</body></html>");

              //La respuesta HTTP termina con otra línea en blanco
              client.println();
              //Salir del bucle while
              break;
            }
            else 
            {
              client.println("HTTP/1.1 401 Unauthorized");
              client.println("WWW-Authenticate: Basic realm=\"Secure\"");
              client.println("Content-Type: text/html");
              client.println();
              client.println("<html>Authentication failed</html>");
            }
          } 
          else 
          { 
            //si tiene una nueva línea, borre currentLine
            currentLine = "";
          }
        } 
        else if (c != '\r') 
        { 
          //si tiene algo más que un carácter de retorno de carro,
          currentLine += c;      //agréguelo al final de la línea actual
        }
      }
    }
    //Borrar la variable de encabezado
    header = "";
    
    //Cerrrar la conexion
    client.stop();

    //Imprime en el puerto serial
    Serial.println("Client disconnected.");
    Serial.println("");
  }
}
```
### Pagina Web al identificarse
<img src="https://github.com/IDiegoUlises/Esp32-HTTP-Autenticacion/blob/main/images/IMG_20230606_215405.jpg" width="500" height="700" />

### Para el usuario y la clave debe estar codificado en base64
<img src="https://github.com/IDiegoUlises/Esp32-HTTP-Autenticacion/blob/main/images/IMG_20230606_213933.jpg" width="450" height="800" />

