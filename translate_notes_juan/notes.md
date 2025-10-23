
# Prueba: poner un codigo de idioma no existente en `translationToLanguages`

```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
        "patata-supersonica"

    ],
    "translation_fields": [
        "text",
        "summarization",
        "aaa",
        "algo"
    ],
    "translation_source": [
        {
            "text": "Reflexiones sobre lo inexplicable y la decisión de retirarse",
            "summarization": "Summarize for meeting agenda only",
            "Aaa": "cuando Zoiberg sale gritando gluglugu por la puerta es gracioso",
            "Algo": "¿Ustedes piensan antes de hablar o hablan tras pensar?",
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            }
        },
        {
            "text": "La incertidumbre como motor de la innovación",
            "summarization": "",
            "aaa": "Hola, ¿que tal estas? Vamos a hacer la compra pesadito",
            "algo": "It´s very difficult todo esto",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            }
        }
    ]
}
```

ó

```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
        "eus"

    ],
    "translation_fields": [
        "text",
        "summarization",
        "aaa",
        "algo"
    ],
    "translation_source": [
        {
            "text": "Reflexiones sobre lo inexplicable y la decisión de retirarse",
            "summarization": "Summarize for meeting agenda only",
            "Aaa": "cuando Zoiberg sale gritando gluglugu por la puerta es gracioso",
            "Algo": "¿Ustedes piensan antes de hablar o hablan tras pensar?",
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            }
        },
        {
            "text": "La incertidumbre como motor de la innovación",
            "summarization": "",
            "aaa": "Hola, ¿que tal estas? Vamos a hacer la compra pesadito",
            "algo": "It´s very difficult todo esto",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            }
        }
    ]
}
```

da error 

```json
{'taskExecutionID': 16, 
'state': 'erroneous', 'errorMessage': 'One or more errors occurred. (Cannot deserialize the current JSON object (e.g. {"name":"value"}) into type \'TranscriptionModule.Transcriptor.TranslationResult[]\' because the type requires a JSON array (e.g. [1,2,3]) to deserialize correctly.To fix this error either change the JSON to a JSON array (e.g. [1,2,3]) or change the deserialized type so that it is a normal .NET type (e.g. not a primitive type like integer, not a collection type like an array or List<T>) that can be deserialized from a JSON object. JsonObjectAttribute can also be added to the type to force it to deserialize from a JSON object.Path \'error\', line 1, position 9.)', 'notifyUrl': 'http://172.29.48.1:5007/notification', 'startDateTime': '2025-10-23T12:13:11.0283342+02:00', 'finishDateTime': '2025-10-23T12:13:11.3206734+02:00', 'progress': -1, 'taskID': 16, 'result': ''}
```

### Conclusión
Cuando se le pasa un codigo de idioma no valido da error. Ademas el json del error está mal formado dado que usa comillas simples en lugar de comillas dobles para las strings y las claves


# Prueba: Ver si es case sensitive

