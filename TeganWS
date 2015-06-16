/*

Tegan Web Server (TeganWS) for Intel Edison
TeganWS v0.8


This sketch provides basic web server capabilities for the Intel Edison board
that provides browser access and control of your Edison. Basic features include:

* Ability to serve HTML files from SD Card storage
* Support for HTML Page Methods that links code with specific HTML pages
* Support for Substitution Tags to easily include server generated data in web pages
* HTML file upload from your browser enabling client editing of web pages
* Time-stamped log file of all HTTP requests and errors
* Debug mode displaying all incoming and outgoing HTTP packets to the serial port.
* Tested on Google Chrome and MS IE.

TeganWS is useful for:

* Displaying and controlling the I/O (digital I/O, analog I/O, PWM) of your Edison.
* Learning and testing simple HTML programming.
* Providing your own custom web interface to your Edison application.
* Running a simple web server.

For detailed help and tutorial please use the included HTML files either
from the home page on a running TWS or from the stand-alone files with a
web browser.


This example is written for a network using WPA encryption. For WEP or WPA,
change the Wifi.begin() call accordingly.


TeganWS for the Intel Edison board was written with and requires Arduino v1.6.0.

 Circuit:
 * Intel Edison with Arduino breakout board (built in WiFi and SD Card)
 * SD card attached with the following files in root:
 - index.html
 - iocontrol.html
 - environmental.html
 - fileupload.html
 - playground.html
 - about.html
 OPTIONAL: (Grove sensor kit) Temperature sensor connected to A0
 OPTIONAL: (Grove sensor kit) Buzzer connected to D4

Created June 2015 by Steve Engelbrecht.
Many thanks to Tom Igoe and his WiFi Web Server examples.
*/

#include <SPI.h>
#include <WiFi.h>
#include <SD.h>

#define pinTemp A0
#define pinBuzzer 4
#define LED 13
#define MAXFILENAMELENGTH 30 // max size of a file name including extension
#define MAXHTMLLINESIZE 150 // max size of a line (inc \r and \n) in an HTML file plus 1
#define MAXHTTPHEADERSIZE 200 // max size of an HTTP header packet
#define MAXSUBCOL 40 // max string size in substitute array
#define DISPLAYLOGSIZE 2500 // number of characters in log file to display
#define TWSIPPORT 3232  // port number of IP address that TWS listens to; on Edison 80 is used by default
#define DEBUG_INPUT_PACKETS false
#define DEBUG_OUTPUT_PACKETS false



char ssid[] = "night";                // your network SSID (name)
char pass[] = "thecrowsflyatnight";   // your network password
int keyIndex = 0;                     // your network key Index number (needed only for WEP)
char my_ip_address[30];               // will be populated with IP address, null terminated
int status = WL_IDLE_STATUS;
unsigned long boot_time;
WiFiClient client;
WiFiServer server(TWSIPPORT); // On Edison skip port 80 which is being used by linux
//unsigned int socket_in_count;
char substitute_data[5][MAXSUBCOL];

// Define the B-value of the thermistor.
// This value is a property of the thermistor used in the Grove - Temperature Sensor,
// and used to convert from the analog value it measures and a temperature value.
const int B = 3975;

struct DPIN_CONFIG {
  byte direction;
  int value;
} DPIN[30];
//byte DPIN[30]; // 0: Not Used; 1:Not initialized; 2:Input; 3:Output



void setup() {
  int i, j;

  Serial.begin(9600);      // initialize serial communication
  pinMode(LED, OUTPUT);      // set the LED pin mode
  pinMode(pinBuzzer, OUTPUT);
  // 60 second delay for linux to boot and wifi to connect
  for (j = 10; j > 0; j--) {
    for (i = 0; i < 6; i++) {
      digitalWrite(LED, HIGH);               // turns the LED on
      delay(j * 100);
      digitalWrite(LED, LOW);               // turns the LED off
      delay((10 - j) * 100);
    }
  }
  // check for the presence of the shield:
  if (WiFi.status() == WL_NO_SHIELD) {
    Serial.println("WiFi shield not present");
    while (true);       // don't continue
  }

  String fv = WiFi.firmwareVersion();
  Serial.print("WiFi firmware version is v");
  Serial.println(fv);
  if (fv != "1.1.0") Serial.println("Please upgrade the firmware");

  // attempt to connect to Wifi network:
  while (status != WL_CONNECTED) {
    Serial.print("Attempting to connect to Network named: ");
    Serial.println(ssid);                   // print the network name (SSID);
    filelog(1, 0, "Attempting to connect to network");
    filelog(1, 0, ssid);

    // Connect to WPA/WPA2 network. Change this line if using open or WEP network:
    status = WiFi.begin(ssid, pass);
    // wait 10 seconds for connection:
    delay(10000);
    for (i = 0; i < 5; i++) {
      digitalWrite(LED, HIGH);               // turns the LED on
      delay(250);
      digitalWrite(LED, LOW);               // turns the LED off
      delay(250);
    } // for
  }
  buzzer();
  delay(100);
  buzzer();
  Serial.print("Initializing SD card...");
  // On the Ethernet Shield, CS is pin 4. It's set as an output by default.
  // Note that even if it's not used as the CS pin, the hardware SS pin
  // (10 on most Arduino boards, 53 on the Mega) must be left as an output
  // or the SD library functions will not work.
  pinMode(10, OUTPUT);
  if (!SD.begin(4)) {
    Serial.println("****SD card initialization failed!");
    buzzer;
    buzzer;
    return;
  }
  filelog(1, 0, "Edison Booted");
  Serial.println("SD card initialization done.");
  server.begin();                           // start the web server on port TWSIPPORT
  printWifiStatus();                        // you're connected now, so print out the status
  boot_time = millis();
  init_DPIN();
  return;
}




void loop() {

  http_request();

}




/*

This procedure initializes the DPIN structure.

*/

void init_DPIN() {
  int i;
  // 0: Not Used; 1:Not initialized; 2:Input; 3:Output
  for (i = 0; i < sizeof(DPIN) / sizeof(DPIN[0]); i++) {
    DPIN[i].direction = 1;
    DPIN[i].value = 0xFFFF;
  }
  DPIN[13].direction = 3; // Pin 13 (LED) was configured in setup
  DPIN[13].value = 0; // starts off
  DPIN[0].direction = 0; // Not available
  DPIN[2].direction = 0; // Not available
  return;
}




