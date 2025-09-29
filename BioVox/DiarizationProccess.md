## 1. Get BearerToken

Get BeareToken using the account/application information provided on `https://biovox.cloud/#/company/edit/me`.

```curl
curl --location 'https://api.biovox.cloud/oauth2/token/api-key' \
--header 'Accept-Charset: utf-8' \
--header 'Content-Type: application/json' \
--header 'Cookie: connect.sid=s%3Aoufwvijcw4KVgxSHO9PtdvsJH733GUjE.MK%2BIzEcb301%2FJm0cWIzxGJrbNzODkSigbIzI0PGkPsU' \
--data '{   
	"app_id": "{{AppId}}",
	"api_key": "{{ApiKey}}"
}'
```

## 2. Get JWT

To download de diarization result, you will be asked to provide a JWT, to get it.

```curl
curl --location 'https://api.biovox.cloud//biovox/download-token' \
--header 'Authorization: Bearer {{BearerToken}}' \
--header 'Cookie: connect.sid=s%3Aoufwvijcw4KVgxSHO9PtdvsJH733GUjE.MK%2BIzEcb301%2FJm0cWIzxGJrbNzODkSigbIzI0PGkPsU'
```

## 3. Upload the audio to diarize

Upload the audio you will like to get diarized and save its `fileName` from the result JSON.

```curl
curl --location 'https://api.biovox.cloud/biovox/audio' \
--header 'Authorization: Bearer {{BearerToken}}' \
--header 'Cookie: connect.sid=s%3Aoufwvijcw4KVgxSHO9PtdvsJH733GUjE.MK%2BIzEcb301%2FJm0cWIzxGJrbNzODkSigbIzI0PGkPsU' \
--form 'file=@"/{{WAV_file_url}}"'
```

## 4. Execute diarization job to the uploaded audio

Start diarization job. You can provide the voiceprints you want to locate in the audio using `voiceprints` part. to get them follow the step `4b`. Save `href` from the response JSON.

```curl
curl --location 'https://api.biovox.cloud/biovox/diarization_voiceprints' \
--header 'Authorization: Bearer {{BearerToken}}' \
--header 'Content-Type: application/json' \
--header 'Cookie: connect.sid=s%3Aoufwvijcw4KVgxSHO9PtdvsJH733GUjE.MK%2BIzEcb301%2FJm0cWIzxGJrbNzODkSigbIzI0PGkPsU' \
--data '{
    "fileName": "{{Filename}}",
    "voiceprints": [
        "{{VoiceprintId1}}",
        "{{VoiceprintId2}}"
    ]
}'
```

### 4b.  Get voiceprint list
___

Get the stored voiceprints and save the disered one´s `id` from the result json.

```curl
curl --location 'https://api.biovox.cloud/biovox/enrollment' \
--header 'Authorization: Bearer {{BearerToken}}' \
--header 'Cookie: connect.sid=s%3Aoufwvijcw4KVgxSHO9PtdvsJH733GUjE.MK%2BIzEcb301%2FJm0cWIzxGJrbNzODkSigbIzI0PGkPsU'
```

## 5. Check the job using `href` url

Check the job status using the url. When the `status` is marked as done, will be provided with another url, save it.

```curl
curl --location 'https://api.biovox.cloud/biovox/diarization_voiceprints/jobs/{{JobId}}' \
--header 'Authorization: Bearer {{BearerToken}}' \
--header 'Cookie: connect.sid=s%3Aoufwvijcw4KVgxSHO9PtdvsJH733GUjE.MK%2BIzEcb301%2FJm0cWIzxGJrbNzODkSigbIzI0PGkPsU'
```

## 6. Get the diarization result json

Using the url from `Step 5`, adding a `token` param, you will be provided with the diarization result. The url should look like this: 

```curl --location 'https://api.biovox.cloud/biovox/diarization_voiceprints/{{VoiceprintId}}/download?token={{JWTToken}}' \
--header 'Cookie: fileDownload=true, path=/; connect.sid=s%3Aoufwvijcw4KVgxSHO9PtdvsJH733GUjE.MK%2BIzEcb301%2FJm0cWIzxGJrbNzODkSigbIzI0PGkPsU'
```