json de entrada:
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
        "uk",
        "eu",
        "it"

    ],
    "translation_fields": [
        "text",
        "summarization",
        "aaa",
        "algo"
    ],
    "translation_source": [
        {
            "text": "Reflexiones sobre lo inexplicable y la decisión de retirarse",
            "summarization": "Summarize for meeting agenda only",
            "Aaa": "cuando Zoiberg sale gritando gluglugu por la puerta es gracioso",
            "Algo": "¿Ustedes piensan antes de hablar o hablan tras pensar?",
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            }
        },
        {
            "text": "La incertidumbre como motor de la innovación",
            "summarization": "",
            "aaa": "Hola, ¿que tal estas? Vamos a hacer la compra pesadito",
            "algo": "It´s very difficult todo esto",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            }
        }
    ]
}
```

resultado: 

```json
{
    "translations": [
        {
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Reflections on the Unexplained and the Decision to Retire"
                },
                {
                    "Language": "fr",
                    "Text": "Réflexions sur l’inexpliqué et la décision de prendre sa retraite"
                },
                {
                    "Language": "uk",
                    "Text": "Роздуми про незрозуміле та рішення піти на пенсію"
                },
                {
                    "Language": "eu",
                    "Text": "Azalezinari eta erretiratzeko erabakiari buruzko gogoetak"
                },
                {
                    "Language": "it",
                    "Text": "Riflessioni sull\'inspiegabile e sulla decisione di andare in pensione"
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "Summarize for meeting agenda only"
                },
                {
                    "Language": "fr",
                    "Text": "Résumer pour l’ordre du jour de la réunion uniquement"
                },
                {
                    "Language": "uk",
                    "Text": "Підбиття підсумків лише для порядку денного засідання"
                },
                {
                    "Language": "eu",
                    "Text": "Summarize for meeting agenda only"
                },
                {
                    "Language": "it",
                    "Text": "Riepiloga solo per l\'ordine del giorno della riunione"
                }
            ],
            "aaa": [],
            "algo": []
        },
        {
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Uncertainty as a driver of innovation"
                },
                {
                    "Language": "fr",
                    "Text": "L’incertitude comme moteur de l’innovation"
                },
                {
                    "Language": "uk",
                    "Text": "Невизначеність як рушійна сила інновацій"
                },
                {
                    "Language": "eu",
                    "Text": "Ziurgabetasuna berrikuntzaren eragile gisa"
                },
                {
                    "Language": "it",
                    "Text": "L\'incertezza come motore dell\'innovazione"
                }
            ],
            "summarization": [],
            "aaa": [
                {
                    "Language": "en",
                    "Text": "Hello, how are you? Let\'s do the heavy shopping"
                },
                {
                    "Language": "fr",
                    "Text": "Bonjour, comment vas-tu? Faisons les gros achats"
                },
                {
                    "Language": "uk",
                    "Text": "Привіт, як справи? Давайте займемося важкими покупками"
                },
                {
                    "Language": "eu",
                    "Text": "Kaixo, zer moduz? Erosketa astuna egingo dugu"
                },
                {
                    "Language": "it",
                    "Text": "Ciao come stai? Facciamo lo shopping pesante"
                }
            ],
            "algo": [
                {
                    "Language": "en",
                    "Text": "It\'s very difficult all this"
                },
                {
                    "Language": "fr",
                    "Text": "C’est très difficile tout cela"
                },
                {
                    "Language": "uk",
                    "Text": "Все це дуже складно"
                },
                {
                    "Language": "eu",
                    "Text": "It ́s very difficult todo esto"
                },
                {
                    "Language": "it",
                    "Text": "È molto difficile tutto questo"
                }
            ]
        }
    ]
}
```

### Conclusión
Es case sensitive, por lo que si se pone con mayisculas y es sin ella la clave del texto a traducir no lo hará

# Prueba: Poner un idioma "no común"

json de entrada:
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
        "uk",
        "eu",
        "fy",
        "sk",
        "sw",
        "sa"
    ],
    "translation_fields": [
        "text",
        "summarization",
        "aaa",
        "algo"
    ],
    "translation_source": [
        {
            "text": "Reflexiones sobre lo inexplicable y la decisión de retirarse",
            "summarization": "Summarize for meeting agenda only",
            "aaa": "cuando Zoiberg sale gritando gluglugu por la puerta es gracioso",
            "algo": "¿Ustedes piensan antes de hablar o hablan tras pensar?",
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            }
        },
        {
            "text": "La incertidumbre como motor de la innovación",
            "summarization": "",
            "aaa": "Hola, ¿que tal estas? Vamos a hacer la compra pesadito",
            "algo": "It´s very difficult todo esto",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            }
        }
    ]
}
```