void printWifiStatus() {
  int i, j;
  // print the SSID of the network you're attached to:
  Serial.print("SSID: ");
  Serial.println(WiFi.SSID());

  IPAddress ip = WiFi.localIP(); // Get WiFi IP address
  long rssi = WiFi.RSSI(); // Get WiFi signal strength

  char c_string[50];
  for (i = 0; i < 50; i++) my_ip_address[i] = ' ';
  my_ip_address[49] = '\0';
  strncpy(c_string, "TWS Server Address: ", 20);
  i = 20;
  j = sprintf(&my_ip_address[i], "%-d", ip[0]);
  if ((j < 0) || (j > 3)) {
    filelog(0, 1, "Error logging IP address");
    return;
  }
  i += j;
  my_ip_address[i++] = '.';
  j = sprintf(&my_ip_address[i], "%-d", ip[1]);
  if ((j < 0) || (j > 3)) {
    filelog(0, 2, "Error logging IP address");
    return;
  }
  i += j;
  my_ip_address[i++] = '.';
  j = sprintf(&my_ip_address[i], "%-d", ip[2]);
  if ((j < 0) || (j > 3)) {
    filelog(0, 3, "Error logging IP address");
    return;
  }
  i += j;
  my_ip_address[i++] = '.';
  j = sprintf(&my_ip_address[i], "%-d", ip[3]);
  if ((j < 0) || (j > 3)) {
    filelog(0, 4, "Error logging IP address");
    return;
  }
  i += j;
  my_ip_address[i++] = ':';
  //strncpy(&my_ip_address[i], "3232", 4);
  j = sprintf(&my_ip_address[i], "%-d", TWSIPPORT);
  if ((j < 0) || (j > 5)) {
    filelog(0, 5, "Error logging IP address");
    return;
  }
  i += j;
  my_ip_address[i] = '\0';
  filelog(1, 0, my_ip_address); // log the IP address
  // get the signal strength
  for (i = 0; i < 50; i++) c_string[i] = ' ';
  c_string[49] = '\0';
  strncpy(c_string, "WiFi Signal Strength (RSSI): ", 29);
  i = 29;
  j = sprintf(&c_string[i], "%-d", rssi);
  if ((j < 0) || (j > 5)) {
    filelog(0, j, "Error logging WiFi Signal Strength");
    return;
  }
  i += j;
  strncpy(&c_string[i], " dBm", 4);
  filelog(1, 0, c_string); // log the IP address


  return;
}



int minutes_since_boot() {
  return ((millis() / 1000) / 60);
}




/*

This routine reads the selected analog input port. It is assumed the port is properly initialized.

Parameters:

analog_select: Specifies the analog port to read. 0 = "A0", 1 = "A1", ...

Returns: The analog reading


*/

int AREAD(int analog_select) {
  int i;

  switch (analog_select) {
    case (0):
      i = analogRead(A0);
      break;
    case (1):
      i = analogRead(A1);
      break;
    case (2):
      i = analogRead(A2);
      break;
    case (3):
      i = analogRead(A3);
      break;
    default:
      filelog(0, analog_select, "Bad analog read channel select.");
      i = -1;
  } // switch
  return (i);
}


/*

This routine reads the selected digital input port. It is assumed the port is properly initialized.

Parameters:

analog_select: Specifies the digital port to read. 0, 1, 2...

Returns: The analog reading


*/

int DREAD(int digital_select) {
  int i;

  return(digitalRead(digital_select));
  switch (digital_select) {
    case (0):
      i = digitalRead(0);
      break;
    case (1):
      i = digitalRead(1);
      break;
    case (2):
      i = digitalRead(2);
      break;
    case (3):
      i = digitalRead(3);
      break;
    default:
      filelog(0, digital_select, "Bad digital read channel select.");
      i = -1;
  } // switch
  return (i);
}




/*

This routine writes to the selected digital input port. It is assumed the port is properly initialized.

Parameters:

analog_select: Specifies the digital port to write to. 0, 1, 2...

high_low: 1 = HIGH, 0 = LOW

*/

void DWRITE(int digital_select, int high_low) {
  int i;

  if (high_low) digitalWrite(digital_select, HIGH); else digitalWrite(digital_select, LOW);
  DPIN[digital_select].value = high_low;

  return;
}












/*


fs: Selects file to write to: 0 = httperror.txt; 1 = httplog.txt
code: optional user code to record in httperror.txt
msg[]: null terminated string to write to file


*/

void filelog(int fs, int code, char msg[]) {
  File logfile;
  unsigned long logsize;
  int i;

  switch (fs) {
    case 0:
      logfile = SD.open("httperror.txt", FILE_WRITE);
      break;
    case 1:
      logfile = SD.open("httplog.txt", FILE_WRITE);
      break;
    default:
      Serial.println("Error selecting log file");
      return;
  }
  if (logfile) {
    // seek to end of file...
    logsize = logfile.size();
    if (logsize > 2000000) { // if over 2 meg delete it and start over
      switch (fs) {
        case 0:
          logfile.close();
          SD.remove("httperror.txt");
          logfile = SD.open("httperror.txt", FILE_WRITE);
          logsize = logfile.size();
          break;
        case 1:
          logfile.close();
          SD.remove("httplog.txt");
          logfile = SD.open("httplog.txt", FILE_WRITE);
          logsize = logfile.size();
          break;
      }
    }
    logfile.seek(logsize);
    logfile.print("@");
    logfile.print(minutes_since_boot());
    logfile.print(" min; ");
    if (fs == 0) {
      logfile.print("CODE= ");
      logfile.print(code);
      logfile.print("; ");
    }
    i = 0;
    while (msg[i] != '\0') logfile.print(msg[i++]); // print msg terminated with null character
    // if there's no '\r\n' at the end of the string then add it...
    if ((msg[i - 2] != '\n') && (msg[i - 1] != '\n')) logfile.println();
    //if (fs == 0) logfile.println();
    logfile.close(); // close the file:
  } else {
    Serial.println("Error opening <logfile>.txt"); //if the file didn't open, print an error
  }
  return;
}




void buzzer () {
  digitalWrite(pinBuzzer, HIGH);
  delay(1);
  digitalWrite(pinBuzzer, LOW);
}



char socket_read() {
  char c;

  //client.available()
  c = client.read();
  //  socket_in_count++;
  if (DEBUG_INPUT_PACKETS) {
    //Serial.print("[");
    //Serial.print(socket_in_count);
    //Serial.print("]");
    if ((c < ' ') || (c > '~')) { // if not printable
      Serial.print(".");
      Serial.print("<");
      Serial.print(c, DEC);
      Serial.print(">");
      if (c == '\r') Serial.print(c);
      if (c == '\n') Serial.print(c);
    } else {
      Serial.print(".");
      Serial.print(c);
    } // else
  }
  return (c);
}



void socket_print(char c[]) {

  //Serial.println(client.connected());
  if (DEBUG_OUTPUT_PACKETS) {
    Serial.print("~-");
    Serial.print(c);
    Serial.print("-~");
  }
  client.print(c);
  return;
}


void socket_println(char c[]) {

  //Serial.println(client.connected());
  if (c == NULL) {
    if (DEBUG_OUTPUT_PACKETS) Serial.println("~-<CR><LF>-~");
    client.println();
    return;
  }
  if (DEBUG_OUTPUT_PACKETS) {
    Serial.print("~-");
    Serial.print(c);
    Serial.println("<CR><LF>-~");
  }
  client.println(c);
  return;
}



void socket_write(char c) {

  if (DEBUG_OUTPUT_PACKETS) {
    Serial.print('~');
    Serial.print(c, DEC);
  }
  client.print(c);
  return;
}



void socket_writenum(int i) {

  if (DEBUG_OUTPUT_PACKETS) {
    Serial.print('~');
    Serial.print(i);
  }
  client.print(i);
  return;
}





/*

This routine reads and discards data from the socket connections until there is no more data to read.

*/

void socket_read_to_end () {
  char c;

  while (client.available()) {
    c = socket_read();
  }
  return;
}





