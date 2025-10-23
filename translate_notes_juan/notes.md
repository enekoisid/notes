
# Prueba 1: poner un codigo de idioma no existente en `translationToLanguages`

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


---

# Prueba 2: Ver si es case sensitive

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

---

# Prueba 3: Poner un idioma "no común"

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


---

# Prueba 4: Enviar sin meta

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


---

# Prueba 5: Mandar sin idiomas a traducir

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

---

# Prueba 6: poner un codigo de idioma no valido en `translationFromLanguage`

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

---

# Prueba 7: `translationFromLanguage` erroneo pero sin `translation_fields`

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

---

# Prueba 8: Doble campo meta
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

---

# Prueba 9: Poner un texto en un idioma no mencionado ni en `translationFromLanguage` ni en `translationToLanguages`

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

---

# Prueba 10: Mandar un texto en otro idioma, pero referir ese idioma en `translationToLanguages`

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

---

# Prueba 11: mas claves en `translation_source` que en `translation_fields`
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

---

# Prueba 12: Texto con emoticonos

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
        "eu"
    ],
    "translation_fields": [
        "text",
        "summarization"
    ],
    "translation_source": [
        {
            "text": "¡Hola! 😊 ¿Cómo estás? 🎉 Espero que tengas un día genial! 🌟",
            "summarization": "Resumen con emojis 📝 y símbolos especiales ✨",
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
            "text": "Los emojis son geniales 🚀 y hacen la comunicación más divertida 🎭",
            "summarization": "Emojis en resumen 🎯",
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
                    "Text": "Hello! 😊 How are you? 🎉 I hope you have a great day! 🌟"
                },
                {
                    "Language": "fr",
                    "Text": "Bonjour! 😊 Comment vas-tu? 🎉 J’espère que vous passez une excellente journée\xa0! 🌟"
                },
                {
                    "Language": "eu",
                    "Text": "Kaixo! 😊 Zer moduz zaude? 🎉 Egun bikaina izatea espero dut! 🌟"
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "Summary with emojis 📝 and special ✨ symbols"
                },
                {
                    "Language": "fr",
                    "Text": "Résumé avec emojis 📝 et symboles spéciaux ✨"
                },
                {
                    "Language": "eu",
                    "Text": "Laburpena emoji 📝 eta sinbolo bereziekin ✨"
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
                    "Text": "Emojis are great 🚀 and make communication more fun 🎭"
                },
                {
                    "Language": "fr",
                    "Text": "Les emojis sont géniaux 🚀 et rendent la communication plus amusante 🎭"
                },
                {
                    "Language": "eu",
                    "Text": "Emojiak bikainak 🚀 dira eta komunikazio dibertigarriagoa 🎭 egiten dute"
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "Emojis in summary 🎯"
                },
                {
                    "Language": "fr",
                    "Text": "Emojis en résumé 🎯"
                },
                {
                    "Language": "eu",
                    "Text": "Emojisa labur esanda 🎯"
                }
            ]
        }
    ]
}
```

### Conclusión
No le importan los emoticonos

---

# Prueba 13: Texto largo

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr"
    ],
    "translation_fields": [
        "text",
        "summarization"
    ],
    "translation_source": [
        {
            "text": "Este es un texto extremadamente largo que contiene muchas palabras y frases complejas para probar cómo se comporta el sistema de traducción cuando se enfrenta a textos de gran longitud. El objetivo es verificar si el servicio puede manejar correctamente textos extensos sin generar errores o problemas de rendimiento. Este párrafo continúa con más contenido para asegurar que realmente sea un texto largo y representativo de lo que podría encontrarse en un escenario real de uso. La idea es probar los límites del sistema y ver cómo responde ante este tipo de desafíos técnicos que podrían surgir en situaciones de producción donde los usuarios envían documentos completos o textos muy extensos para su traducción automática.",
            "summarization": "Este es un resumen también muy largo que contiene múltiples oraciones y conceptos complejos para probar el comportamiento del sistema ante textos extensos. El resumen incluye información detallada sobre diversos temas y aspectos que requieren una traducción precisa y coherente. La longitud del texto es intencional para evaluar la capacidad del servicio de manejar contenido de gran volumen sin comprometer la calidad de la traducción o generar errores en el procesamiento.",
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
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
                    "Text": "This is an extremely long text that contains many complex words and phrases to test how the translation system behaves when faced with long texts. The goal is to check if the service can properly handle long texts without generating errors or performance issues. This paragraph continues with more content to ensure that it really is a long text and representative of what could be found in a real scenario of use. The idea is to test the limits of the system and see how it responds to these types of technical challenges that could arise in production situations where users send complete documents or very long texts for machine translation."
                },
                {
                    "Language": "fr",
                    "Text": "Il s’agit d’un texte extrêmement long qui contient de nombreux mots et phrases complexes pour tester le comportement du système de traduction lorsqu’il est confronté à des textes longs. L’objectif est de vérifier si le service peut gérer correctement les textes longs sans générer d’erreurs ou de problèmes de performance. Ce paragraphe se poursuit avec plus de contenu pour s’assurer qu’il s’agit vraiment d’un texte long et représentatif de ce que l’on pourrait trouver dans un scénario réel d’utilisation. L’idée est de tester les limites du système et de voir comment il répond à ce type de défis techniques qui pourraient survenir dans des situations de production où les utilisateurs envoient des documents complets ou des textes très longs pour la traduction automatique."
                }
            ],
            "summarization": [
                {
                    "Language": "en",
                    "Text": "This is also a very long summary that contains multiple sentences and complex concepts to test the behavior of the system in the face of long texts. The summary includes detailed information on various topics and aspects that require an accurate and consistent translation. Text length is intended to assess the service\'s ability to handle high-volume content without compromising translation quality or generating errors in processing."
                },
                {
                    "Language": "fr",
                    "Text": "Il s’agit également d’un très long résumé qui contient de multiples phrases et des concepts complexes pour tester le comportement du système face à de longs textes. Le résumé comprend des informations détaillées sur divers sujets et aspects qui nécessitent une traduction précise et cohérente. La longueur du texte est destinée à évaluer la capacité du service à gérer du contenu volumineux sans compromettre la qualité de la traduction ou générer des erreurs de traitement."
                }
            ]
        }
    ]
}
```