da error 
```json
{'taskExecutionID': 19, 'state': 'erroneous', 'errorMessage': 'One or more errors occurred. (Cannot deserialize the current JSON object (e.g. {"name":"value"}) into type \'TranscriptionModule.Transcriptor.TranslationResult[]\' because the type requires a JSON array (e.g. [1,2,3]) to deserialize correctly.\r\nTo fix this error either change the JSON to a JSON array (e.g. [1,2,3]) or change the deserialized type so that it is a normal .NET type (e.g. not a primitive type like integer, not a collection type like an array or List<T>) that can be deserialized from a JSON object. JsonObjectAttribute can also be added to the type to force it to deserialize from a JSON object.\r\nPath \'error\', line 1, position 9.)', 'notifyUrl': 'http://172.29.48.1:5007/notification', 'startDateTime': '2025-10-23T12:29:39.0215687+02:00', 'finishDateTime': '2025-10-23T12:29:39.3044063+02:00', 'progress': -1, 'taskID': 19, 'result': ''}
```

### Observación
codigos de idioma sacados de la pagina de wikipedia [ISO_639-1](https://es.wikipedia.org/wiki/ISO_639-1)

### Conclusión
Idiomas como el sánscrito, frisón, eslovaco... No los acepta


# Prueba: Enviar sin meta

```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
        "uk",
        "eu",
        "it"
    ],
    "translation_fields": [
        "text",
        "summarization",
        "aaa",
        "algo"
    ],
    "translation_source": [
        {
            "text": "Reflexiones sobre lo inexplicable y la decisión de retirarse",
            "summarization": "Summarize for meeting agenda only",
            "aaa": "cuando Zoiberg sale gritando gluglugu por la puerta es gracioso",
            "algo": "¿Ustedes piensan antes de hablar o hablan tras pensar?"
        },
        {
            "text": "La incertidumbre como motor de la innovación",
            "summarization": "",
            "aaa": "Hola, ¿que tal estas? Vamos a hacer la compra pesadito",
            "algo": "It´s very difficult todo esto",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            }
        }
    ]
}
```

resultado
```json
{
    "translations": [
        {
            "meta": {},
            "text": [
                {
                    "Language": "en",
                    "Text": "Reflections on the Unexplained and the Decision to Retire"
                },
                {
                    "Language": "fr",
                    "Text": "Réflexions sur l’inexpliqué et la décision de prendre sa retraite"
                },
                {
                    "Language": "uk",
                    "Text": "Роздуми про незрозуміле та рішення піти на пенсію"
                },
                {
                    "Language": "eu",
                    "Text": "Azalezinari eta erretiratzeko erabakiari buruzko gogoetak"
                },
                {
                    "Language": "it",
                    "Text": "Riflessioni sull\'inspiegabile e sulla decisione di andare in pensione"
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "Summarize for meeting agenda only"
                },
                {
                    "Language": "fr",
                    "Text": "Résumer pour l’ordre du jour de la réunion uniquement"
                },
                {
                    "Language": "uk",
                    "Text": "Підбиття підсумків лише для порядку денного засідання"
                },
                {
                    "Language": "eu",
                    "Text": "Summarize for meeting agenda only"
                },
                {
                    "Language": "it",
                    "Text": "Riepiloga solo per l\'ordine del giorno della riunione"
                }
            ],
            "aaa": [
                {
                    "Language": "en",
                    "Text": "when Zoiberg comes out screaming gluglugu out the door it\'s funny"
                },
                {
                    "Language": "fr",
                    "Text": "quand Zoiberg sort en criant gluglugu à la porte, c’est drôle"
                },
                {
                    "Language": "uk",
                    "Text": "коли Зойберг виходить з криком «Глуглугу за двері», це смішно"
                },
                {
                    "Language": "eu",
                    "Text": "Zoibergek atetik gluglugu oihuka ateratzen denean, barregarria da"
                },
                {
                    "Language": "it",
                    "Text": "quando Zoiberg esce urlando gluglugu fuori dalla porta è divertente"
                }
            ],
            "algo": [
                {
                    "Language": "en",
                    "Text": "Do you think before you speak or do you speak after thinking?"
                },
                {
                    "Language": "fr",
                    "Text": "Pensez-vous avant de parler ou parlez-vous après avoir réfléchi ?"
                },
                {
                    "Language": "uk",
                    "Text": "Ви думаєте, перш ніж говорити, чи ви говорите, подумавши?"
                },
                {
                    "Language": "eu",
                    "Text": "Zuek hitz egin baino lehen pentsatzen duzue ala pentsatu ondoren hitz egiten duzue?"
                },
                {
                    "Language": "it",
                    "Text": "Pensi prima di parlare o parli dopo aver pensato?"
                }
            ]
        },
        {
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Uncertainty as a driver of innovation"
                },
                {
                    "Language": "fr",
                    "Text": "L’incertitude comme moteur de l’innovation"
                },
                {
                    "Language": "uk",
                    "Text": "Невизначеність як рушійна сила інновацій"
                },
                {
                    "Language": "eu",
                    "Text": "Ziurgabetasuna berrikuntzaren eragile gisa"
                },
                {
                    "Language": "it",
                    "Text": "L\'incertezza come motore dell\'innovazione"
                }
            ],
            "summarization": [],
            "aaa": [
                {
                    "Language": "en",
                    "Text": "Hello, how are you? Let\'s do the heavy shopping"
                },
                {
                    "Language": "fr",
                    "Text": "Bonjour, comment vas-tu? Faisons les gros achats"
                },
                {
                    "Language": "uk",
                    "Text": "Привіт, як справи? Давайте займемося важкими покупками"
                },
                {
                    "Language": "eu",
                    "Text": "Kaixo, zer moduz? Erosketa astuna egingo dugu"
                },
                {
                    "Language": "it",
                    "Text": "Ciao come stai? Facciamo lo shopping pesante"
                }
            ],
            "algo": [
                {
                    "Language": "en",
                    "Text": "It\'s very difficult all this"
                },
                {
                    "Language": "fr",
                    "Text": "C’est très difficile tout cela"
                },
                {
                    "Language": "uk",
                    "Text": "Все це дуже складно"
                },
                {
                    "Language": "eu",
                    "Text": "It ́s very difficult todo esto"
                },
                {
                    "Language": "it",
                    "Text": "È molto difficile tutto questo"
                }
            ]
        }
    ]
}
```

### Conclusión
Pone un array vacio en el lugar del meta aun no exitiendo el nodo meta en el json de entrada


# Prueba: Mandar sin idiomas a traducir

Json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
        "uk",
        "eu",
        "it"
    ],
    "translation_fields": [
    ],
    "translation_source": [
        {
            "text": "Reflexiones sobre lo inexplicable y la decisión de retirarse",
            "summarization": "Summarize for meeting agenda only",
            "aaa": "cuando Zoiberg sale gritando gluglugu por la puerta es gracioso",
            "algo": "¿Ustedes piensan antes de hablar o hablan tras pensar?"
        },
        {
            "text": "La incertidumbre como motor de la innovación",
            "summarization": "",
            "aaa": "Hola, ¿que tal estas? Vamos a hacer la compra pesadito",
            "algo": "It´s very difficult todo esto",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            }
        }
    ]
}
```

resultado
```json
{
    "translations": [
        {
            "meta": {}
        },
        {
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            }
        }
    ]
}
```

### Conclusión 
Solo devuelve el meta

# Prueba: poner un codigo de idioma no valido en `translationFromLanguage`

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "patata",
    "translationToLanguages": [
        "en",
        "fr",
        "uk",
        "eu",
        "it"
    ],
    "translation_fields": [
        "text",
        "algo"
    ],
    "translation_source": [
        {
            "text": "Reflexiones sobre lo inexplicable y la decisión de retirarse",
            "summarization": "Summarize for meeting agenda only",
            "aaa": "cuando Zoiberg sale gritando gluglugu por la puerta es gracioso",
            "algo": "¿Ustedes piensan antes de hablar o hablan tras pensar?"
        },
        {
            "text": "La incertidumbre como motor de la innovación",
            "summarization": "",
            "aaa": "Hola, ¿que tal estas? Vamos a hacer la compra pesadito",
            "algo": "It´s very difficult todo esto",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            }
        }
    ]
}
```