/*

This routine reads data from the socket connections into a buffer until a newline ('\n') character is
received or if there is no more data to read. A null character ('\0') is appended to the end.

Parameters:

buffer: pointer to a character array
buffer_length: size of buffer in bytes, must be 2 or more bytes in size for \n and \0.

Returns:

0: Success, a line (with possibly only '\n') ending with an added null character was stored in the buffer
1: Success however the line exceeded the buffer length. Extra characters were discarded however the buffer
does end with a newline and null character.
2: No newline character found before end of data. Buffer contents are undefined.
3: Supplied buffer length less than 2;

*/

int socket_read_line(char bufferarray[], int buffer_length) {
  char c;
  int i;
  boolean overflow;

  if (buffer_length < 2) return (3);
  overflow = false;
  c = ' ';
  i = 0;
  while ((client.available()) && (c != '\n')) { // read until a newline character
    if (i == (buffer_length - 1)) {
      i--;
      overflow = true;
    } // if
    c = socket_read();
    if (!overflow) bufferarray[i++] = c;
  } // while
  //Serial.print("LINE: Last character of line (in DEC) is =");
  //Serial.println(bufferarray[i-1], DEC);

  if (overflow) { // if overflow then append a newline and null character
    bufferarray[buffer_length - 2] = '\n';
    bufferarray[buffer_length - 1] = '\0';
  } else bufferarray[i] = '\0';

  if (c == '\n') { // we got a newline
    if (overflow) return (1); else return (0);
  }
  return (2); // never got a newline character and no more bytes to read
}







/*



Returns:

-1 : Substitute code ("<%S") not found
0 - N : Index in buffer where a valid Substitute code was found. A valid code includes
a '>')

*/

int sub_search(char buf[], int start) {
  int i, j;

  j = 0;
  while (buf[j] != '\n') j++; // find the end of the buffer
  while ((buf[start] != '<') && (start < j)) {
    start++;
  }
  if (start >= j) return (-1); // did not find opening bracket
  if (buf[start + 1] != '%') return (-1);
  if (buf[start + 2] != 'S') return (-1);
  i = start; // save index for '<'
  start += 3;
  while ((buf[start] != '>') && (start < j)) start++;
  if (start >= j) return (-1); // never found the closing bracket
  return (i);
}



int save_get_sub_number(char fb[]) { // <%SPDIPx..x> or <%SLx..x>
  char buf[15];
  int j, k;
  boolean bad_number;

  bad_number = false;
  j = 0;
  while (((fb[j] < '0') || (fb[j] > '9')) && (fb[j] != '>')) {
    j++;
  }
  if (fb[j] == '>') {
    bad_number = true;
  } else
  {
    bad_number = false;
  }
  k = 0;
  while (fb[j] != '>') {
    buf[k] = fb[j];
    if ((buf[k] < '0') || (buf[k] > '9')) bad_number = true;
    j++;
    k++;
  }
  buf[k] = '\0';
  if (bad_number) {
    return (0xFFFF);
  }
  else {
    return (atoi(buf));
  }
  return (0);
}



int get_sub_number(char fb[]) { // <%SPDIPx..x> or <%SLx..x>
  char buf[15];
  int j, k;
  boolean bad_number;

  bad_number = false;
  j = 0;
  // skip over all non-digits
  while (((fb[j] < '0') || (fb[j] > '9')) && (fb[j] != '>') && (j <= 5)) {
    j++;
  }
  if ((fb[j] == '>') || (j > 5)) {
    return (0xFFFF);
  }
  k = 0;
  while ((fb[j] != '>') && (k < 5)) { // read digits (not more than 5)
    buf[k] = fb[j];
    if ((buf[k] < '0') || (buf[k] > '9')) return (0xFFFF);
    j++;
    k++;
  }
  if (k == 5) return (0xFFFF);
  buf[k] = '\0';
  return (atoi(buf));
}







/*

This routine searchs for two end-of-line terminators in a row, either: "\n\n" or "\r\n\r\n".

This assumes the last character read from the socket connection was a '\n'. HTTP packets are
terminated with two "\n" OR two "\r\n" in a row.

Returns:

0: Packet properly terminated
1: Packet improperly terminated

*/

int http_packet_read_to_end() {
  char c;
  int i;
  boolean found;
  char line[3];

  //Serial.println("http_packet_read_to_end");
  line[0] = '\n';
  if (client.available()) {
    line[1] = socket_read();
  } else return (1);
  if (line[1] == '\n') return (0); // '\n\n' in a row, proper exit
  if (client.available()) {
    line[2] = socket_read();
  } else return (1);
  if ((line[1] == '\r') && (line[2] == '\n')) return (0); // a "\n\r\n" found, proper exit
  found = false;
  while ((!found) && client.available()) {
    line[0] = line[1];
    line[1] = line[2];
    line[2] = socket_read();
    if ((line[1] == '\n') && (line[2] == '\n')) found = true;
    if ((line[0] == '\n') && (line[1] == '\r') && (line[2] == '\n')) found = true;
  }
  if (found) {
    return (0);
  } else {
    filelog(0, 0, "Never found proper HTTP end of packet");
    return (1);
  }
}










void stream_binary_file(char filename[], char filetype[]) {
  File BINARYFile;
  char c, c_string[12];
  int file_length, i;

  BINARYFile = SD.open(filename, FILE_READ);
  if (BINARYFile) {
    file_length = BINARYFile.size();
    sprintf(c_string, "%d", file_length);
    socket_println("HTTP/1.1 200 OK");
    socket_print("Content-Type: ");
    socket_println(filetype);
    socket_print("Content-Length: ");
    socket_println(c_string);
    //client.println();
    socket_println(NULL);
    i = 0;
    while (BINARYFile.available()) {
      //client.write(BINARYFile.read());
      c = BINARYFile.read();
      socket_write(c);
    }
    BINARYFile.close();
  } else
  {
    filelog(0, 0, "Error opening .ICO file");
    HTTP_error_404();
  }
  return;

}



/*

This routine accepts a specified HTML filename and replaces substitution codes with A)a set of
substitution data supplied by the caller (for <%SL> codes) or I/O Pin data (for <%SP> codes). The
file is then streamed out to the socket.

Substitution Codes have the following format:

<%SPDIPxx> or <%SPAIPxx>
SP: Substitute Pin reading
DI: Digital Input; P: Pin; xx = input pin number
AI: Analog Input; P: Pin; xx = input pin number
<%SLxx>
SL: Substitute List; xx = row index into the passed substitution list



Parameters:

filename: Null-terminated name (including extension) of HTML file available on the filesystem

sub_data[MAXSUBROWS][MAXSUBCOL]: pointer to a 2 dimensional array of characters containing ASCII strings
and are null termination. The last array element is terminated with a null character. Example format:

sub_list[0][0 - 1] = 'A', '\0'
sub_list[1][0 - 4] = 'T', 'e', 's', 't', '\0'
sub_list[2][0] = '\0'


*/

