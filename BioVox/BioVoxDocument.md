### **Funcionalidades y Conceptos de BioVox.cloud 1.3**
---
* **Creación de Huella de Voz:** Para crear una huella de voz (`voiceprint`), se necesita un audio de **voz natural** (`free speech`) con una duración mínima de **30 segundos**, siendo recomendable **1 minuto**.

    > No se puede almacenar mas de una huella de voz por identificador, por lo que haria falta el uso de una base de datos externa que guarde la relación persona-identificadores.
    > Tambien está la posibilidad de poner el mismo nombre al voiceprint dado que el nombre no es un identificador unico dentro de los datos de identificación del voiceprint.
* **Detección Anti-Impostor:** El sistema cuenta con una medida de seguridad para detectar si se reproduce una grabación en lugar de una voz en vivo, lo que resultaría en un reconocimiento negativo.

    > Esto puede acarrear problemas para la identificación de personas usando grabaciones de llamadas por la forma de codificación de las llamadas telefónicas.
* **Independencia del Idioma:** La biometría de voz funciona con la misma persona hablando en **diferentes idiomas**, ya que se centra en las características únicas de la voz, no en el contenido.
* **Operación en audios sin streaming**: BioVox no admite la identificación en tiempo real, para ello habria que ingtegrarlo en un sistema que recoja el video en streaming y se lo pase como audio independiente.
* **Operaciones de Reconocimiento:**
    * **Identificación (1:N):** El servicio **siempre** devuelve un porcentaje de probabilidad para cada persona registrada en la base de datos que podría coincidir con el audio.
    * **Verificación (1:1):** Permite verificar si una persona específica ha hablado en una grabación.
* **Umbral de Certeza:** El umbral de certeza para considerar una coincidencia es **modificable**, lo que permite ajustar el nivel de confianza necesario para aprobar un resultado.
* **Audio con Múltiples Personas:**
    * Para grabar y reconocer a varias personas, es necesario que sus huellas de voz hayan sido creadas previamente a través del proceso de **enrolamiento**.
    * La precisión del reconocimiento se ve afectada por factores como el **ruido ambiental**, la **lejanía del micrófono** y el **solapamiento de voces**.
* **Pre-procesamiento del Audio:** Para obtener los mejores resultados, es recomendable realizar un **pre-procesamiento** del audio antes de enviarlo a la API (limpiar ruido, separar voces, normalizar volumen, etc.).
* **Limitaciones del Servicio:**
    * **No hace diarización:** El software solo identifica a las personas, pero **no separa** las voces en el audio ni las etiqueta por tramo. Para ello, se deben usar herramientas externas como pyannote.audio, Kaldi o WhisperX.
    * **No hace transcripción:** BioVox.cloud se enfoca únicamente en la **biometría de voz** y no convierte el audio a texto (STT - Speech to Text).
    * **Sin Garantía de 100% de Exactitud:** La exactitud del sistema no está garantizada al 100%.

### **Estructura y Versión de la API**
---
* **Formatos de Audio:** El servicio funciona con archivos de audio en formato **WAV y MP3**.
* **`Endpoints` principales:**
    * **Enrolamiento:** `https://api.biovox.cloud/biovox/enrol` (POST)
    * **Verificación:** `https://api.biovox.cloud/biovox/verify` (POST)
    * **Identificación:** `https://api.biovox.cloud/biovox/identify` (POST)

### Conclusiones
* Al no contar con un proceso de diarización haria falta contratar otro servicio que se encargue de ello para el uso de identificacion de quien ha dicho que en un audio, lo que aumentaria el costo y la dificultad, teniendo que segmentar el audio en fragmentos por hablante antes de pasarlo a BioVox para su reconocimiento 1:1.
* Util sobretodo para la identificacion de si alguien ha hablado en un grupo de voces (1:N) o para casos de biometria por voz (1:1).
* La API depende en gran medida de la calidad del audio de entrada. La recomendación de pre-procesar el audio (limpiar ruido, normalizar volumen) sugiere que el éxito de la identificación podría depender del trabajo previo que se realice en el audio antes de ser enviado al servicio.
* Hace falta si o si audios limpios y constantes de duracion entre 1 min o 30s de cada persona que se quiera identificar para poder hacer la identificacion de dicha persona.