da error 
```json
{'taskExecutionID': 23, 'state': 'erroneous', 'errorMessage': 'One or more errors occurred. (Cannot deserialize the current JSON object (e.g. {"name":"value"}) into type \'TranscriptionModule.Transcriptor.TranslationResult[]\' because the type requires a JSON array (e.g. [1,2,3]) to deserialize correctly.\r\nTo fix this error either change the JSON to a JSON array (e.g. [1,2,3]) or change the deserialized type so that it is a normal .NET type (e.g. not a primitive type like integer, not a collection type like an array or List<T>) that can be deserialized from a JSON object. JsonObjectAttribute can also be added to the type to force it to deserialize from a JSON object.\r\nPath \'error\', line 1, position 9.)', 'notifyUrl': 'http://172.29.48.1:5007/notification', 'startDateTime': '2025-10-23T12:41:16.0109955+02:00', 'finishDateTime': '2025-10-23T12:41:16.4088477+02:00', 'progress': -1, 'taskID': 23, 'result': ''}
```

### Conclusión
Mismo resultado que el anterior relacionado con los codigos de idioma

# Prueba: `translationFromLanguage` erroneo pero sin `translation_fields`

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "patata",
    "translationToLanguages": [
        "en",
        "fr",
        "uk",
        "eu",
        "it"
    ],
    "translation_fields": [
        "eu"
    ],
    "translation_source": [
        {
            "text": "Reflexiones sobre lo inexplicable y la decisión de retirarse",
            "summarization": "Summarize for meeting agenda only",
            "aaa": "cuando Zoiberg sale gritando gluglugu por la puerta es gracioso",
            "algo": "¿Ustedes piensan antes de hablar o hablan tras pensar?"
        },
        {
            "text": "La incertidumbre como motor de la innovación",
            "summarization": "",
            "aaa": "Hola, ¿que tal estas? Vamos a hacer la compra pesadito",
            "algo": "It´s very difficult todo esto",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            }
        }
    ]
}
```

resultado
```json
{
    "translations": [
        {
            "meta": {},
            "eu": []
        },
        {
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2",
                "gluglu": 32
            },
            "eu": []
        }
    ]
}
```

### Conclusión
Los errores del idioma se generan al intentar crear el json de la traducción a un idioma no valido o no soportado(teoria mía que es no soportado) por el traductor

# Prueba: Doble campo meta
json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "eu"
    ],
    "translation_fields": [
        "text",
        "summarization"
    ],
    "translation_source": [
        {
            "text": "Reflexiones sobre lo inexplicable y la decisión de retirarse",
            "summarization": "Summarize for meeting agenda only",
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            },
            "meta": {
                "gluglu":32,
                "aaaaaaaaaa":"test"
            }
        },
        {
            "text": "La incertidumbre como motor de la innovación",
            "summarization": "",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2"
            }
        }
    ]
}
```

