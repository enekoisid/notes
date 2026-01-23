**Global settings**
- conf_score = 0.5
- imgsz = 640

# Yolo11
En esta prueba se han comparado los modelos yolo11S y yolo 11N, pasandose un video de una camara apuntando a una calle con cocher y peatones, se ha visto una ligera mejora en la detección de objetos con el modelo `S`, sobreotdo a la hora de detectar `handbag`, cosa que el modelo `N` no consiguió detectar enn ningun momento. En el caso de los peatones, coches y camiones, ambos modelos han dado una respuesta similar por no decir identica.

En este caso, se ve clara la mejora usando un modelo mas grande, dado que obtiene mejor los resultados, siendo mas preciso en cosas que el modelo mas pequeño pasa por alto.

# yolo11n Objects 365 vs Yolo11n
En esta prueba, se han comparado los resultados del modelo base yolo11n de ultralytics con el modelo entrenado con Objects365, usando otro video de una calle, en diferente ángulo con una altua parecida a un poste de luz. Las detecciones han sido parecidas, siendo la unica diferencia los objetos que existen en el modelo entrenado con yolo365. Ambas han detectado a las mismas personas y mismos coches. Una diferencia a marcar, aunque pequeña, es que el entrenado con 365 ha encontrado `handbag` en las bolsas de una señora, siendo que en modelo base no lo ha detectado en ningun momento, pero, viendo los resultados del anterior, se puede sacar en claro que un modelo mas grande como el `S` o el `M` lo detectarian sin problemas.

En este caso, habría que entrenar un modelo con este dataset, dado que el modelo base no detecta los objets del 365, que nos son necesarios, por ejemplo, SUV, pero se le ve bastante capaz al mnodelo base para tareas que no reqquieran objetos solo existentes en el 365.

# visdrones N vs yolo 11n
En esta prueba, se han comparado el modelo yolo11n y el mismo entrenado con el dataset visdrones, en un video captado por pegasus (helicotero de trafico).

En la prueba se puede ver claramente la superioridad del modelo entrenado con visdrones, siendo que detecta muchisimo mejor los coches. Un apunte es que ninguno consigue detectar el tractor que se ve en el video, pero en cuanto a los coches, lo hace bastante mejor el modelo entrenado con visdrones que el modelo base

# visdrones 11s vs visdrones5m
En esta prueba, se han comparado los modelos yolo11n y yolo11s.
Se ha usado 5 videos en total para estas pruebas, 1 de parkour, el de pegasus, uno de una vista de drone de unos alpinistas y videos varios de carretera.

- En el video de parkour, el modelo yolo11, ha captado mejor los resultados del video, dando mas información que el modelo yolo5. Ambos han dado la misma información en los planos mnos de tipo dron, detectando muy rara vez a las personas, pero en cuanto a la detección aerea de vehiculos y peatones, hay un claro ganador, que es el yolo11
- En el video de la rotonda, hay un claro ganador, y vuelve a ser yolo11, dando mas detecciones de los coches que se ven en el video
- En el video del coche solitario en la carretera, ambos modelos han dado el mismo resultado
- En el video de la montaña, aunque los resultados han sido bastante parecidos, ha tenido mas estabilidad el modelo yolo5 a la hora de detectar a las personas
- En el video de pegasus, el modelo yolo11 ha tenido mas falsos positivos con las letras al rededor del video, pero en cuanto a las detecciones de los coches, ha tenido mejor resultado total que el modelo 5