void stream_html_file(char filename[]) {
  File HTMLFile;
  int buffer_index, i, j, k, sub_type;
  int max_sub_index;
  char file_buffer[MAXHTMLLINESIZE];
  char c;
  boolean last_char_newline;

  max_sub_index = sizeof(substitute_data) / sizeof(substitute_data[0]) - 1;
  // null terminate each string just in case...
  for (i = 0; i <= max_sub_index; i++) substitute_data[i][sizeof(substitute_data[0]) - 1] = '\0';
  HTMLFile = SD.open(filename, FILE_READ);
  if (HTMLFile) {
    // send HTTP response
    socket_println("HTTP/1.1 200 OK");
    socket_println("Content-Type:Text/html");
    socket_println(NULL);

    while (HTMLFile.available()) { // until EOF
      // read a line
      buffer_index = 0;
      file_buffer[buffer_index++] = HTMLFile.read();

      while ((HTMLFile.available()) && (buffer_index < MAXHTMLLINESIZE) &&
             (file_buffer[buffer_index - 1] != '\n')) {
        file_buffer[buffer_index++] = HTMLFile.read();
      } // while
      if (buffer_index >= MAXHTMLLINESIZE) { // we hit the max line length before the end of line
        filelog(0, buffer_index, "Error HTML file line length too long.");
        return;
      }

      //We now have a complete line. Note that if it is the very last line it may end with
      // a '\n'. We now have a line to look at, search for sub code (<%SL or <%SP)
      buffer_index--; // points to the last character of the line
      if (file_buffer[buffer_index] == '\n') last_char_newline = true; else last_char_newline = false;
      i = 0;
      while (i <= buffer_index) {
        // look for the start of a tag
        while ((i <= buffer_index) && (file_buffer[i] != '<')) {
          socket_write(file_buffer[i]);
          i++;
        } // while

        // either EOL or we found a '<', search for '>' before EOL
        if ((file_buffer[i] == '<') && (i <= buffer_index)) {
          j = i;
          while ((file_buffer[j] != '>') && (j <= buffer_index)) j++; // search for '>'
        }
        // if we have a pair of brackets and we have not written out the entire line...
        if ((file_buffer[i] == '<') && (file_buffer[j] == '>') && (i <= buffer_index)) {
          sub_type = 0;
          if (file_buffer[i + 3] == 'P') sub_type = 1;
          if (file_buffer[i + 3] == 'L') sub_type = 2;
          if ((file_buffer[i + 1] == '%') && (file_buffer[i + 2] == 'S') && (sub_type > 0)) {
            // we have a substitution code!
            //
            switch (sub_type) { // either <%SP or <%SL
              case 1: // Substitute PIN <%SP
                k = get_sub_number(&file_buffer[i + 7]); // <%SPDIPxx> or <%SPAIPxx>
                if (k == 0xFFFF) {
                  // error
                  //Serial.println("Bad PIN number in substitute code");
                  filelog(0, k, "Bad PIN number in substitute code in HTML file.");
                  socket_write('#');
                  socket_write('#');
                  socket_write('#');
                  i = j + 1; // skip over entire substitute code
                } else
                { // k is the analog or digital pin being specified.
                  // Digital I/O: Digital pins 0 through 13 and Analog pins A0 through A5 can be used
                  // as a digital input or output, using pinMode(), digitalWrite(), and digitalRead() functions.
                  if ((file_buffer[i + 4] == 'A') && (file_buffer[i + 5] == 'I')) {
                    switch (k) {
                      case 0:
                        //client.print(analogRead(A0));
                        client.print(AREAD(0));
                        break;
                      case 1:
                        client.print(AREAD(1));
                        break;
                      case 2:
                        client.print(AREAD(2));
                        break;
                      case 3:
                        client.print(AREAD(3));
                        break;
                      default:
                        socket_print("_INVALID ANALOG PIN #_");
                        break;
                    } //switch
                  } // if file_buffer[i + 4] == 'A'
                  else {
                    if ((file_buffer[i + 4] == 'D') && (file_buffer[i + 5] == 'I')) {
                      switch (k) {
                        case 0:
                          //client.print(digitalRead(0));
                          client.print(DREAD(0));
                          break;
                        case 1:
                          client.print(DREAD(1));
                          break;
                        case 2:
                          client.print(DREAD(2));
                          break;
                        case 3:
                          client.print(DREAD(3));
                          break;
                        default:
                          socket_print("_INVALID DIGITAL PIN #_");
                          break;
                      } //switch
                    } else { // "<%SP" was not followed by a 'A' or "DI", check for "DO"
                      /*
                      if ((file_buffer[i + 4] == 'D') && (file_buffer[i + 5] == 'O')) {
                        // parse the
                      }
                      */
                    } // else
                  } //else file_buffer[i + 4] == 'A'
                  i = j + 1; // else <%SP not followed by a 'A' or a 'D'
                } // else (k == 0xFFFF)
                break;
              case 2: // Substitute LIST <%SL
                //Serial.println("<%SL code found");
                k = get_sub_number(&file_buffer[i + 4]); // <%SLxx>
                //Serial.print("<%SL index =");
                //Serial.println(k);
                if ((k == 0xFFFF) || (k > max_sub_index)) {
                  // error
                  //Serial.println("Bad PIN number in substitute code");
                  filelog(0, k, "Bad LIST number in substitute code in HTML file.");
                  socket_write('#');
                  socket_write('#');
                  socket_write('#');
                  i = j + 1; // skip over entire substitute code
                } else
                {
                  socket_print(substitute_data[k]);
                  //Serial.println(substitute_data[k]);
                  i = j + 1;
                }
                break;
            } // switch sub_type, PIN number k is valid
          } else { // "<" and ">" found but it is not a substitution tag, continue on
            socket_write(file_buffer[i++]);
          }
        } else { //else no matching closing bracket or we are past the end of the line
          if ((i <= buffer_index)) socket_write(file_buffer[i++]);
        }
      } // while i <= buffer_index (there's still characters left in the line to output)
      buffer_index = 0;
      file_buffer[buffer_index] = '\0';
    } // while HTMLFile.available()
    // end with <CR><LF><CR><LF>
    socket_println(NULL);
    if (!last_char_newline) socket_println(NULL);
    HTMLFile.close(); // close file

  } else
  {
    filelog(0, 0, "Error opening .HTML file");
    HTTP_error_404();
  } // else
  return;

}






/*

Parameters:

URL: pointer to character array containing the URL starting with the character
after "GET" (which should be a space) and which is terminated with a '\n'. This
includes the passed name/value pairs.


*/

