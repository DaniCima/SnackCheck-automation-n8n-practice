# SnackCheck

Automatización en n8n que recibe el código de barras de un alimento, consulta Open Food Facts y devuelve un veredicto con indicadores nutricionales y un párrafo generado mediante Groq.

## Archivos de entrega

- `README.md`: funcionamiento, configuración y uso.
- `NODOS.md`: fichas de cada nodo para copiar en Sticky Notes.
- `PRUEBAS.md`: casos, resultados esperados y registro de ejecución.
- `CHANGELOG.md`: historial de cambios.
- `snackcheck.json`: exportación del flujo de n8n, pendiente de incorporar.
- `snackcheck.excalidraw` y/o `snackcheck-diagrama.png`: diagrama, pendiente de incorporar.

## Estado

Documentación basada en la construcción guiada. La exportación final del flujo todavía no se ha revisado. Los resultados de prueba sin evidencia registrada figuran como pendientes.

**Requisito pendiente de verificar:** el ejercicio exige Merge en modo Append. La unión final de datos y texto se configuró como Combine → By Position. Antes de entregar, incorporar un Merge Append con una función justificada en el flujo y comprobar su ejecución. No cambiar la unión final a Append sin adaptar el tratamiento de sus dos ítems.

## Configuración

1. Importar `snackcheck.json` en n8n cuando esté disponible.
2. Seleccionar una credencial de Groq en el nodo de modelo. No incluir la clave en archivos ni capturas.
3. Revisar el método, la ruta y la ubicación del campo `barcode` en el Webhook. Estos detalles deben completarse a partir del flujo exportado.
4. Configurar el Webhook para responder mediante Respond to Webhook.
5. Probar con la URL de test mientras el Webhook escucha. Para uso continuado, publicar/activar el flujo y utilizar la URL de producción que muestre n8n.

Entrada conceptual; adaptar su envío al método configurado:

```json
{"barcode":"3017620422003"}
```

El código se conserva como texto para mantener posibles ceros iniciales. Se exige que contenga únicamente dígitos: `^[0-9]+$`.

## Recorrido del flujo

1. Webhook recibe la petición.
2. IF comprueba que existe el código; si falta, devuelve instrucciones de uso.
3. IF valida que contiene únicamente dígitos.
4. HTTP Request consulta Open Food Facts mediante GET, sin autenticación.
5. IF comprueba que el producto fue encontrado.
6. IF valida azúcar, sal y grasa: campos presentes, no vacíos, numéricos y no negativos. Cero es válido.
7. Edit Fields prepara nombre, marca, nutrientes, energía y Nutri-Score.
8. Edit Fields calcula semáforos. Code, en modo Run Once for Each Item, cuenta los nutrientes altos sin recorrer un arreglo.
9. Edit Fields calcula las dos clasificaciones y elige la peor.
10. Basic LLM Chain con Groq redacta el párrafo sin decidir la clasificación.
11. Merge Combine → By Position reúne los datos y el texto, un ítem por entrada.
12. Edit Fields prepara el JSON plano. Respond to Webhook devuelve la respuesta.

## Fuente y tratamiento de datos

```text
GET https://world.openfoodfacts.org/api/v2/product/{barcode}.json?fields=product_name,brands,nutriscore_grade,nutriments
```

El flujo se construyó con la respuesta completa del HTTP Request: el cuerpo está en `$json.body`, el producto en `$json.body.product` y los nutrientes en `$json.body.product.nutriments`. El estado de producto encontrado se consulta en `$json.body.status` y debe ser `1`.

Activar Never Error permite tratar las respuestas HTTP como 404. Esto no sustituye una rama para fallos de red, tiempo de espera o servicio externo; comprobar su tratamiento antes de entregar.

Campos: `sugars_100g`, `salt_100g`, `fat_100g` y `nutriments['energy-kcal_100g']`. La energía no participa en la clasificación; si falta, se muestra como dato no disponible.

## Reglas

| Nutriente por 100 g | Bajo / no alto | Medio | Alto |
|---|---|---|---|
| Azúcar | < 5 g | 5 a 22,5 g, inclusive | > 22,5 g |
| Sal | ≤ 1,5 g | No definido en el ejercicio | > 1,5 g |
| Grasa | ≤ 17,5 g | No definido en el ejercicio | > 17,5 g |

Para sal y grasa, el flujo denomina `bajo` al caso no alto; el ejercicio no define un umbral intermedio.

| Lectura | Saludable | Moderado | No saludable |
|---|---|---|---|
| Conteo de nutrientes altos | 0 | 1 | 2 o 3 |
| Nutri-Score | a, b | c | d, e |

El veredicto es el peor de las dos lecturas. Si Nutri-Score falta o es `unknown`, decide el conteo. Si falta alguno de los tres nutrientes, el flujo responde con datos insuficientes y no clasifica. La puntuación de preocupación es el conteo de altos, de 0 a 3; no sustituye el veredicto combinado.

## IA y salida

Groq recibe los datos, los niveles por nutriente y el veredicto calculado. Una expresión en el prompt selecciona la instrucción de tono según `verdict`. El modelo debe mencionar el veredicto, usar un tono alentador para saludable y moderado, y recomendar con delicadeza el consumo ocasional para no saludable. En este último caso explica qué nutrientes están altos o, si no hay ninguno, indica que el resultado se debe al Nutri-Score. El prompt también pide respetar los datos y evitar lenguaje promocional.

El JSON final contiene `headline`, `sugar`, `salt`, `fat`, `energy`, `nutriscore`, `concern_score` y `paragraph`. Incluye marcadores verde, ámbar y rojo, y unidades por 100 g. El cliente consumidor es responsable de presentar estos campos de forma legible.

| Situación | HTTP | Respuesta |
|---|---|---|
| Sin código | 400 | Explicación de SnackCheck e instrucciones |
| Código inválido | 400 | Solicitud de un código con solo dígitos |
| Producto no encontrado | 404 | Mensaje útil para revisar el código o probar otro |
| Nutrientes insuficientes | 422 | No se puede evaluar el producto |
| Veredicto disponible | 200 | Titular, cifras con marcadores y párrafo |

## Validación y limitaciones

Consultar [PRUEBAS.md](PRUEBAS.md). Probar tanto peticiones reales como límites con datos fijados temporalmente en n8n. Restaurar los nodos antes de exportar. La calidad de la información depende de Open Food Facts; el texto de IA requiere revisión con los tres niveles de veredicto.

El texto del ejercicio recibido termina en «Entrégalo como un pro» y no contiene una lista detallada posterior de entregables. Confirmar si existe otra sección antes de cerrar la entrega.