### Conclusión
no tiene problemas con los textos largos, y no ha tenido problemas de rendimiento ni ha tradado mas de lo habitual

---

# Prueba 14: `translationFromLanguage` vacio

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "",
    "translationToLanguages": [
        "en",
        "fr",
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
                    "Text": "Reflections on the Unexplained and the Decision to Retire"
                },
                {
                    "Language": "fr",
                    "Text": "Réflexions sur l’inexpliqué et la décision de prendre sa retraite"
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
                    "Language": "fr",
                    "Text": "Résumer pour l’ordre du jour de la réunion uniquement"
                },
                {
                    "Language": "eu",
                    "Text": "Laburpena bileraren agendarako soilik"
                }
            ]
        }
    ]
}
```

### Conclusión
Detecta el idioma del texto y lo traduce sin problemas, parece mas eficiente esto que poner el idioma para no limitarlo

---

# Prueba 15: `translationFromLanguage` vacio y textos en diferente idioma

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "",
    "translationToLanguages": [
        "en",
        "fr",
        "eu"
    ],
    "translation_fields": [
        "text",
        "summarization"
    ],
    "translation_source": [
        {
            "text": "Riflessioni sull'inspiegabile e sulla decisione di andare in pensione",
            "summarization": "Summarize for meeting agenda only",
            "meta": {
                "in": 2222,
                "out": 4444,
                "markable_id": 12345,
                "markable_type": "note",
                "annotable_id": 67890,
                "annotable_type": "transcript",
                "speaker": "S1"
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
                    "Text": "Reflections on the inexplicable and the decision to retire"
                },
                {
                    "Language": "fr",
                    "Text": "Réflexions sur l’inexplicable et la décision de prendre sa retraite"
                },
                {
                    "Language": "eu",
                    "Text": "Azalezinari buruzko hausnarketak eta erretiroa hartzeko erabakia"
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
                    "Language": "eu",
                    "Text": "Laburpena bileraren agendarako soilik"
                }
            ]
        }
    ]
}
```

