para entrenar hay que usar el archivo pt de la versión que se quiera, ya que en el pt se da la estructura de neuronas junto a un "cerebro preentrenado", que, al saber detectar objetos basicos, facilita el entrenamiento con los datos nuevos. Este pt se usará para entrenar el onnx.

con los yaml se le dice al proceso de entrenamiento donde tiene que buscar las imagenes, los labels y las relaciones entre ambos
 - en el caso de los yaml de ultralytics vienen con una función de autodescarga de estos recursos
 - en el caso de ser un dataset propio nuestro, habría que recuperar el yaml usado para entrenar el modelo anterior, o, generarlo de nuevo

 descargando el pt te aseguras de descargar el modelo con el numero de parametros que se necesitan, siendo los n los que menos recursos consumen pero "peor" detectan los objetos, y los X los que mas consumen pero mejor detectan los objetos (incluso lejos y borrosos)

 con el pt descargado se puede entrenar con un script de python muy sencillo
 ```python
 from ultralytics import YOLO

# Load a model
model = YOLO("yolo11n.pt")  # load a pretrained model (recommended for training)

# Train the model
results = model.train(data="Objects365.yaml", epochs=100, imgsz=640)
```

de esta forma se enbtrenaría, con una nueva tecnología, el mismo dataset que tendriamos en una versión anterior (o con menos parametros) en formato pt, luego habría que convertir ese pt en onnx con otro comando extra.

```python
success = model.export(format="onnx")
```

es importante que el tamaño `imgsz` sea igual al tamaño que el modelo esperará recibir en la vida real (inferencia). Si lo entrenas a 640 pero luego en tu aplicación le pasas imágenes a 320, la precisión no será la adecuada.

a la hora de entrenar el modelo, el propio "proceso" se encarga de pasar las fotos, por ejemplo en 4k, al tamaño especificado en `imgsz`

para saber el `imgsz` que hay que poner se puede sacar desde un onnx con este script
```json
import onnx

model = onnx.load("C:\\Users\\EnekoRebollo\\Documents\\repos\\carpeta_hermes\\hermes\\Logic\\Analysis\\VideoAnalytics\\Models\\yolov5m_Objects365.onnx")

print(model.graph.input[0].type.tensor_type.shape)
```

