## Enroll
Subir archivo wav del enroll y asignarle un nombre y tags (separadas por coma). Devolver JSON de la "persona" en el output_file. 
- .../biovox_xt.exe
- enroll
- {name}
- {tags}
- .../file.wav
- .../output_file

## Deletion
Borrar voiceprint asociada al id proporcionado. Devolver el json del usuario borrado al archivo output_file.
- .../biovox_xt.exe
- delete
- {user_id}
- .../output_file

## Verification
Subir archivo wav a verificar el hablante (a traves del id de hablante proporcionado), verificar con el score necesario para darlo como success y devolver el JSON de respuesta en el output_file.
- .../biovox_xt.exe
- verify
- {user_id}
- .../file.wav
- {min_score}
- .../output_file

## Identification
Subir archivo wav a identificar el hablante (filtrar por tags si no llega como "null"), escoger el de mayor concordancia y devolver el json resultante al output_file.
- .../biovox_xt.exe
- identify
- {tags}
- .../file.wav
- {min_score}
- .../output_file

## Diarization
Subir archivo wav a diarizar y comparar hacia el grupo de tags proporcionado (si no se manda como "null"). Devolver el json resultante al output_file.
- .../biovox_xt.exe
- diarize
- {tags}
- .../file.wav
- .../output_file