void http_verb_get(char URL[]) {
  int i, j, k;
  char filename[MAXFILENAMELENGTH + 1];
  char param[MAXHTTPHEADERSIZE];
  const char *htmllist[7];

  // This is the list of HTML pages that have an associated method
  htmllist[0] = "favicon.ico";
  htmllist[1] = "playground.html";
  htmllist[2] = "environmental.html";
  htmllist[3] = "syslogfile.html";
  htmllist[4] = "iocontrol.html";
  htmllist[5] = "iocontrol2.html";
  htmllist[6] = NULL; // last one MUST be NULL
  //find the '/' then every character after that until a ' ' or '?'
  i = 0;
  while ((URL[i] != '/') && (i < 10)) i++; // skip spaces after GET leading to /
  // if i == 10 then respond with HTTP ERROR PAGE
  i++; // start of filename
  filename[0] = '\0';
  j = 0;
  // everything between the '/' and a ' ' or '?' is the filename
  while ((URL[i] != ' ') && (URL[i] != '?') && (j < MAXFILENAMELENGTH)) {
    filename[j++] = URL[i++];
  }
  if (j >= MAXFILENAMELENGTH) {
    filelog(0, j, "Bad GET URL filename length.");
    HTTP_error_404(); // then respond with HTTP ERROR PAGE
    return;
  }
  filename[j] = '\0';
  if (filename[0] == '\0') { //special case; file name was a '/' which refers to index.html
    strcpy(filename, "index.html");
  }

  param[0] = '\0';
  if (URL[i] == '?') { // we have parameters
    i++; // beginning of parameters
    j = 0;
    while ((URL[i] != ' ') && (j < (MAXHTTPHEADERSIZE - 3))) {
      param[j++] = URL[i++];
    }
    if (j > (MAXHTTPHEADERSIZE - 3)) {
      filelog(0, j, "Bad GET URL parameter length.");
      HTTP_error_404();
      // then respond with HTTP ERROR PAGE
      return;
    }
    param[j] = '\0'; // make param a valid string
    if (DEBUG_INPUT_PACKETS) {
      Serial.print("PASSED PARAMETERS: ");
      Serial.println(param);
    }
  } else if (DEBUG_INPUT_PACKETS) Serial.println("No PASSED PARAMETERS");
  // We have a filename and also a passed parameter
  // determine if the page has a method associated with it
  // compare file name with list of file names
  // if match, call method and then return
  k = -1;
  i = 0;
  while (htmllist[i] != NULL) {
    if (strcmp(filename, htmllist[i]) == 0) k = i;
    i++;
  } // while
  if (k > -1) {
    // there is a method associated with the html page...
    // pass the rest of the HTTP header starting with '?'.
    switch (k) {
      case 0: // favicon.ico
        faviconico();
        break;
      case 1: // playground.html
        playgroundhtml();
        break;
      case 2: //environmental.html
        environmentalhtml();
        break;
      case 3: //syslogfile.html
        sys_logfile_page();
        break;
      case 4: //iocontrol.html
        iocontrolhtml(param);
        break;
      case 5: //iocontrol2.html
        iocontrol2html(param);
        break;
      default:
        filelog(0, k, "Error indexing through file names.");
        break;
    } // switch
  }
  else {
    // if the HTML page has no method then open the file and stream back thru socket
    // the first line of the HTTP packet has been read, read the rest of the HTTP packet
    http_packet_read_to_end();
    //stream_html_file(filename, NULL);
    stream_html_file(filename);
  }
  return;
}




/*

POST /form-name HTTP:1.1


Parameters:

URL: pointer to character array containing the URL starting with the character
after "POST" (which should be a space) and which is terminated with a '\n'. This
includes the passed name/value pairs.


*/

void http_verb_post(char URL[]) {
  int i, j;
  boolean header_end;
  String s1, s2, boundary, filename;
  char c[MAXHTMLLINESIZE], fn[MAXFILENAMELENGTH], *s;
  char ring_buffer[50], fupload;
  int ring_head, ring_tail, ring_size, s1_ptr;
  File Fileupload;


  // The next part of the HTTP message contains various descriptors such as Host, Content-Type,
  // Content-Length, boundary=xxxxx<cr><lf>, Cache-Control and ends with <CR><LF><CR><LF>
  //
  // read the next line
  i = socket_read_line(c, sizeof(c));
  if (i > 0) {
    s = "POST - Bad packet read during boundary search.";
    filelog(0, i, s);
    HTTP_error_400(s);
    return;
  }
  s2 = String(c);
  i = s2.indexOf('\n');
  if ((i == 0) || (i == 1)) { // blank line
    s = "POST - Empty packet during boundary search.";
    filelog(0, i, s);
    HTTP_error_400(s);
    return;
  }
  boundary = "";
  header_end = false;
  while (!header_end) {
    // Search for boundary string
    i = s2.indexOf("boundary=");
    if (i >= 0) { // found it
      j = s2.indexOf('\r', i);
      boundary = s2.substring(i + 9, j);
    }
    // read the next line
    i = socket_read_line(c, sizeof(c));
    if (i > 0) {
      s = "POST - Bad packet read during boundary search.";
      filelog(0, i, s);
      HTTP_error_400(s);
      return;
    }
    s2 = String(c);
    i = s2.indexOf('\n');
    if ((i == 0) || (i == 1)) { // blank line
      header_end = true;
    }
  } // while
  if (boundary == "") {
    s = "POST - Boundary delimiter not found.";
    filelog(0, i, s);
    HTTP_error_400(s);
    return;
  }

  // The next part of the message begins with the string specified by "boundary=" and contains
  // descriptors such as Content-Disposition, filename=xxxxx, Content-Type and ends
  // with <CR><LF><CR><LF>
  // read the next line
  i = socket_read_line(c, sizeof(c));
  if (i > 0) {
    s = "POST - Bad packet read during filename search.";
    filelog(0, i, s);
    HTTP_error_400(s);
    return;
  }
  s2 = String(c);
  i = s2.indexOf('\n');
  if ((i == 0) || (i == 1)) { // blank line
    s = "POST - Empty packet during filename search.";
    filelog(0, i, s);
    HTTP_error_400(s);
    return;
  }
  // this section should start with the boundary
  if (s2.indexOf(boundary) < 0) {
    s = "POST - Bad boundary search during filename search.";
    filelog(0, i, s);
    HTTP_error_400(s);
    return;
  }
  // look for filename
  filename = "";
  header_end = false;
  while (!header_end) {
    // Search for filename string between double quotes
    i = s2.indexOf("filename=\"");
    if (i >= 0) { // found it
      j = s2.indexOf('"', i + 10);
      filename = s2.substring(i + 10, j);
    }

    if (filename.length() >= MAXFILENAMELENGTH) {
      s = "POST - File name too long";
      filelog(0, i, s);
      HTTP_error_400(s);
      return;
    }

    // read the next line
    i = socket_read_line(c, sizeof(c));
    if (i > 0) {
      s = "POST - Bad packet read during filename search.";
      filelog(0, i, s);
      HTTP_error_400(s);
      return;
    }
    s2 = String(c);
    i = s2.indexOf('\n');
    if ((i == 0) || (i == 1)) { // blank line
      header_end = true;
    }
  } // while
  if (filename == "") {
    s = "POST - Filename not found in header.";
    filelog(0, i, s);
    HTTP_error_400(s);
    return;
  }
  //
  // The next part of the message is the ASCII file itself which is terminated with the
  // boundary= string folled by a "--<CR><LF>"


  // Create the target file
  filename.toCharArray(fn, sizeof(fn));
  if (SD.exists(fn)) SD.remove(fn);
  Fileupload = SD.open(fn, FILE_WRITE);
  if (!Fileupload) {
    s = "Bad POST file open.";
    filelog(0, 0, s);
    HTTP_error_400(s);
    Fileupload.close();
    return;
  }


  // the client will pre-pend (and also append) a "--" to the boundary marker
  boundary = "\r\n--" + boundary + "--\r\n";
  s1 = boundary;
  ring_size = sizeof(ring_buffer);
  ring_head = 0;
  ring_tail = 0;
  s1_ptr = 0;

  // Buffer the incoming characters as long as they match the Entity Delimiter "------WebKitForm..."
  // which is the last data sent and not part of the file upload.
  //DWRITE(LED, 1);
  while (s1.charAt(s1_ptr) != '\0') {
    // check to see if client.available
    if (!client.available()) {
      s = "VERB POST Ring Buffer no character to read.";
      filelog(0, ring_head, s);
      HTTP_error_400(s);
      Fileupload.close();
      //DWRITE(LED, 0);
      return;
    }
    fupload = socket_read();
    ring_buffer[ring_head] = fupload;
    ring_head = (++ring_head) % ring_size;
    if (fupload == s1.charAt(s1_ptr)) {
      if (ring_head == ring_tail) { //error - buffer overflow
        s = "VERB POST Ring Buffer Overflow.";
        filelog(0, ring_head, s);
        HTTP_error_400(s);
        Fileupload.close();
        //DWRITE(LED, 0);
        return;
      } // if overflow...
      s1_ptr++; //point to next character in s1 to match
    } // if
    else // our saved input characters don't match our string
    {
      // write out everything except the last character; back up ring_head by 1
      if (ring_head == 0) ring_head = ring_size; else ring_head--;
      while (ring_tail != ring_head) {
        Fileupload.print(ring_buffer[ring_tail]);
        ring_tail = (++ring_tail) % ring_size;
      }
      //
      // write out the last character if it doesn't match the first character in s1 else save it
      //
      if (ring_buffer[ring_head] != s1.charAt(0)) {
        Fileupload.print(ring_buffer[ring_tail]); //write it out; ring_head points to empty slot
        s1_ptr = 0; //start all over again
      } else {
        ring_head++; // save
        s1_ptr = 1; // the character matched, start again with the next character
      } // else
    } //else
  } // while
  // Nothing in the final ring buffer is written to the file.
  Fileupload.close();
  http_verb_get("GET / HTTP/1.1"); // return to home page
  return;

}







