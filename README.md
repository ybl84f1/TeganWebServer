# TeganWebServer
The Tegan Web Server for Arduino on Intel Edison

Written by Steve Engelbrecht

About the Tegan Web Server 

The Tegan Web Server (TeganWS) is a basic web server for the Intel Edison platform inspired by Tom Igoe's Arduino WiFi web server. TeganWS can be used as a simple web server, as a simple web interface to control Edison's I/O, or as the basis of an extendable web server application. TeganWS responds to basic HTTP browser requests for web pages by streaming out HTML files from the SD storage device and also has the ability to include server-generated data in those pages.

TeganWS's features include: 

* Pre-built web pages for setting and displaying analog and digital pins.
* Ability to serve standard HTML files by supporting HTTP GET and POST requests from browsers.
* HTML Page Methods and Substitution Tags enabling web pages to include server-generated data.
* Built-in logging capability with support for both HTTP requests and errors.
* HTML file upload capability for easy updating of server web pages from a client.
* Debug setting for dislaying all HTTP incoming and outgoing packets.
* Favicon support

The Tegan Web Server runs in the Arduino environment on the Edison and should be easily portable to both Arduino and Galileo. 
TeganWS is useful for:

* Displaying and controlling the I/O (digital, analog, PWM) of your Edison.
* Providing a custom web interface to your Edison application.
* Running a simple web server serving simple HTML web pages.

What TeganWS is not useful for:

* Hosting a high-volume web site.
* Advanced web site services (e.g. SSL, CSS, Java)
* Real-time applications

Note: Ths software is still in beta form. The software is fully functional however it is not completely commented and some of the HTML tutorials have not been completed. Please send any comments/feedback my way.

REQUIREMENTS:
* Intel Edison connected to a network via built-in WiFi
* SD Card for HTML file storage
 
OPTIONAL:
* Grove Expansion kit with temperature sensor on A0
* Grove buzzer on D4