### Conclusión
Identifica en efecto el idioma del texto, lo que hace mas eficiente dejarlo vacio que limitarlo a un idioma 'base'


---

# Prueba 16: `translation_source` vacio

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
        "eu"
    ],
    "translation_fields": [
        "text",
        "summarization"
    ],
    "translation_source": []
}
```

resultado
```json
{}
```

### Conclusión
si no hay `translation_source` no hay resultado pero tampoco error

---

# Prueba 17: `translationFromLanguage` como array

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": ["es", "en"],
    "translationToLanguages": [
        "en",
        "fr",
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
            }
        }
    ]
}
```

resultado
```json

```

### Conclusión
Da error de http

---

# Prueba 18: Sin `type`

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
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
            }
        }
    ]
}
```

resultado
```json
{'taskExecutionID': 34, 'state': 'erroneous', 'errorMessage': 'Module  not supported!', 'notifyUrl': 'http://172.29.48.1:5007/notification', 'startDateTime': '2025-10-23T15:40:16.2977677+02:00', 'finishDateTime': '2025-10-23T15:40:16.3344839+02:00', 'progress': -1, 'taskID': 34, 'result': ''}
```

### Conclusión
Devuellve un error de modulo no soportado

---

# Prueba 19: Sin `translationFromLanguage`

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationToLanguages": [
        "en",
        "fr",
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
                    "Text": "Reflections on the Unexplained and the Decision to Retire"
                },
                {
                    "Language": "fr",
                    "Text": "Réflexions sur l’inexpliqué et la décision de prendre sa retraite"
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
                    "Language": "fr",
                    "Text": "Résumer pour l’ordre du jour de la réunion uniquement"
                },
                {
                    "Language": "eu",
                    "Text": "Laburpena bileraren agendarako soilik"
                }
            ]
        }
    ]
}
```

### Conclusión
Lo mismo que mandarlo como string vacío

---

# Prueba 20: Sin `translationToLanguages`

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
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
            }
        }
    ]
}
```

resultado
```json
{'taskExecutionID': 36, 'state': 'erroneous', 'errorMessage': 'One or more errors occurred. (Cannot deserialize the current JSON object (e.g. {"name":"value"}) into type \'TranscriptionModule.Transcriptor.TranslationResult[]\' because the type requires a JSON array (e.g. [1,2,3]) to deserialize correctly.\r\nTo fix this error either change the JSON to a JSON array (e.g. [1,2,3]) or change the deserialized type so that it is a normal .NET type (e.g. not a primitive type like integer, not a collection type like an array or List<T>) that can be deserialized from a JSON object. JsonObjectAttribute can also be added to the type to force it to deserialize from a JSON object.\r\nPath \'error\', line 1, position 9.)', 'notifyUrl': 'http://172.29.48.1:5007/notification', 'startDateTime': '2025-10-23T15:42:24.8023999+02:00', 'finishDateTime': '2025-10-23T15:42:25.0305166+02:00', 'progress': -1, 'taskID': 36, 'result': ''}
```

### Conclusión
Busca una array que no existe

---

# Prueba 21: `notifyUrl` invalida

json de entrada
```json
{
    "notifyUrl": "not-a-valid-url",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
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
            }
        }
    ]
}
```

resultado
```json

