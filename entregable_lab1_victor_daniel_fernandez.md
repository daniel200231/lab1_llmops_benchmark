# Entregable Laboratorio 1: Benchmarking de Serving Local y Cuantización

- **Curso:** LLMOps, Serving Local y Arquitecturas RAG
- **Estudiante:** victor daniel fernandez
- **Hardware Utilizado:** AMD Ryzen 7 7735HS with Radeon Graphics / 15.6 GB RAM (15.611 MB) / Windows 10.0.26200.
- **Fecha:** 13 de noviembre de 2026

---

## Análisis Técnico de Resultados

### 1. Impacto en Memoria (MEM USAGE y KV Cache)
> ¿Cuál fue la diferencia exacta en MB/GB de RAM consumida entre `q4_K_M` y `q8_0` al ejecutar `docker stats`? ¿Por qué el consumo en ejecución supera el tamaño del archivo en disco?

**Respuesta:**
Con docker stats y num_ctx = 2048, el contenedor consumió 1.318 GiB con q4_K_M y 1.933 GiB con q8_0. La diferencia exacta es de 0.615 GiB (≈ 630 MiB, unos 660 MB decimales), y coincide con la diferencia de tamaño de los archivos en disco (986 MB vs 1.6 GB, ≈ 614 MB). Con el contenedor en reposo, sin modelo cargado, docker stats marcaba 294.5 MiB; al restarlo, el modelo cargado aporta ≈ 1.1 GB (q4_K_M) y ≈ 1.8 GB (q8_0), valores que coinciden con la columna SIZE de ollama ps (1.1 GB y 1.8 GB).

Con docker stats (num_ctx = 2048) el contenedor consumió 1.318 GiB con q4_K_M y 1.933 GiB con q8_0, una diferencia de 0.615 GiB (≈ 630 MiB), coherente con la diferencia de tamaño en disco (986 MB vs 1.6 GB). El consumo supera el tamaño del archivo porque, además de los pesos, se reserva el KV Cache del contexto (2048 tokens), más los buffers de cómputo de llama.cpp y el overhead del runtime. El KV Cache depende de num_ctx y no de la cuantización, por lo que la diferencia entre ambos modelos viene casi toda de los pesos que es 8 bits vs 4 bits por parámetro.

---

### 2. Ancho de Banda de Memoria y Velocidad (TPS)
> En inferencia sobre CPU, el cuello de botella suele ser el ancho de banda de la memoria RAM. ¿En qué porcentaje aumentó la velocidad (TPS) al pasar de 8 bits a 4 bits?

**Respuesta:**
El TPS promedio pasó de 15.90 tok/s con q8_0 a 23.85 tok/s con q4_K_M, un aumento de = 50%. Por prompt fue de +45% corto, +62% medio y +44% largo. El prompt medio del q8_0 tuvo una generación anómala un bucle de 20480 tokens; sin él, el aumento es de = 45%. Esto es coherente con que la inferencia en CPU está limitada por el ancho de banda de la RAM: cada token generado exige leer todos los pesos, y q4_K_M pesa = 1.6 veces menos que q8_0 986 MB vs 1.6 GB, así que se leen menos datos por token.

---

### 3. Latencia Inicial (TTFT) y Fase Prefill
> ¿Por qué el `TTFT (ms)` es significativamente más alto en el Prompt Largo que en el Prompt Corto?

**Respuesta:**
El TTFT depende de la longitud del prompt de entrada en fase prefill, donde se procesan todos sus tokens antes de emitir el primero, no de cuánto se genere después. El prompt corto fue el más rápido en ambos modelos 413 ms con q4_K_M y 605 ms con q8_0, mientras que el medio y el largo, de unos 160 caracteres el doble que el corto, tuvieron más TTFT 920 y 676 ms con q4_K_M; 980 y 1082 ms con q8_0. En q4_K_M el prompt largo no fue el más lento, así que la diferencia entre medio y largo puede ser ruido, pues cada valor es una sola ejecución. Además, q8_0 tuvo mayor TTFT que q4_K_M en los tres prompts.

---

### 4. Calidad vs. Eficiencia en `resultados_benchmark.md`
> Revisando las 6 respuestas completas generadas, ¿hubo alguna degradación perceptible en la lógica del código Python o en el razonamiento generado por el modelo de 4 bits frente al de 8 bits?

**Respuesta:**
No observé una degradación atribuible a los 4 bits: ambos modelos tuvieron fallos de calidad similares. Ambos definieron mal la cuantización (como convertir texto en números, en vez de reducir la precisión de los pesos), lo que apunta a una limitación del modelo de 1.5B y no de la cuantización; el q8_0 siguió mejor el formato de "exactamente 3 viñetas", mientras el q4_K_M añadió introducción y resumen. En código, los dos implementan bien la búsqueda binaria pero fallan en las excepciones (el q4_K_M valida todos los elementos de la lista en vez del target y su prueba se corta en el primer ValueError; el q8_0 lanza TypeError pero su prueba solo captura ValueError). En razonamiento ambos fueron genéricos: el q4_K_M dio plantillas repetitivas y el q8_0 alucinó modelos "Claude" y cayó en un bucle de 20480 tokens, aunque es una sola ejecución y no prueba que 8 bits sea peor. En eficiencia, q4_K_M fue mas o menos 50% más rápido y usó mas o menos 0.6 GB menos de RAM sin pérdida de calidad evidente.

---
*Nota: Adjuntar junto con este documento los archivos generados `resultados_benchmark.csv` y `resultados_benchmark.md`.*
