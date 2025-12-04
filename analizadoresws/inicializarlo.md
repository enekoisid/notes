1. poner el contenido de la carpeta dlls dentro del root de la app
2. copiar el config que se quiere usar y cambiar el archivo a Config.xml
3. copiar la carpeta tools a la carpeta root de la app
4. copiar las carpetas de los servicios que se quieran usar dentro de root de la aplicación (por ejemplo, biovox)
5. compilar SIEMPRE en x86

## para el servicio
    > sc create "ISID - Servicio Analizadores 4" binPath="C:\Users\development\Desktop\AnalizadoresWS\{fileName}.exe" start=AUTO depend=TCPIP

1. Ir a servicios
2. Buscar el servicio e ir a propiedades
3. Poner ejecución en manual
4. Cambiar usuario a VIDEO/{passwd}