```

### Conclusión
No da error y tampoco devuelve nada (logicamente). Solo se guarda en la base de datos

---

# Prueba 22: muchos objetos en `translation_source`

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr"
    ],
    "translation_fields": [
        "text"
    ],
    "translation_source": [
        {
            "text": "Texto 1: Reflexiones sobre lo inexplicable",
            "meta": {"in": 1000, "out": 2000, "markable_id": 1, "markable_type": "note", "annotable_id": 100, "annotable_type": "transcript", "speaker": "S1"}
        },
        {
            "text": "Texto 2: La incertidumbre como motor",
            "meta": {"in": 2000, "out": 3000, "markable_id": 2, "markable_type": "note", "annotable_id": 200, "annotable_type": "transcript", "speaker": "S2"}
        },
        {
            "text": "Texto 3: Innovación y tecnología",
            "meta": {"in": 3000, "out": 4000, "markable_id": 3, "markable_type": "note", "annotable_id": 300, "annotable_type": "transcript", "speaker": "S3"}
        },
        {
            "text": "Texto 4: Desarrollo sostenible",
            "meta": {"in": 4000, "out": 5000, "markable_id": 4, "markable_type": "note", "annotable_id": 400, "annotable_type": "transcript", "speaker": "S4"}
        },
        {
            "text": "Texto 5: Inteligencia artificial",
            "meta": {"in": 5000, "out": 6000, "markable_id": 5, "markable_type": "note", "annotable_id": 500, "annotable_type": "transcript", "speaker": "S5"}
        },
        {
            "text": "Texto 6: Transformación digital",
            "meta": {"in": 6000, "out": 7000, "markable_id": 6, "markable_type": "note", "annotable_id": 600, "annotable_type": "transcript", "speaker": "S6"}
        },
        {
            "text": "Texto 7: Economía circular",
            "meta": {"in": 7000, "out": 8000, "markable_id": 7, "markable_type": "note", "annotable_id": 700, "annotable_type": "transcript", "speaker": "S7"}
        },
        {
            "text": "Texto 8: Sostenibilidad ambiental",
            "meta": {"in": 8000, "out": 9000, "markable_id": 8, "markable_type": "note", "annotable_id": 800, "annotable_type": "transcript", "speaker": "S8"}
        },
        {
            "text": "Texto 9: Innovación social",
            "meta": {"in": 9000, "out": 10000, "markable_id": 9, "markable_type": "note", "annotable_id": 900, "annotable_type": "transcript", "speaker": "S9"}
        },
        {
            "text": "Texto 10: Futuro del trabajo",
            "meta": {"in": 10000, "out": 11000, "markable_id": 10, "markable_type": "note", "annotable_id": 1000, "annotable_type": "transcript", "speaker": "S10"}
        },
        {
            "text": "Texto 11: Blockchain y criptomonedas",
            "meta": {"in": 11000, "out": 12000, "markable_id": 11, "markable_type": "note", "annotable_id": 1100, "annotable_type": "transcript", "speaker": "S11"}
        },
        {
            "text": "Texto 12: Realidad virtual y aumentada",
            "meta": {"in": 12000, "out": 13000, "markable_id": 12, "markable_type": "note", "annotable_id": 1200, "annotable_type": "transcript", "speaker": "S12"}
        },
        {
            "text": "Texto 13: Internet de las cosas",
            "meta": {"in": 13000, "out": 14000, "markable_id": 13, "markable_type": "note", "annotable_id": 1300, "annotable_type": "transcript", "speaker": "S13"}
        },
        {
            "text": "Texto 14: Computación cuántica",
            "meta": {"in": 14000, "out": 15000, "markable_id": 14, "markable_type": "note", "annotable_id": 1400, "annotable_type": "transcript", "speaker": "S14"}
        },
        {
            "text": "Texto 15: Machine learning avanzado",
            "meta": {"in": 15000, "out": 16000, "markable_id": 15, "markable_type": "note", "annotable_id": 1500, "annotable_type": "transcript", "speaker": "S15"}
        },
        {
            "text": "Texto 16: Automatización industrial",
            "meta": {"in": 16000, "out": 17000, "markable_id": 16, "markable_type": "note", "annotable_id": 1600, "annotable_type": "transcript", "speaker": "S16"}
        },
        {
            "text": "Texto 17: Energías renovables",
            "meta": {"in": 17000, "out": 18000, "markable_id": 17, "markable_type": "note", "annotable_id": 1700, "annotable_type": "transcript", "speaker": "S17"}
        },
        {
            "text": "Texto 18: Medicina personalizada",
            "meta": {"in": 18000, "out": 19000, "markable_id": 18, "markable_type": "note", "annotable_id": 1800, "annotable_type": "transcript", "speaker": "S18"}
        },
        {
            "text": "Texto 19: Ciudades inteligentes",
            "meta": {"in": 19000, "out": 20000, "markable_id": 19, "markable_type": "note", "annotable_id": 1900, "annotable_type": "transcript", "speaker": "S19"}
        },
        {
            "text": "Texto 20: Biotecnología moderna",
            "meta": {"in": 20000, "out": 21000, "markable_id": 20, "markable_type": "note", "annotable_id": 2000, "annotable_type": "transcript", "speaker": "S20"}
        },
        {
            "text": "Texto 21: Nanotecnología aplicada",
            "meta": {"in": 21000, "out": 22000, "markable_id": 21, "markable_type": "note", "annotable_id": 2100, "annotable_type": "transcript", "speaker": "S21"}
        },
        {
            "text": "Texto 22: Robótica avanzada",
            "meta": {"in": 22000, "out": 23000, "markable_id": 22, "markable_type": "note", "annotable_id": 2200, "annotable_type": "transcript", "speaker": "S22"}
        },
        {
            "text": "Texto 23: Impresión 3D industrial",
            "meta": {"in": 23000, "out": 24000, "markable_id": 23, "markable_type": "note", "annotable_id": 2300, "annotable_type": "transcript", "speaker": "S23"}
        },
        {
            "text": "Texto 24: Ciberseguridad avanzada",
            "meta": {"in": 24000, "out": 25000, "markable_id": 24, "markable_type": "note", "annotable_id": 2400, "annotable_type": "transcript", "speaker": "S24"}
        },
        {
            "text": "Texto 25: Big data y analytics",
            "meta": {"in": 25000, "out": 26000, "markable_id": 25, "markable_type": "note", "annotable_id": 2500, "annotable_type": "transcript", "speaker": "S25"}
        },
        {
            "text": "Texto 26: Cloud computing híbrido",
            "meta": {"in": 26000, "out": 27000, "markable_id": 26, "markable_type": "note", "annotable_id": 2600, "annotable_type": "transcript", "speaker": "S26"}
        },
        {
            "text": "Texto 27: Edge computing",
            "meta": {"in": 27000, "out": 28000, "markable_id": 27, "markable_type": "note", "annotable_id": 2700, "annotable_type": "transcript", "speaker": "S27"}
        },
        {
            "text": "Texto 28: 5G y conectividad",
            "meta": {"in": 28000, "out": 29000, "markable_id": 28, "markable_type": "note", "annotable_id": 2800, "annotable_type": "transcript", "speaker": "S28"}
        },
        {
            "text": "Texto 29: Vehículos autónomos",
            "meta": {"in": 29000, "out": 30000, "markable_id": 29, "markable_type": "note", "annotable_id": 2900, "annotable_type": "transcript", "speaker": "S29"}
        },
        {
            "text": "Texto 30: Drones y aeronaves no tripuladas",
            "meta": {"in": 30000, "out": 31000, "markable_id": 30, "markable_type": "note", "annotable_id": 3000, "annotable_type": "transcript", "speaker": "S30"}
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
                "in": 1000,
                "out": 2000,
                "markable_id": 1,
                "markable_type": "note",
                "annotable_id": 100,
                "annotable_type": "transcript",
                "speaker": "S1"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 1: Reflections on the Inexplicable"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 1 : Réflexions sur l’inexplicable"
                }
            ]
        },
        {
            "meta": {
                "in": 2000,
                "out": 3000,
                "markable_id": 2,
                "markable_type": "note",
                "annotable_id": 200,
                "annotable_type": "transcript",
                "speaker": "S2"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 2: Uncertainty as a driver"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 2 : L’incertitude comme facteur"
                }
            ]
        },
        {
            "meta": {
                "in": 3000,
                "out": 4000,
                "markable_id": 3,
                "markable_type": "note",
                "annotable_id": 300,
                "annotable_type": "transcript",
                "speaker": "S3"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 3: Innovation and technology"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 3 : Innovation et technologie"
                }
            ]
        },
        {
            "meta": {
                "in": 4000,
                "out": 5000,
                "markable_id": 4,
                "markable_type": "note",
                "annotable_id": 400,
                "annotable_type": "transcript",
                "speaker": "S4"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 4: Sustainable development"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 4 : Développement durable"
                }
            ]
        },
        {
            "meta": {
                "in": 5000,
                "out": 6000,
                "markable_id": 5,
                "markable_type": "note",
                "annotable_id": 500,
                "annotable_type": "transcript",
                "speaker": "S5"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 5: Artificial Intelligence"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 5 : Intelligence artificielle"
                }
            ]
        },
        {
            "meta": {
                "in": 6000,
                "out": 7000,
                "markable_id": 6,
                "markable_type": "note",
                "annotable_id": 600,
                "annotable_type": "transcript",
                "speaker": "S6"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 6: Digital Transformation"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 6 : Transformation numérique"
                }
            ]
        },
        {
            "meta": {
                "in": 7000,
                "out": 8000,
                "markable_id": 7,
                "markable_type": "note",
                "annotable_id": 700,
                "annotable_type": "transcript",
                "speaker": "S7"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 7: Circular economy"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 7 : Économie circulaire"
                }
            ]
        },
        {
            "meta": {
                "in": 8000,
                "out": 9000,
                "markable_id": 8,
                "markable_type": "note",
                "annotable_id": 800,
                "annotable_type": "transcript",
                "speaker": "S8"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 8: Environmental sustainability"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 8 : Durabilité environnementale"
                }
            ]
        },
        {
            "meta": {
                "in": 9000,
                "out": 10000,
                "markable_id": 9,
                "markable_type": "note",
                "annotable_id": 900,
                "annotable_type": "transcript",
                "speaker": "S9"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 9: Social Innovation"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 9 : Innovation sociale"
                }
            ]
        },
        {
            "meta": {
                "in": 10000,
                "out": 11000,
                "markable_id": 10,
                "markable_type": "note",
                "annotable_id": 1000,
                "annotable_type": "transcript",
                "speaker": "S10"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 10: Future of work"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 10\xa0: L’avenir du travail"
                }
            ]
        },
        {
            "meta": {
                "in": 11000,
                "out": 12000,
                "markable_id": 11,
                "markable_type": "note",
                "annotable_id": 1100,
                "annotable_type": "transcript",
                "speaker": "S11"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 11: Blockchain and cryptocurrencies"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 11 : Blockchain et crypto-monnaies"
                }
            ]
        },
        {
            "meta": {
                "in": 12000,
                "out": 13000,
                "markable_id": 12,
                "markable_type": "note",
                "annotable_id": 1200,
                "annotable_type": "transcript",
                "speaker": "S12"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 12: Virtual and augmented reality"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 12 : Réalité virtuelle et augmentée"
                }
            ]
        },
        {
            "meta": {
                "in": 13000,
                "out": 14000,
                "markable_id": 13,
                "markable_type": "note",
                "annotable_id": 1300,
                "annotable_type": "transcript",
                "speaker": "S13"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 13: Internet of Things"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 13 : Internet des objets"
                }
            ]
        },
        {
            "meta": {
                "in": 14000,
                "out": 15000,
                "markable_id": 14,
                "markable_type": "note",
                "annotable_id": 1400,
                "annotable_type": "transcript",
                "speaker": "S14"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 14: Quantum computing"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 14 : L’informatique quantique"
                }
            ]
        },
        {
            "meta": {
                "in": 15000,
                "out": 16000,
                "markable_id": 15,
                "markable_type": "note",
                "annotable_id": 1500,
                "annotable_type": "transcript",
                "speaker": "S15"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 15: Advanced machine learning"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 15 : Apprentissage automatique avancé"
                }
            ]
        },
        {
            "meta": {
                "in": 16000,
                "out": 17000,
                "markable_id": 16,
                "markable_type": "note",
                "annotable_id": 1600,
                "annotable_type": "transcript",
                "speaker": "S16"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 16: Industrial Automation"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 16 : Automatisation industrielle"
                }
            ]
        },
        {
            "meta": {
                "in": 17000,
                "out": 18000,
                "markable_id": 17,
                "markable_type": "note",
                "annotable_id": 1700,
                "annotable_type": "transcript",
                "speaker": "S17"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 17: Renewable energies"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 17\xa0: Énergies renouvelables"
                }
            ]
        },
        {
            "meta": {
                "in": 18000,
                "out": 19000,
                "markable_id": 18,
                "markable_type": "note",
                "annotable_id": 1800,
                "annotable_type": "transcript",
                "speaker": "S18"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 18: Personalized Medicine"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 18 : Médecine personnalisée"
                }
            ]
        },
        {
            "meta": {
                "in": 19000,
                "out": 20000,
                "markable_id": 19,
                "markable_type": "note",
                "annotable_id": 1900,
                "annotable_type": "transcript",
                "speaker": "S19"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 19: Smart Cities"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 19 : Villes intelligentes"
                }
            ]
        },
        {
            "meta": {
                "in": 20000,
                "out": 21000,
                "markable_id": 20,
                "markable_type": "note",
                "annotable_id": 2000,
                "annotable_type": "transcript",
                "speaker": "S20"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 20: Modern biotechnology"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 20\xa0: Biotechnologie moderne"
                }
            ]
        },
        {
            "meta": {
                "in": 21000,
                "out": 22000,
                "markable_id": 21,
                "markable_type": "note",
                "annotable_id": 2100,
                "annotable_type": "transcript",
                "speaker": "S21"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 21: Applied nanotechnology"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 21 : Nanotechnologies appliquées"
                }
            ]
        },
        {
            "meta": {
                "in": 22000,
                "out": 23000,
                "markable_id": 22,
                "markable_type": "note",
                "annotable_id": 2200,
                "annotable_type": "transcript",
                "speaker": "S22"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 22: Advanced Robotics"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 22 : Robotique avancée"
                }
            ]
        },
        {
            "meta": {
                "in": 23000,
                "out": 24000,
                "markable_id": 23,
                "markable_type": "note",
                "annotable_id": 2300,
                "annotable_type": "transcript",
                "speaker": "S23"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 23: Industrial 3D Printing"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 23 : Impression 3D industrielle"
                }
            ]
        },
        {
            "meta": {
                "in": 24000,
                "out": 25000,
                "markable_id": 24,
                "markable_type": "note",
                "annotable_id": 2400,
                "annotable_type": "transcript",
                "speaker": "S24"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 24: Advanced Cybersecurity"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 24 : Cybersécurité avancée"
                }
            ]
        },
        {
            "meta": {
                "in": 25000,
                "out": 26000,
                "markable_id": 25,
                "markable_type": "note",
                "annotable_id": 2500,
                "annotable_type": "transcript",
                "speaker": "S25"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 25: Big data and analytics"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 25 : Mégadonnées et analytique"
                }
            ]
        },
        {
            "meta": {
                "in": 26000,
                "out": 27000,
                "markable_id": 26,
                "markable_type": "note",
                "annotable_id": 2600,
                "annotable_type": "transcript",
                "speaker": "S26"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 26: Hybrid cloud computing"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 26 : Cloud computing hybride"
                }
            ]
        },
        {
            "meta": {
                "in": 27000,
                "out": 28000,
                "markable_id": 27,
                "markable_type": "note",
                "annotable_id": 2700,
                "annotable_type": "transcript",
                "speaker": "S27"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 27: Edge computing"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 27 : Informatique de périphérie"
                }
            ]
        },
        {
            "meta": {
                "in": 28000,
                "out": 29000,
                "markable_id": 28,
                "markable_type": "note",
                "annotable_id": 2800,
                "annotable_type": "transcript",
                "speaker": "S28"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 28: 5G and connectivity"
                },
                {
                    "Language": "fr",
                    "Text": "Texto 28 : 5G et connectivité"
                }
            ]
        },
        {
            "meta": {
                "in": 29000,
                "out": 30000,
                "markable_id": 29,
                "markable_type": "note",
                "annotable_id": 2900,
                "annotable_type": "transcript",
                "speaker": "S29"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 29: Autonomous vehicles"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 29 : Véhicules autonomes"
                }
            ]
        },
        {
            "meta": {
                "in": 30000,
                "out": 31000,
                "markable_id": 30,
                "markable_type": "note",
                "annotable_id": 3000,
                "annotable_type": "transcript",
                "speaker": "S30"
            },
            "text": [
                {
                    "Language": "en",
                    "Text": "Text 30: Drones and unmanned aircraft"
                },
                {
                    "Language": "fr",
                    "Text": "Texte 30 : Drones et aéronefs sans pilote"
                }
            ]
        }
    ]
}
```