resultado
```json
{
    "translations": [
        {
            "meta": {
                "gluglu": 32,
                "aaaaaaaaaa": "test"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Reflections on the Unexplained and the Decision to Retire"
                },
                {
                    "Language": "eu",
                    "Text": "Azalezinari eta erretiratzeko erabakiari buruzko gogoetak"
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "Summarize for meeting agenda only"
                },
                {
                    "Language": "eu",
                    "Text": "Summarize for meeting agenda only"
                }
            ]
        },
        {
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Uncertainty as a driver of innovation"
                },
                {
                    "Language": "eu",
                    "Text": "Ziurgabetasuna berrikuntzaren eragile gisa"
                }
            ],
            "summarization": []
        }
    ]
}
```

### Conclusión
Coge el ultimo objeto `meta` del json para asignarlo en la respuesta

# Prueba: Poner un texto en un idioma no mencionado ni en `translationFromLanguage` ni en `translationToLanguages`

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "eu"
    ],
    "translation_fields": [
        "text",
        "summarization"
    ],
    "translation_source": [
        {
            "text": "Les Espagnols sont très espagnols et très espagnols.",
            "summarization": "Los chuches, nos suben hasta el IVA de los chuches",
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            }
        },
        {
            "text": "Todo lo que se refiere a mí y que figura allí y a los compañeros de partido que figuran ahí, no es cierto, salvo alguna cosa que han publicado los medios",
            "summarization": "Un vaso es un vaso y un plato es un plato",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2"
            }
        }
    ]
}
```

resultado
```json
{
    "translations": [
        {
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Les Espagnols sont très espagnols et très espagnols."
                },
                {
                    "Language": "eu",
                    "Text": "Les Espagnols sont très espagnols et très espagnols."
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "The sweets, they even raise the VAT on the sweets"
                },
                {
                    "Language": "eu",
                    "Text": "Txutxeak, txutxeen BEZera igotzen gaituzte"
                }
            ]
        },
        {
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Everything that refers to me and that appears there and to the party colleagues who appear there, is not true, except for something that has been published in the media"
                },
                {
                    "Language": "eu",
                    "Text": "Niri dagokidan eta han agertzen den guztia eta bertan agertzen diren alderdikideei dagokienez, ez da egia, hedabideek argitaratu duten zerbait izan ezik."
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "A glass is a glass and a plate is a plate"
                },
                {
                    "Language": "eu",
                    "Text": "Edalontzi bat edalontzi bat da eta plater bat plater bat da"
                }
            ]
        }
    ]
}
```

### Conclusión
No lo traduce porque no esta en el idioma referido

# Prueba: Mandar un texto en otro idioma, pero referir ese idioma en `translationToLanguages`

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "eu",
        "fr"
    ],
    "translation_fields": [
        "text",
        "summarization"
    ],
    "translation_source": [
        {
            "text": "Les Espagnols sont très espagnols et très espagnols.",
            "summarization": "Los chuches, nos suben hasta el IVA de los chuches",
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            }
        },
        {
            "text": "Todo lo que se refiere a mí y que figura allí y a los compañeros de partido que figuran ahí, no es cierto, salvo alguna cosa que han publicado los medios",
            "summarization": "Todo lo que se refiere a mí y que figura allí y a los compañeros de partido que figuran ahí, no es cierto, salvo alguna cosa que han publicado los medios",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2"
            }
        }
    ]
}
```

