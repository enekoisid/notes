pasos a seguir para setup:
1. `git clone https://gitlab.isid.com/backend/transcoderms.git` (no hacer si ya se tiene, lógico)
2. en el cursor/vs code/carpeta `git switch MP-154-bug-recursividad-get-transcoder`
3. dotnet restore ; dotnet clean ; dotnet run
4. crear una tarea (ejemplo usando el de crops usando un video, que las imagenes tienen el bug de la otra rama)
```curl
curl --location 'https://localhost:8091/api/task' \
--header 'Content-Type: application/json' \
--data '{
    "pathOut": "/tmp",
    "pathIn": "http://172.29.48.1:8000/test.mp4",
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

5. hacer un get de la tarea
```curl
curl --location 'https://localhost:8091/api/task/{{taskId}}' \
--header 'Content-Type: application/json' \
--data ''
```