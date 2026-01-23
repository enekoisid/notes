# IMPORTANTE: **Esta tarea es para crops a partir de imagenes**

pasos a seguir para setup:
1. `git clone https://gitlab.isid.com/backend/transcoderms.git` (no hacer si ya se tiene, lógico)
2. en el cursor/vs code/carpeta `git switch MP-73-revision-de-sacar-crop-del-transcoder-para-una-ima`
3. dotnet restore ; dotnet clean ; dotnet run
4. en el postman
```curl
curl --location 'https://localhost:8091/api/task' \
--header 'Content-Type: application/json' \
--data '{
    "pathOut": "/tmp",
    "pathIn": "http://172.29.48.1:8000/test.png",
    "notifyUrl": "http://172.29.48.1:8001/notification",
    "priority": 1,
    "type": "crops",
    "crops": [{
                    "id": 1,
                    "SMPTEIN": 0,
                    "x": 400,
                    "y": 10,
                    "widthPx": 1000,
                    "heightPx": 200
                }
                ],
    "profileID": 1,
    "watermarks": [],
    "audioMapping": "",
    "taskTranscodeElements": [       
    ]
}'
```
5. para ver el crop ir a la carpeta de `pathOut` o en los logs (en caso de estar en vscode/cursor) control+click en la ruta que dice que se ha hecho el crop