resultado
```json
{
    "translations": [
        {
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Les Espagnols sont très espagnols et très espagnols."
                },
                {
                    "Language": "eu",
                    "Text": "Les Espagnols sont très espagnols et très espagnols."
                },
                {
                    "Language": "fr",
                    "Text": "Les Espagnols sont très espagnols et très espagnols."
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "The sweets, they even raise the VAT on the sweets"
                },
                {
                    "Language": "eu",
                    "Text": "Txutxeak, txutxeen BEZera igotzen gaituzte"
                },
                {
                    "Language": "fr",
                    "Text": "Les bonbons, ils augmentent même la TVA sur les bonbons"
                }
            ]
        },
        {
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Everything that refers to me and that appears there and to the party colleagues who appear there, is not true, except for something that has been published in the media"
                },
                {
                    "Language": "eu",
                    "Text": "Niri dagokidan eta han agertzen den guztia eta bertan agertzen diren alderdikideei dagokienez, ez da egia, hedabideek argitaratu duten zerbait izan ezik."
                },
                {
                    "Language": "fr",
                    "Text": "Tout ce qui se réfère à moi et qui apparaît là-bas et aux collègues du parti qui y figurent, n’est pas vrai, sauf quelque chose qui a été publié dans les médias"
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "Everything that refers to me and that appears there and to the party colleagues who appear there, is not true, except for something that has been published in the media"
                },
                {
                    "Language": "eu",
                    "Text": "Niri dagokidan eta han agertzen den guztia eta bertan agertzen diren alderdikideei dagokienez, ez da egia, hedabideek argitaratu duten zerbait izan ezik."
                },
                {
                    "Language": "fr",
                    "Text": "Tout ce qui se réfère à moi et qui apparaît là-bas et aux collègues du parti qui y figurent, n’est pas vrai, sauf quelque chose qui a été publié dans les médias"
                }
            ]
        }
    ]
}
```