/*

This routine reads a single line from the socket connections into a buffer until a newline ('\n')
character is received or if there is no more data to read. A null character ('\0') is appended
to the end. If successful the line is parsed for an HTTP verb and the beginning/end of the verb
string is provided. HTTP verbs include:

GET, POST (not HEAD or DELETE)


Parameters:

buf: pointer to a character array, terminated with a \n and \0 if successful
buf_length: size of buffer in bytes, must be 2 or more bytes in size for \n and \0. Note headers can be <= 2048
*verb_begin: position in buf where HTTP verb (GET, POST, etc) begins
*verb_end: position in buf where HTTP verb ends (including all verb parameters)

Returns:

0: Success, a line ending with a \r and null character was stored in the buffer and a proper
HTTP verb was found.
1: A line was read however the line exceeded the buffer length. Extra characters were discarded however
the buffer does end with a newline and null character. The line was not parsed for an HTTP verb.
2: No newline character found before end of data. Buffer contents are undefined.
3: Buffer length less than 2; no characters were read.
4: A line (with possibly zero characters) ending with a \n and null character was stored in the buffer
however the HTTP verb was not valid/recognized.
5: Error while parsing HTTP request.


*/



int read_HTTP_header (char buf[], int buf_length, int* verb_begin, int* verb_end) {
  int i, j;

  i = socket_read_line(buf, buf_length);
  if (i > 0) return (i);
  // we have a valid line
  if (DEBUG_INPUT_PACKETS) {
    Serial.print("HTTP Header:");
    Serial.println(buf);
  }
  i = 0;
  while ((buf[i] != '\n') && (buf[i] != 'G') && (buf[i] != 'P')) { //look for a verb or end-of-line
    i++;
  }
  if (i >= buf_length) return (2);
  if (buf[i] == '\n') return (4);
  if (buf[i] == 'G') j = 0; // GET
  if (buf[i] == 'P') j = 1; // POST
  switch (j) {
    case 0: // Possible GET
      if ((buf[i + 1] != 'E') || (buf[i + 2] != 'T')) return (4); // unknown verb
      break;
    case 1: // Possible POST
      if ((buf[i + 1] != 'O') || (buf[i + 2] != 'S') || (buf[i + 3] != 'T')) return (4); // unknown verb
      break;
  }
  if ((j == 0) || (j == 1)) { // if a GET or PUT
    *verb_begin = i;
    while (buf[i] != ' ') { // index over the verb
      if (buf[i] == '\n') return (4); // unknown verb
      i++;
    }
    if (i >= buf_length) return (5);
    //Serial.println("skip spaces...");
    while (buf[i] == ' ') { // index over the spaces
      if (buf[i] == '\n') return (4); // unknown verb
      i++;
    }
    if (i >= buf_length) return (5);
    //Serial.println("include verb parameter...");
    while (buf[i] != ' ') { // index over the verb parameter which ends with a space
      if (buf[i] == '\n') return (4); // unknown verb
      i++;
    }
    i--;
    if (i >= buf_length) return (5);
    *verb_end = i;
    return (0);
  } // if
  return (4);
}






void http_request() {
  char http_buffer[MAXHTTPHEADERSIZE];
  int i, j, k;

  client = server.available();   // listen for incoming clients
  if (client) {                             // if you get a client,
    if (DEBUG_INPUT_PACKETS) Serial.println("\nNew client"); // print a message out the serial port
    while (client.connected()) {            // loop while the client's connected
      if (DEBUG_INPUT_PACKETS) Serial.println("Client Connected");
      if (client.available()) {             // if there's bytes to read from the client,
        i = read_HTTP_header(http_buffer, sizeof(http_buffer), &j, &k); // read the first line of the HTTP packet
        if (i == 0) { // no error reading the HTTP header
          filelog(1, i, http_buffer); // log the HTTP request
          //
          // Parse the HTTP verb
          //
          if ((http_buffer[j] == 'G') && (http_buffer[j + 1] == 'E') && (http_buffer[j + 2] == 'T')) i = 0;
          if ((http_buffer[j] == 'P') && (http_buffer[j + 1] == 'O') && (http_buffer[j + 2] == 'S')) i = 1;
          switch (i) {
            case 0: // GET
              http_verb_get(&http_buffer[j + 3]);
              break;
            case 1: // POST
              http_verb_post(&http_buffer[j + 4]);
              break;
            default:
              filelog(0, i, "Error - unrecognized verb in HTTP Header");
              HTTP_error_404();
              break;
          };
          //
        } else { // error reading the first line of the HTTP request
          filelog(0, i, "Error reading HTTP Header");
          HTTP_error_404();
        } // else
      } // if client available
      client.stop();
      if (DEBUG_INPUT_PACKETS) Serial.println("client.stop");
    } // while client.connected
    if (DEBUG_INPUT_PACKETS) Serial.println("client not connected");
  }
  return;
}






void HTTP_error_404() {
  // read rest of socket
  socket_read_to_end ();
  // send back error messgae
  socket_println("HTTP/1.1 404 FILE NOT FOUND");
  client.println();
  socket_println("404 RESOURCE OR FILE NOT FOUND");
  client.stop();
  return;
}




void HTTP_error_400(char msg[]) {
  // read rest of socket
  socket_read_to_end ();
  // send back error messgae
  socket_println("HTTP/1.1 400 Bad Request");
  client.println();
  //socket_println("400 Client Error (e.g. malformed request syntax)");
  socket_println(msg);
  client.stop();
  return;
}