### Conclusión
ha tardado un poco mas (logicamente) pero no ha tenido ningun problema

---

# Prueba 23: Codigos de idioma duplicados en `translationToLanguages`

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
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
                    "Text": "Reflections on the Unexplained and the Decision to Retire"
                },
                {
                    "Language": "fr",
                    "Text": "Réflexions sur l’inexpliqué et la décision de prendre sa retraite"
                },
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
                    "Language": "fr",
                    "Text": "Résumer pour l’ordre du jour de la réunion uniquement"
                },
                {
                    "Language": "en",
                    "Text": "Summarize for meeting agenda only"
                },
                {
                    "Language": "eu",
                    "Text": "Summarize for meeting agenda only"
                }
            ]
        }
    ]
}
```

### Conclusión
duplica el resultado del idioma, no hace un distinct

---

# Prueba 24: Duplicados en `translation_fields`

json de entrada
```json
{
    "notifyUrl": "{{notificationUrl}}",
    "type": "azureTranslation",
    "translationFromLanguage": "es",
    "translationToLanguages": [
        "en",
        "fr",
        "eu"
    ],
    "translation_fields": [
        "text",
        "summarization",
        "text",
        "algo",
        "summarization"
    ],
    "translation_source": [
        {
            "text": "Reflexiones sobre lo inexplicable y la decisión de retirarse",
            "summarization": "Summarize for meeting agenda only",
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
                    "Text": "Reflections on the Unexplained and the Decision to Retire"
                },
                {
                    "Language": "fr",
                    "Text": "Réflexions sur l’inexpliqué et la décision de prendre sa retraite"
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
                    "Language": "fr",
                    "Text": "Résumer pour l’ordre du jour de la réunion uniquement"
                },
                {
                    "Language": "eu",
                    "Text": "Summarize for meeting agenda only"
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
                    "Language": "eu",
                    "Text": "Zuek hitz egin baino lehen pentsatzen duzue ala pentsatu ondoren hitz egiten duzue?"
                }
            ]
        }
    ]
}
```

### Conclusión
No duplica los resultados a diferencia de lo que pasa con los idiomas