### Conclusión
Tampoco lo traduce, pero en la prueba de juan si tradujo una frase en ingles cuando el idioma en `translationFromLanguage` estaba como `es`

# Prueba: mas claves en `translation_source` que en `translation_fields`
json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "eu",
        "fr"
    ],
    "translation_fields": [
        "text",
        "summarization"
    ],
    "translation_source": [
        {
            "text": "Les Espagnols sont très espagnols et très espagnols.",
            "summarization": "Los chuches, nos suben hasta el IVA de los chuches",
            "LosChuches":"Los chuches",
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            }
        },
        {
            "text": "Todo lo que se refiere a mí y que figura allí y a los compañeros de partido que figuran ahí, no es cierto, salvo alguna cosa que han publicado los medios",
            "summarization": "Todo lo que se refiere a mí y que figura allí y a los compañeros de partido que figuran ahí, no es cierto, salvo alguna cosa que han publicado los medios",
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2"
            }
        }
    ]
}
```

resultado
```json
{
    "translations": [
        {
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Les Espagnols sont très espagnols et très espagnols."
                },
                {
                    "Language": "eu",
                    "Text": "Les Espagnols sont très espagnols et très espagnols."
                },
                {
                    "Language": "fr",
                    "Text": "Les Espagnols sont très espagnols et très espagnols."
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "The sweets, they even raise the VAT on the sweets"
                },
                {
                    "Language": "eu",
                    "Text": "Txutxeak, txutxeen BEZera igotzen gaituzte"
                },
                {
                    "Language": "fr",
                    "Text": "Les bonbons, ils augmentent même la TVA sur les bonbons"
                }
            ]
        },
        {
            "meta": {
                "in": 5555,
                "out": 7777,
                "markable_id": 22334,
                "markable_type": "note",
                "annotable_id": 98765,
                "annotable_type": "transcript",
                "speaker": "S2"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Everything that refers to me and that appears there and to the party colleagues who appear there, is not true, except for something that has been published in the media"
                },
                {
                    "Language": "eu",
                    "Text": "Niri dagokidan eta han agertzen den guztia eta bertan agertzen diren alderdikideei dagokienez, ez da egia, hedabideek argitaratu duten zerbait izan ezik."
                },
                {
                    "Language": "fr",
                    "Text": "Tout ce qui se réfère à moi et qui apparaît là-bas et aux collègues du parti qui y figurent, n’est pas vrai, sauf quelque chose qui a été publié dans les médias"
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "Everything that refers to me and that appears there and to the party colleagues who appear there, is not true, except for something that has been published in the media"
                },
                {
                    "Language": "eu",
                    "Text": "Niri dagokidan eta han agertzen den guztia eta bertan agertzen diren alderdikideei dagokienez, ez da egia, hedabideek argitaratu duten zerbait izan ezik."
                },
                {
                    "Language": "fr",
                    "Text": "Tout ce qui se réfère à moi et qui apparaît là-bas et aux collègues du parti qui y figurent, n’est pas vrai, sauf quelque chose qui a été publié dans les médias"
                }
            ]
        }
    ]
}
```

### Conclusión
Ignora las claves que no existen en `translation_fields`

# Prueba: Texto con emoticonos

# Prueba: Texto largo

# Prueba: `translationFromLanguage` vacio

# Prueba: `translation_source` vacio

# Prueba: `translationFromLanguage` como array

# Prueba: Sin `type`

# Prueba: Sin `translationFromLanguage`

# Prueba: Sin `translationToLanguages`

# Prueba: `notifyUrl` invalida

# Prueba: muchos objetos en `translation_source`

# Prueba: Codigos de idioma duplicados en `translationToLanguages`

# Prueba: Duplicados en `translation_fields`