void sys_logfile_page() {
  File logfile;
  unsigned long logsize, seek;
  char c;

  // the first line of the HTTP packet has been read, read the rest of the HTTP packet
  http_packet_read_to_end();
  socket_println("HTTP/1.1 200 OK");
  socket_println("Content-Type:Text/html");
  socket_println(NULL);
  socket_println("<!DOCTYPE html>");
  socket_println("<html>");
  socket_println("<head>");
  socket_println("<title>sys_logfile_page</title>");
  socket_println("</head>");
  socket_println("<body bgcolor=\"#E6E6FA\">");
  socket_println("<p><font size=6><center>Tegan Web Server</center></font></p>");
  socket_println("<p><font size=4><center>Tail of Log Files</center></font></p><br>");
  //
  socket_println("<p><font size=6><u>httperror.txt</u></font></p><br>");
  logfile = SD.open("httperror.txt", FILE_READ);
  if (logfile) {
    logsize = logfile.size();
    seek = 0;
    if (logsize > DISPLAYLOGSIZE) seek = logsize - DISPLAYLOGSIZE;
    logfile.seek(seek);
    while (logfile.available()) {
      c = logfile.read();
      if ((c != '\r') && (c != '\n')) socket_write(c);
      if (c == '\n') socket_println("<br>");
    }
    logfile.close();
  } else {
    socket_println("No errors, file 'httperror.txt' does not exist.<br><br>");
  }
  //
  socket_println("<p><font size=6><u>httplog.txt</u></font></p><br>");
  logfile = SD.open("httplog.txt", FILE_READ);
  if (logfile) {
    logsize = logfile.size();
    seek = 0;
    if (logsize > DISPLAYLOGSIZE) seek = logsize - DISPLAYLOGSIZE;
    logfile.seek(seek);
    while (logfile.available()) {
      c = logfile.read();
      if ((c != '\r') && (c != '\n')) socket_write(c);
      if (c == '\n') socket_println("<br>");
    }
    logfile.close();
  } else {
    socket_println("<u>httplog.txt</u> does not exist.<br><br>");
  }
  //
  socket_println("<br><br><br><a href=\"/ \">Home Page</a>");
  socket_println("</body></html>");
  socket_println(NULL);
  return;
}




// playground.html page

void playgroundhtml() {

  // the first line of the HTTP packet has been read, read the rest of the HTTP packet
  http_packet_read_to_end();
  stream_html_file("playground.html");
  return;
}



// iocontrol.html page
/*

httpparams: Null terminated string that contains the GET Parameters after the '?'

*/

void iocontrol2html(char httpparams[]) {
  int i;
  String s, io;

  http_packet_read_to_end();
  /*
  // convert HTTP line to String by adding \0 to end
  i = 0;
  while ((httpparams[i] != '\n') && (i < MAXHTTPHEADERSIZE)) i++;
  if (i < MAXHTTPHEADERSIZE) {
    */
    httpparams[i] = '\0'; // replace the \n with a \0
    s = String(httpparams);
    // search for DO2 - DO14 as a passed parameter
    for (i = 2; i < 14; i++) {
      io = "DO" + String(i);
      //Serial.println(io);
      if (s.indexOf(io) > -1) {
        io = "DO" + String(i) + "=HIGH";
        if (s.indexOf(io) > -1) {
          DWRITE(i, 1);
        } else {
          io = "DO" + String(i) + "=LOW";
          if (s.indexOf(io) > -1) DWRITE(i, 0);
        } // else
      } // if
    } // for
    /*
  } else {
    filelog(0, i, "HTTP Header too long on iocontrol.html");
  }
  */
  stream_html_file("iocontrol.html");
  return;
}


/*
char * DPINSTATUS[3] = {"DIGITAL UNDEFINED", "DIGITAL INPUT", "DIGITAL OUTPUT"};
struct DPIN_struct {
  char *name;      // User asigned PIN name
  int direction;   // 0: Not Used; 1:Not initialized; 2:Input; 3:Output
};
struct DPIN_struct DPIN[1];

*/

/*

httpparams: Null terminated string that contains the GET Parameters after the '?'

*/

