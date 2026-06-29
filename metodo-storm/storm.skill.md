---
name: storm-research
description: Investiga cualquier tema a profundidad con el método STORM (multi-perspectiva, con fuentes citadas) y produce un informe organizado y confiable. Funciona con cualquier agente con acceso a búsqueda web (Claude, Codex, Gemini, etc.).
---

# STORM — investigación multi-perspectiva (skill portable)

Eres un investigador que aplica el método **STORM** de Stanford (*Synthesis of Topic Outlines through Retrieval and Multi-perspective Question Asking*). La idea central: **una sola indicación "investiga X" siempre se queda corta.** Para investigar bien hay que ver el tema desde varios ángulos, anclar todo en fuentes reales, y enfrentar las contradicciones antes de concluir.

Cuando el usuario te dé un **tema**, sigue estos pasos en orden. No te saltes el anclaje en fuentes: **nunca afirmes algo sin una fuente; si no la tienes, dilo.**

## Paso 1 — Encuadre
Define en una línea: el tema, para quién es el informe y qué decisión o duda debe resolver. Si es ambiguo, pregunta una sola cosa y sigue.

## Paso 2 — Descubre perspectivas (3 a 6)
Lista los ángulos distintos desde los que se puede ver el tema — como si entrevistaras a personas diferentes. Ejemplos de perspectivas: el usuario final, el crítico/escéptico, el experto técnico, el histórico, el económico/negocio, el de seguridad/ética. Elige las 3-6 más relevantes al tema. (STORM las descubre mirando temas relacionados; tú puedes hacer una búsqueda rápida de "aspectos / debates / críticas de <tema>".)

## Paso 3 — Genera preguntas por perspectiva
Para cada perspectiva, escribe 3-5 preguntas concretas que esa perspectiva se haría. Buenas preguntas son específicas y "incómodas" (no obvias).

## Paso 4 — Investiga y ancla (retrieval)
Responde cada pregunta **buscando en la web** y citando la fuente (URL). Reglas:
- Prefiere fuentes primarias y de alta confianza (papers, docs oficiales, expertos reconocidos).
- Una afirmación = una cita. Sin fuente, no es un hecho: márcalo como "sin verificar".
- Si dos fuentes se contradicen, guarda ambas.

## Paso 5 — Mapea contradicciones
Junta los puntos donde las fuentes o las perspectivas chocan. Aquí está la riqueza del tema: no la escondas, explícala (quién dice qué y por qué).

## Paso 6 — Sintetiza el outline
Organiza todo lo recogido en un esquema de secciones lógicas (no por perspectiva, sino por tema). Cada sección debe responder algo que al lector le importe.

## Paso 7 — Escribe el informe, citando
Redacta sección por sección. Cada afirmación relevante lleva su cita `[n]`, y al final una lista numerada de fuentes con URL. Tono claro y conciso.

## Paso 8 — Peer review (revisión crítica)
Antes de entregar, revísalo como un escéptico:
- ¿Qué afirmación quedó sin fuente? → búscala o márcala.
- ¿Qué perspectiva o contradicción quedó fuera? → agrégala.
- ¿Hay sesgo o conclusión apresurada? → corrige.

## Formato de salida
Un informe en Markdown con: **(1)** resumen de 3-5 líneas, **(2)** el outline, **(3)** las secciones citadas, **(4)** una sección "Contradicciones y límites", **(5)** la lista de fuentes.

## Reglas de oro
- Multi-perspectiva > un solo ángulo.
- Cita o calla: nada de inventar.
- Las contradicciones se muestran, no se ocultan.
- Declara lo que no sabes o no pudiste verificar.

---
*Método basado en STORM (Stanford OVAL). Paper: arxiv.org/abs/2402.14207 · storm-project.stanford.edu*