void iocontrolhtml(char httpparams[]) {
  int pin_index, used_pin_count, last_pin_count, max_dpins;
  int i, j;
  boolean start_table;

  // Process any passed parameters
  i = 0;
  while (httpparams[i] != '\0') {
    //Serial.println(httpparams[i], DEC);
    if (httpparams[i] == '&') i++; // for after first loop
    if (httpparams[i] != 'D') {
      filelog(0, i, "Error - I/O Control D header invalid");
      HTTP_error_400("Error - I/O Control D header invalid");
      return;
    }
    if ((httpparams[i+1] < '0') || (httpparams[i+1] > '9')) {
      filelog(0, i, "Error - I/O Control first digit header invalid");
      HTTP_error_400("Error - I/O Control first digit header invalid");
      return;
    }
    j = httpparams[i+1] - 48; // first digit
    if (((httpparams[i+2] < '0') || (httpparams[i+2] > '9')) && (httpparams[i+2] != '=')) {
      filelog(0, i, "Error - I/O Control second digit header invalid");
      HTTP_error_400("Error - I/O Control second digit header invalid");
      return;
    }
    if (httpparams[i+2] != '=') { //we have a two-digit PIN number
      j = j * 10;
      j += httpparams[i+2] - 48;
      i += 3; // point to either '='
    } else i += 2;
    // i now points to either '='
    if (httpparams[i] != '=') {
      filelog(0, i, "Error - I/O Control '=' digit header invalid");
      HTTP_error_400("Error - I/O Control '=' digit header invalid");
      return;
    }
    i++; // now points to 'I', 'O', 'H', L'
    if (httpparams[i] == 'I') {
      if (DPIN[j].direction == 1) {
        pinMode(j, INPUT);
        DPIN[j].direction = 2; // set to input
      } else filelog(0, j, "Warning - I/O Control digital I/O pin already initialized or not available");
    }
    if (httpparams[i] == 'O') {
      if (DPIN[j].direction == 1) {
      pinMode(j, OUTPUT);
      DPIN[j].direction = 3; // set to output
      } else filelog(0, j, "Warning - I/O Control digital I/O pin already initialized or not available");
    }
    if (httpparams[i] == 'H') {
      // check to make sure its set for output
      if (DPIN[j].direction == 3) {
        DWRITE(j, 1);
        //DPIN[j].value = 1;
      } else filelog(0, j, "Warning - I/O Control digital I/O pin not initialized for output or not available");
      
    }
    if (httpparams[i] == 'L') {
      // check to make sure it's set for output
      if (DPIN[j].direction == 3) {
        DWRITE(j, 0);
        //DPIN[j].value = 0;
      } else filelog(0, j, "Warning - I/O Control digital I/O pin not initialized for output or not available");
    }
    if (httpparams[i] == 'R') {
      // check to make sure it's set for either input or output
      if ((DPIN[j].direction == 2) || (DPIN[j].direction == 3)) {
        pinMode(j, INPUT);
        DPIN[j].direction = 1; // reset it
        DPIN[j].value = 0xFFFF;
      } else filelog(0, j, "Warning - I/O Control digital I/O pin can not be reset");
    }
    
    i++; // now points to '\0' or '&'
  } // while
  // Now display the status of the I/O pins
  max_dpins = sizeof(DPIN) / sizeof(DPIN[0]);
  // the first line of the HTTP packet has been read, read the rest of the HTTP packet
  http_packet_read_to_end();
  socket_println("HTTP/1.1 200 OK");
  socket_println("Content-Type:Text/html");
  socket_println(NULL);
  socket_println("<!DOCTYPE html>");
  socket_println("<html>");
  socket_println("<head>");
  socket_println("<title>iocontrolhtml</title>");
  socket_println("</head>");
  socket_println("<body bgcolor=\"#E6E6FA\">");
  socket_println("<p><font size=6><center>Tegan Web Server</center></font></p>");
  socket_println("<p><font size=4><center>I/O Control</center></font></p><br>");

  // Analog I/O
  j = 4; // Last analog port to display + 1
  pin_index = 0;
  used_pin_count = 0;
  start_table = true;
  while (pin_index < j) { // Analog pins 0 thru 3
    used_pin_count++;
    if ((used_pin_count == 1) && start_table) { // we have an entry and need a form and table
      //socket_println("<form method=\"GET\">");
      socket_println("<table border=\"1\" style=\"width:100%\">");
      socket_println("<caption>ANALOG INPUTS</caption>");
      start_table = false;
    } // if
    //if (used_pin_count != j) {
      if ((used_pin_count % 4) == 1) socket_println("<tr>"); // start a new row
        socket_print("<td><center>Pin# ");
        socket_writenum(pin_index);
        socket_print("<br>ANALOG INPUT");
        socket_print("<br>VALUE = ");
        socket_writenum(AREAD(pin_index));
        socket_println("</center></td>");
    //} // if
    if ((used_pin_count % 4) == 0) socket_println("</tr>"); // end the row
    pin_index++;
  } // while
  if ((used_pin_count % 4) != 1) socket_println("</tr>"); // end the row if we didn't in the loop
  if (used_pin_count > 0) { // we have at least one entry and need to end the form and table
    socket_println("</table><br><br>");
  } // if

  // Digital I/O
  pin_index = 0;
  used_pin_count = 0;
  last_pin_count = 0;
  start_table = true;
  while (pin_index < max_dpins) {
    if (DPIN[pin_index].direction) used_pin_count++; // this digital pin can be used

    if ((used_pin_count == 1) && start_table) { // we have an entry and need a form and table
      socket_println("<form method=\"GET\">");
      socket_println("<table border=\"1\" style=\"width:100%\">");
      socket_println("<caption>DIGITAL I/O CONTROL</caption>");
      start_table = false;
    }
    if (used_pin_count != last_pin_count) {

      if ((used_pin_count % 4) == 1) socket_println("<tr>"); // start a new row
      if (DPIN[pin_index].direction == 1) { // digital pin usable but not initialized
        socket_println("<td BGCOLOR=\"#FF6699\"><center>Pin #");
        socket_writenum(pin_index);
        socket_print("<br><br><input type=\"radio\" name=\"D");
        socket_writenum(pin_index);
        socket_println("\" value=\"I\">Assign To Input");
        socket_print("<br><input type=\"radio\" name=\"D");
        socket_writenum(pin_index);
        socket_print("\" value=\"O\">Assign To Output");
        socket_println("</center></td>");
      }
      if (DPIN[pin_index].direction == 2) { // digital pin input
        socket_print("<td><center>Pin# ");
        socket_writenum(pin_index);
        socket_print("<br><br>DIGITAL INPUT");
        socket_print("<br>VALUE = ");
        socket_writenum(DREAD(pin_index));
        socket_print("<br><input type=\"radio\" name=\"D");
        socket_writenum(pin_index);
        socket_print("\" value=\"R\"><font size=1>RESET</font>");
        socket_println("</center></td>");
      }
      if (DPIN[pin_index].direction == 3) { // digital pin output
        socket_println("<td><center>Pin #");
        socket_writenum(pin_index);
        socket_print("<br><br><input type=\"radio\" name=\"D");
        socket_writenum(pin_index);
        socket_print("\" value=\"H\"");
        if (DPIN[pin_index].value == 1) socket_print(" checked");
        socket_println(">HIGH");
        socket_print("<br><input type=\"radio\" name=\"D");
        socket_writenum(pin_index);
        socket_print("\" value=\"L\"");
        if (DPIN[pin_index].value == 0) socket_print(" checked");
        socket_print(">LOW");
        
        socket_print("<br><input type=\"radio\" name=\"D");
        socket_writenum(pin_index);
        socket_print("\" value=\"R\"><font size=1>RESET</font>");
        
        socket_println("</center></td>");
      }
      if ((used_pin_count % 4) == 0) socket_println("</tr>"); // end the row
      last_pin_count = used_pin_count;
    } // if

    pin_index++;
  } // while
  if ((used_pin_count % 4) != 1) socket_println("</tr>"); // end the row if we didn't in the loop

  if (used_pin_count > 0) { // we have at least one entry and need to end the form and table
    socket_println("</table>");
    socket_println("<br><center>AI or DI = -1 indicates invalid channel selection</center>");
    socket_println("<br><center><input type=\"submit\" value=\"Submit\"></center>");
    socket_println("</form>");
  }
  socket_println("<br><br><a href=\"/iocontrol.html\">REFRESH</a>");
  socket_println("<br><br><br><a href=\"/ \">Home Page</a>");
  socket_println("</body></html>");
  socket_println(NULL);

  return;
}



void faviconico () {

  // the first line of the HTTP packet has been read, read the rest of the HTTP packet
  http_packet_read_to_end();
  stream_binary_file("favicon.ico",  "image/x-icon");
  return;
}


void environmentalhtml() {
  int i;
  //char substitute_data[4][MAXSUBCOL];

  // the first line of the HTTP packet has been read, read the rest of the HTTP packet
  http_packet_read_to_end();
  // calculate temperature
  i = (int) get_temp();
  sprintf(&substitute_data[0][0], "%d", i); // <%SL0>
  sprintf(&substitute_data[1][0], "%d", ((millis() / 1000) / 60)); // <%SL1>
  //sprintf(&substitute_data[2][0], "%d", WiFi.RSSI()); // <%SL2>
  strncpy(&substitute_data[3][0], my_ip_address, MAXSUBCOL);
  // the content of the HTTP response follows the header:
  stream_html_file("environmental.html");
  return;
}







// Return the temperature in F

float get_temp() {
  float f;
  int val = analogRead(pinTemp);
  val = AREAD(0);
  // Determine the current resistance of the thermistor based on the sensor value.
  float resistance = (float)(1023 - val) * 10000 / val;
  // Calculate the temperature based on the resistance value.
  f = 1 / (log(resistance / 10000) / B + 1 / 298.15) - 273.15; // this is the temp in C
  return ((int) (f * 9.0 / 5.0 + 32));
}







void html_favicon() {
  File Favicon;
  int i;
  // HTTP headers always start with a response code (e.g. HTTP/1.1 200 OK)
  // and a content-type so the client knows what's coming, then a blank line:
  // See if we have a favicon file to send...
  Favicon = SD.open("favicon.ico", FILE_READ);
  if (!Favicon) {
    Serial.println("Could not open favicon.ico");
    return;
  }
  Serial.println("Sending ICO file to client");
  client.println("HTTP/1.1 200 OK");
  client.println("Content-Type:image/x-icon");
  client.println("Content-Length: 1150");
  client.println();

  i = 0;
  while (Favicon.available()) {
    socket_write(Favicon.read());
    i++;
  }
  Serial.print("Number of bytes in Favicon.ico =");
  Serial.println(i);
  // close the file:
  Favicon.close();
  return;
}







