# División de Honor Juvenil

Base de datos de estadísticas de jugadores construida a partir de las actas
arbitrales oficiales de la Real Federación Española de Fútbol, con una capa
de análisis de vídeo encima, un cuadro de mando en Power BI sobre ambas y un
generador de informes de ojeo en PDF.

Temporada 2026-2027, los siete grupos de División de Honor.

El grupo 7 —Comunidad Valenciana, Murcia y Baleares— es el que tiene la
curación manual completa y sobre el que se ha montado el análisis de vídeo.

| Comunidad Valenciana | Murcia | Baleares |
|---|---|---|
| Valencia CF | Real Murcia CF | RCD Mallorca |
| Villarreal C.F. | FC Cartagena | Penya Arrabal |
| Levante U.D. | UCAM Universidad Católica de Murcia CF | |
| Elche CF | Murcia Promises Club de Fútbol | |
| CD Castellón | | |
| Torrent C.F. | | |
| Kelme C.F. | | |
| C.D. Roda | | |
| Patacona C.F. | | |
| C.D. San Francisco | | |

## Por qué existe

No hay estadísticas públicas de fútbol juvenil español más allá de goles y
partidos jugados. Ningún proveedor de datos de eventos (pases, disparos,
duelos) cubre estas categorías, y las webs de resultados se limitan a
clasificaciones y goleadores.

Las actas arbitrales, en cambio, son públicas y contienen alineaciones con
dorsal, sustituciones con minuto exacto, goles y tarjetas. A partir de ahí se
pueden derivar métricas que no publica nadie: minutos reales por jugador,
goles por 90 minutos, ratio de titularidad, rotación por equipo, reparto de
minutos por año de nacimiento y —reconstruyendo el marcador minuto a minuto—
en qué situación de partido juega cada jugador sus minutos.

Y donde el acta no llega, llega el vídeo. Varios clubes suben sus partidos a
YouTube, así que las acciones se etiquetan a mano sobre la grabación y se
cruzan con el acta por dorsal. Eso da pases, duelos, presiones y zonas de
intervención para los partidos analizados: datos de evento en una categoría
donde no existen.

Encima de las dos, una tercera capa con lo que aporta el analista: notas por
competencia del puesto, veredicto, nivel actual y potencial, y los clips que
lo sostienen. De ahí sale el PDF de una página que es lo que se entrega a un
club.

Este proyecto convierte las actas, los partidos etiquetados y los informes en
una base de datos relacional consultable.

## Tres decisiones del código

El código vive en un repositorio privado (ver más abajo), así que estos tres
fragmentos son el resumen de cómo está escrito. Son los tres sitios donde una
decisión pequeña cambiaba la fiabilidad de todo lo que viene detrás.

**Traducir el reloj de la grabación al reloj del partido.** La herramienta de
etiquetado devuelve el segundo del vídeo, no el minuto de juego: durante el
descanso el vídeo corre y el partido no. Hacen falta dos instantes, el saque
inicial y la reanudación, y de ahí sale un desplazamiento distinto para cada
parte.

```python
def a_minuto(seg: int, pitido1: int, pitido2: int) -> tuple[int, int]:
    """Devuelve (parte, minuto de juego). El acta cuenta minutos cumplidos."""
    if seg < pitido2:
        return 1, max((seg - pitido1) // 60, 0)
    return 2, 45 + max((seg - pitido2) // 60, 0)
```

La división entera no es descuido: el acta anota el minuto cumplido, así que a
los 59 segundos de juego sigue siendo el minuto 0. Y el `max(..., 0)` evita que
una etiqueta puesta antes del saque inicial produzca un minuto negativo.

Lo que valida esta función no es un test escrito por mí, es el acta: registra
una amonestación en el minuto 39, y el tag de falta cometida de ese mismo
jugador, traducido, cae en el 39:03. Dos mediciones independientes del mismo
suceso con tres segundos de diferencia. Ese caso está fijado en la suite para
que un cambio en la traducción no pueda romperlo sin que se note.

**Localizar un partido en el HTML por su contenido, no por su posición.** El
listado de la jornada no trae identificadores: para saber a qué partido
pertenece un enlace de acta hay que subir por el árbol del documento. Subir
cuatro niveles fijos funciona con el maquetado de hoy, pero basta un div de más
—una etiqueta de aplazado, un icono de televisión— para que caigan en la fila
de los escudos. Y entonces el partido entra igual, pero sin fecha y sin
estadio, y el `ON CONFLICT DO UPDATE` las machaca con NULL sin que nada se
queje.

```python
candidatos = []
nodo = a
for _ in range(NIVELES):
    nodo = nodo.parent
    if nodo is None:
        break
    if _escudos(nodo) == 2:
        candidatos.append(nodo)
    elif candidatos:
        break

if not candidatos:
    return None
for c in candidatos:
    if RE_FECHA.search(c.get_text(" ", strip=True)):
        return c
return candidatos[-1]
```

Exactamente **dos** escudos, porque en cuanto aparecen más ya se está metiendo
el partido vecino. Y de los candidatos válidos, el más pequeño que **ya trae la
fecha**: el más alto, en una página de un solo partido, sería la página entera
y el menú acabaría dentro del nombre del estadio.

Este cambio no se aplicó a ojo. Se escribió la versión nueva al lado de la
vieja y se pasaron las dos por los listados guardados en disco —doce páginas,
setenta y cinco partidos— antes de tocar nada. Los setenta y cinco daban el
mismo bloque. Solo entonces se sustituyó.

**Derivar en lugar de leer, para que los datos no puedan contradecirse.** De
cada jugador se guardan los minutos y el tramo que estuvo en el campo. El tramo
podría leerse de la sustitución, pero entonces habría tres columnas —entrada,
salida y minutos— que pueden decir cosas distintas.

```python
# un suplente que no entro no tiene tramo: se deja en NULL para
# que se distinga de uno que entro y jugo cero minutos
if mins == 0 and not j["titular"] and entra is None:
    j["min_entra"] = None
    j["min_sale"] = None
else:
    j["min_entra"] = 0 if j["titular"] else entra
    j["min_sale"] = (None if j["min_entra"] is None
                     else j["min_entra"] + mins)
```

Al derivar `min_sale`, la invariante `min_sale = min_entra + minutos` se cumple
siempre, pase lo que pase con los cambios, las expulsiones y el descuento. La
suite comprueba esa propiedad sobre varias combinaciones en lugar de comprobar
casos concretos.

Y el NULL de las dos primeras líneas es deliberado: un suplente que no entró y
uno que entró en el minuto 90 tienen los mismos cero minutos, pero no son lo
mismo, y dejar el tramo a cero haría imposible distinguirlos después.

## Qué contiene

```sql
-- estado de la base en cualquier momento
SELECT competicion, grupo,
       COUNT(DISTINCT cod_acta) AS partidos,
       MAX(jornada)             AS ultima_jornada
FROM dh.partidos WHERE acta_parseada
GROUP BY competicion, grupo ORDER BY competicion, grupo;
```

Dieciocho tablas y seis vistas, en tres capas.

### Capa de acta

| Tabla | Contenido |
|---|---|
| `equipos` | Clubes, con su código federativo, escudo en base64 y colores |
| `partidos` | Calendario y resultados, una fila por acta |
| `jugadores` | Ficha por jugador: posición, año, pierna hábil, altura, contrato |
| `jugadores_alias` | Otras grafías del mismo nombre, para que el cruce no dependa de una |
| `participaciones` | Una fila por jugador convocado y partido, con minutos y tramo en el campo |
| `eventos` | Goles, amarillas y rojas, con minuto |
| `v_jugador_temporada` | Agregado por jugador: dimensión de jugador para Power BI |
| `v_partidos_equipo` | Una fila por equipo y partido, con los goles ya orientados |
| `v_clasificacion` | Clasificación derivada, con desglose local/visitante y rankings |
| `v_estado_partido` | Tramos de marcador de cada partido, desde la perspectiva de cada equipo |
| `v_minutos_por_estado` | Minutos de cada jugador según si su equipo iba por delante, empatado o por detrás |

### Capa de vídeo

| Tabla | Contenido |
|---|---|
| `videos` | Una grabación, con los dos instantes de pitido que calibran su reloj |
| `acciones` | Una fila por acción etiquetada, con minuto de juego, zona y enlace al clip |
| `acciones_catalogo` | Diccionario de botones: categoría, fase y si la acción es acierto, fallo o neutra |
| `zonas` | Rejilla 3×5 del campo, con banda, franja y coordenadas |
| `v_videos` | Etiqueta legible de cada grabación, para los segmentadores |

### Capa de ojeo

Lo que aporta el analista, separado de lo que miden las otras dos capas.

| Tabla | Contenido |
|---|---|
| `informes` | Una fila por jugador y partido, o general de temporada: texto, veredicto, nivel actual y potencial, recomendación, y las tres frases (*lo que es*, *lo que le falta*, *lo que condiciona*) |
| `valoraciones` | Notas de 1 a 5 por aspecto, en bloques técnico, táctico, físico y mental |
| `puntos` | Fortalezas, debilidades y puntos de trabajo |
| `posiciones` | Las ocho posiciones evaluables, con su orden |
| `competencias` | Las competencias de cada puesto, con su definición: cincuenta en total |
| `valoraciones_competencia` | Nota de 1 a 5 y comentario por informe y competencia |
| `niveles` | Escala de nivel, de «titular en 1ª División» a «no llega a División de Honor» |
| `clips_informe` | Los clips elegidos para un informe, atados a la competencia que demuestran |

La diferencia entre `valoraciones` y `valoraciones_competencia` es la que
separa describir a un jugador de evaluarlo. Los dieciséis aspectos son
atributos —velocidad, concentración, golpeo— y valen igual para cualquier
puesto. Las competencias son tareas del puesto: lo que se le pide a un
central no es lo mismo que lo que se le pide a un extremo, y un central que
puntúa alto en todos los atributos puede seguir sin defender bien el espacio.
El informe que se entrega va por competencias; los atributos se quedan en el
cuadro de mando, como radar.

## Arquitectura

```
rfef.es/es/resultados          →  listado de partidos por jornada
        ↓                          (equipos, marcador, fecha, enlace al acta)
resultados.rfef.es             →  acta completa de cada partido
        ↓
bronze/                        →  HTML crudo guardado en disco
        ↓
parser_listado.py              →  partidos, fecha, estadio, marcador
parser_acta.py                 →  alineaciones, sustituciones, goles, tarjetas
        ↓
carga_listado.py               →  upsert idempotente en PostgreSQL
carga_acta.py
        ↓                                          ↑
        ↓                              acciones.py │ CSV de tagging de vídeo
        ↓                                          │ (cruce por dorsal)
        ↓                                          ↑
        ↓                               informe.py │ notas, competencias,
        ↓                                          │ veredicto y clips
        ↓
v_jugador_temporada            →  capa de consumo
v_minutos_por_estado
        ↓                   ↘
   Power BI                  informe_pdf.py  →  salida/informes/*.pdf
   cuadro de mando           (plantillas/informe.html + wkhtmltopdf)
```

`pipeline.py` es el único módulo que hace peticiones a la red; los parsers no
tocan la base de datos y los cargadores no tocan la red. Esa separación es lo
que permite probar el parseo contra un HTML guardado, sin servidor y sin
base de datos.

La capa `bronze/` guarda el HTML tal como llega. Cambiar la lógica de parseo y
reprocesar todas las actas no cuesta ni una petición al servidor: reprocesar
451 actas desde disco tarda tres minutos. Esa decisión se pagó sola el día que
apareció un fallo en el parseo de goles. Toda la carga usa
`ON CONFLICT DO UPDATE`, así que el pipeline es idempotente: se puede relanzar
sin duplicar datos.

La idempotencia protege de duplicar y no protege de empeorar. Un `DO UPDATE`
puede machacar un valor bueno con un `NULL` si el parser deja de leer un
campo, y eso no se queja en ningún sitio. Por eso `carga_listado.py` compara
antes y después los campos que le importan y avisa de lo que se ha perdido en
lugar de darlo por bueno.

El modelo de Power BI es una estrella: `equipos`, `v_jugador_temporada`,
`partidos`, `zonas` y `acciones_catalogo` son dimensiones, y `participaciones`,
`eventos` y `acciones` son las tablas de hechos. Las métricas no están
pre-agregadas, de modo que la misma medida sirve para un partido o para la
temporada entera según lo que filtren los segmentadores.

## El cuadro de mando

Cuatro páginas, cada una con un grano distinto.

**Equipo.** Clasificación del grupo, goles a favor y en contra con desglose
local/visitante y ranking dentro del grupo, racha de los últimos partidos con
el rival, plantilla completa y reparto de minutos. La tabla de clasificación
funciona además como selector.

**Jugador.** Ficha de temporada: minutos por jornada, porcentaje de los
minutos posibles, perfil técnico y táctico a partir del vídeo, radar de
valoración en dieciséis aspectos, panel de fortalezas y debilidades, e informe
en texto.

**Vídeo.** Un partido etiquetado: desglose de acciones por categoría, mapa de
zonas de intervención, volumen por tramos de quince minutos y tabla de clips
con enlace al segundo exacto de la grabación.

**Exploración.** Todos los jugadores de la base con filtros por competición,
grupo, equipo, posición, año, minutos, partidos y goles.

Los escudos y los colores de cada club salen de la base: el escudo va
codificado en base64 dentro del modelo, así que el informe funciona sin
conexión y no depende de ningún servidor externo.

## El informe de ojeo

El cuadro de mando es para explorar. El informe es para entregar, y son dos
problemas distintos: un jefe de scouting no dedica veinte minutos a leer un
informe. Así que el PDF se lee en capas, y cada capa tiene su tiempo.

- **Cinco segundos.** Recomendación —fichar, seguir de cerca, seguir,
  descartar—, nivel actual y nivel potencial. El nivel no es una nota de 1 a
  10 sino una frase operativa: «titular en División de Honor», «suplente en
  2ª RFEF». Dice en qué sitio del fútbol encaja hoy y en qué sitio puede
  encajar.
- **Treinta segundos.** Tres frases: lo que es, lo que le falta, lo que
  condiciona la lectura. La tercera es la que separa un informe honesto de un
  folleto: un central que defiende el espacio en un equipo que presiona muy
  alto no está resolviendo el mismo problema que en un equipo que se repliega.
- **Dos minutos.** Las competencias del puesto, con nota de 1 a 5 y una línea
  cada una, y debajo de cada competencia el clip que la demuestra, enlazado al
  segundo exacto.
- **El resto.** Segunda página: minutos por jornada, minutos según el
  marcador, perfil de vídeo y el informe escrito completo.

Lo cuantitativo sale de la base, lo escrito sale de `informe.py` y el PDF solo
compone. La plantilla vive en `plantillas/informe.html` y se renderiza con
wkhtmltopdf, cuyo motor es un WebKit viejo: sin CSS grid y con flexbox
inestable, de modo que la maquetación es de tablas y flotados a propósito.

Dos detalles que parecen decoración y no lo son. El color del club se usa como
acento solo si se lee sobre blanco: el amarillo del Villarreal tiene una razón
de contraste de 1,3 contra un mínimo de 4,5, así que como texto es ilegible y
se sustituye por el azul oscuro del proyecto, mientras el amarillo se queda en
la chapa de la posición, que lleva su propio color de texto. Y las notas bajas
se marcan en ámbar en vez de rojo, porque el rojo y el verde son el par que no
distingue un lector dáltropo.

## Tests

```bash
pytest
```

Ciento ocho casos, y ninguno es un test de cobertura: cada uno corresponde a
un fallo que de verdad ha mordido.

| Fichero | Cubre |
|---|---|
| `tests/test_reloj.py` | La traducción del reloj del vídeo al minuto de juego, incluido el caso que validó la calibración: el acta da una amarilla en el 39' y el tag cae en 39:03 |
| `tests/test_acta.py` | Los siete casos de minutos, el cambio anotado en el 99' que daba minutos negativos, y la invariante `min_sale = min_entra + minutos` sobre cuatro combinaciones de cambio, roja y descuento |
| `tests/test_listado.py` | Sobre todo fallar bien: los tres modos en los que una jornada puede no dar partidos, y que nunca se devuelva una lista vacía sin decir por qué |
| `tests/test_informe.py` | Los parsers del editor, y que una plantilla devuelta en blanco no borre un veredicto ya escrito |
| `tests/test_pdf.py` | El contraste WCAG, el tinte de la banda y que un `<` escrito a mano no acabe siendo etiqueta en el PDF |

Los tests no se escribieron para presumir de disciplina, sino porque los tres
fallos más caros del proyecto fueron silenciosos: datos que entraban mal sin
que nada se quejara. Un test es la única forma de que un fallo así no vuelva.

`herramientas/comparar_bloque.py` es un caso aparte y vale la pena mirarlo: no
es un test sino un comparador. Cuando hubo que cambiar la forma de localizar el
bloque HTML de un partido, se escribió la versión nueva al lado de la vieja y
se pasaron las dos por los listados guardados en `bronze/` —doce páginas,
setenta y cinco partidos— antes de tocar nada. Los setenta y cinco daban el
mismo bloque. Solo entonces se aplicó el cambio.

## Cómo se opera

Python 3.14, PostgreSQL 17 y wkhtmltopdf para generar los PDF. Todo se opera
desde la línea de comandos, y estas son las órdenes reales del sistema.

Actualización semanal, un solo comando por grupo:

```bash
python src/pipeline.py --grupo 7
python src/pipeline.py --solo-actas      # reparsea lo pendiente sin pedir nada
```

Lee del calendario qué jornadas se han jugado, actualiza el listado, y
descarga y parsea las actas que falten. Solo pide al servidor lo que no tiene
ya en `bronze/`, con cinco segundos de pausa entre peticiones.

Enriquecimiento y curación:

```bash
python src/posiciones.py --descargar          # posiciones de lapreferente
python src/posiciones.py --cargar
python src/importar_curado.py fichero.csv --simular
python src/escudos.py images --paleta         # ver los colores de cada escudo
python src/escudos.py images                  # escudo y colores a la base
```

Análisis de vídeo:

```bash
python src/acciones.py partido.csv --acta 70694259 \
    --video "https://www.youtube.com/watch?v=..." \
    --equipo villarreal --pitido1 411 --pitido2 4310 --simular
```

Informes de ojeo. Todos abren el editor con una plantilla precargada —en el
caso del informe de partido, con la ficha de ese partido como cabecera— y
guardan al cerrarlo:

```bash
python src/informe.py --jugador vivo --partidos       # ver sus partidos
python src/informe.py --jugador vivo --acta 70694259  # informe de un partido
python src/informe.py --jugador vivo                  # informe de temporada
python src/informe.py --jugador vivo --valorar        # los 16 aspectos, 1 a 5
python src/informe.py --jugador vivo --puntos         # fortalezas y debilidades
python src/informe.py --jugador vivo --competencias --posicion central
python src/informe.py --jugador vivo --veredicto      # nivel y recomendación
python src/informe.py --jugador vivo --clips          # elegir los clips
python src/informe.py --listar
```

El PDF:

```bash
python src/informe_pdf.py --jugador vivo
python src/informe_pdf.py --jugador vivo --foto ruta\a\la\foto.jpg
python src/informe_pdf.py --jugador vivo --acta 70694259   # el de un partido
python src/informe_pdf.py --equipo villarreal              # todos los escritos
```

## Estructura del proyecto

```
src/
  pipeline.py          orquestador: el único que pide algo a la red
  parser_listado.py    partidos de una jornada, a partir del HTML
  parser_acta.py       alineaciones, sustituciones, goles, tarjetas y minutos
  carga_listado.py     upsert de partidos y equipos
  carga_acta.py        upsert de jugadores, participaciones y eventos
  acciones.py          importa el CSV de tagging y lo cruza con el acta
  escudos.py           escudo y colores de cada club a la base
  importar_curado.py   devuelve a la base el CSV de curación manual
  informe.py           todo lo que se escribe a mano, con el editor
  informe_pdf.py       compone el PDF de un informe
  depurar_goles.py     diagnóstico del parseo de goles, en tres pasadas
sql/
  schema_division_honor.sql   el esquema completo, idempotente
plantillas/
  informe.html         la plantilla del PDF
tests/                 108 casos, uno por fallo real
herramientas/
  comparar_bloque.py   comparador de dos versiones de un parser sobre bronze/
bronze/                HTML crudo (no se versiona)
salida/informes/       los PDF generados (no se versionan)
```

## Cómo se cruza el vídeo con el acta

La herramienta de tagging exporta el reloj de la **grabación**, no el del
partido: durante el descanso el vídeo sigue corriendo y el partido no. El
último tag del Villarreal–Elche está en el minuto 119 de vídeo.

La traducción necesita dos instantes, el saque inicial y la reanudación, y de
ahí sale un desplazamiento distinto para cada parte. Esos dos valores se
guardan en `videos`, porque pertenecen a la grabación y no al partido.

Lo interesante es que **el acta calibra el vídeo gratis**: un cambio con minuto
conocido basta para fijar la segunda parte. Y una vez calibrado, las dos
fuentes se validan entre sí. En el partido analizado, el acta registra una
amarilla a un jugador en el minuto 39; el tag de falta cometida de ese mismo
jugador, traducido, cae en el 39:03. Dos fuentes independientes contando el
mismo suceso con tres segundos de diferencia. Ese caso está fijado en
`tests/test_reloj.py`, para que un cambio en la traducción del reloj no pueda
romperlo sin que se note.

El cruce de jugadores va por **dorsal dentro del partido**, no por nombre, y el
nombre se usa solo como verificación: si el dorsal apunta a alguien cuyo nombre
no encaja, el importador avisa en lugar de callarse.

## Notas sobre los datos

Cosas que conviene saber antes de usar estas cifras.

**Minutos.** El acta no recoge el tiempo de descuento, así que 90 minutos es
una convención para quien juega el partido completo, y el convenio se aplica
igual en todas partes: los cambios y las rojas anotados en el descuento se
topan en 90, y la suma de minutos por estado de un equipo es siempre 90. Una
expulsión recorta los minutos al momento de la roja. El tramo en el campo
—`min_entra` y `min_sale`— se deriva de los minutos en lugar de leerse de la
sustitución, para que las tres columnas no puedan contradecirse.

**Marcador en la tabla de goles.** La web ofusca el marcador acumulado con CSS
generado y JavaScript, de modo que el texto plano devuelve cifras falsas, y a
veces deja un número sin poner y escribe `- 3`. El parser ignora ese dato: el
resultado real se toma del listado de partidos, y de la tabla de goles solo se
leen minuto, goleador y tipo.

**Tipos de gol, y por qué el parseo no depende de un icono.** El acta pone un
icono por tipo de gol y sus clases CSS no comparten prefijo: `lgol` el normal,
`lgolpp` el gol en propia puerta, `lpenalti` el de penalti. Buscar por la clase
`lgol` estaba condenado, y el fallo no se quedaba en el gol: como el mismo
icono servía para localizar la tabla, un tipo de gol no contemplado se llevaba
por delante los goles del partido completo. Medido: 33 partidos con el marcador
mal y 9 goles sin registrar. Ahora el tipo se lee del atributo `title`, que sí
es homogéneo, la tabla se reconoce por su forma y no por el icono, y un gol sin
tipo se registra igual con el tipo vacío. El fallo se detectó comparando con un
partido visto en directo, que es el recordatorio de que ninguna validación
automática sustituye a mirar un caso conocido.

**Una jornada sin partidos no significa nada por sí sola.** Puede ser que la
federación no sirva los datos, que la jornada esté publicada y todavía sin
actas, o que la página no traiga nada reconocible. Los tres casos se veían
igual —«0 partidos»— y durante semanas el grupo 1 tuvo tres jornadas en blanco
sin que se supiera de quién era el problema. Ahora cada uno lanza su propia
excepción y el proceso de arriba los trata distinto: el servicio caído se
registra y se sigue, la jornada pendiente no es un error, y la página ilegible
se señala para mirarla a mano.

**Tarjetas a técnicos.** Las actas sancionan también a entrenadores y personal
del cuerpo técnico. Esas tarjetas no cruzan contra ninguna alineación y quedan
registradas como no asignadas en lugar de descartarse en silencio.

**Identificadores.** Cada jugador se identifica por su código federativo,
estable entre partidos y temporadas. Los equipos, por el código que aparece en
la URL de su escudo, más fiable que el nombre porque la grafía varía entre
temporadas. Los dos espacios de códigos de la RFEF y de la federación
valenciana son distintos: un mismo jugador tiene un número en cada uno.

**Posición, año de nacimiento y pierna hábil.** El acta no los recoge. La
posición procede de lapreferente.com, una fuente colaborativa de cobertura
irregular, cruzada por conjuntos de palabras del nombre; el año de nacimiento y
la pierna hábil son curación manual. Las columnas `posicion_origen`,
`edad_origen` y `pierna_origen` registran la procedencia de cada dato, y los
huecos se dejan vacíos en lugar de rellenarse con valores inventados.

**Minutos según el marcador.** La vista reparte los minutos de cada jugador
entre ir por delante, empatado y por detrás, pero el reparto en bruto confunde
dos cosas: cuánto rota un jugador y cómo le han ido los partidos a su equipo.
Un suplente que solo entra en los partidos ya ganados acumula minutos «por
delante» que no dicen nada de él. La comparación que tiene sentido es contra
su propia base: el reparto del jugador frente al reparto de los partidos que
él jugó.

**Zona de la acción.** El tagging permite marcar la casilla del campo, pero no
obliga. En el partido analizado, cerca de una cuarta parte de las acciones no
la llevan. El cuadro de mando muestra ese porcentaje a propósito, porque es
una medida de la calidad del etiquetado y no un dato que convenga esconder.

**Colores de club.** Se extraen del escudo por superficie, pero el color con el
que se identifica a un club es el de su camiseta: el escudo del Villarreal es
mayoritariamente azul marino. Por eso `color_origen` distingue lo detectado de
lo corregido a mano, y la corrección manda.

**Muestra del vídeo.** Un partido etiquetado no sostiene una conclusión sobre
un jugador. Las cifras del cuadro de mando llevan siempre a la vista el
alcance real, y cada acción enlaza al segundo exacto de la grabación para que
cualquiera pueda comprobarla.

**No hay goles esperados.** El xG necesita las coordenadas de cada disparo y un
modelo entrenado sobre decenas de miles de ellos. Aquí no hay coordenadas de
disparo, y un modelo tomado prestado de otra competición —y de otra categoría—
produciría un número con aspecto de medición que en realidad no mide nada de
esta liga. Es el único hueco del proyecto que se deja abierto a propósito, y
queda abierto antes que rellenarse con una cifra que no se puede defender.

**Lo medido y lo valorado.** Las notas de `valoraciones` y
`valoraciones_competencia` y los textos de `informes` y `puntos` son juicio del
analista, no medición. Viven en sus propias tablas y tanto en el cuadro de
mando como en el PDF aparecen etiquetados como tales, al lado de las métricas
pero nunca mezclados con ellas. Cuando alguien pregunte de dónde sale un 4 en
velocidad, la respuesta es que lo puso una persona viendo el partido.

**Validación.** Los agregados se han contrastado contra las fichas oficiales de
estadísticas de jugador de la RFEF, que publican convocatorias, titularidades y
goles por temporada. Los números coinciden. La capa de vídeo se valida contra
el acta, como se explica más arriba.

## Próximos pasos

- **Drill-through entre páginas**: de la plantilla de un equipo a la ficha del
  jugador, y de ahí al detalle de un partido etiquetado.
- **Posición secundaria en el informe.** Un lateral que juega de interior se
  evalúa con dos juegos de competencias, y hoy solo cabe uno.
- **Fase de juego derivada.** El catálogo de botones ya tiene la columna, pero
  la fase —ataque, defensa, y las dos transiciones— se puede deducir de la
  secuencia de posesión en lugar de etiquetarla a mano.
- **Dorsal por partido.** El dorsal vive hoy en `jugadores` y es el último
  visto, no el de cada acta. Moverlo a `participaciones` haría el cruce del
  tagging inmune a un cambio de número a mitad de temporada.
- **Tabla de procedencias.** Hay ya cinco columnas `_origen` —posición, edad,
  pierna, ficha y color— y el patrón pide una tabla propia.
- **Más partidos etiquetados**, para que las métricas de vídeo pasen de ser
  una demostración de método a una muestra con la que comparar jugadores.
- **Convocatorias de selecciones** autonómicas, territoriales y nacionales.
  Añadiría una señal de valoración externa al dataset y permitiría preguntar
  si el rendimiento medible predice la convocatoria.
- **Temporadas anteriores.** El selector de la RFEF llega hasta 1990 y el
  código de jugador es estable, así que cargar 2025-26 permitiría seguir
  trayectorias y detectar cambios de club.
- **Motivos de amonestación.** El acta recoge el motivo literal de cada tarjeta
  según el reglamento. Clasificarlos daría una dimensión disciplinaria que no
  publica ninguna otra fuente.
- **Liga Nacional y categorías autonómicas.** La federación valenciana sirve
  los datos por una API JSON más rica que las actas de la RFEF, pero cerró el
  acceso con un token de página. Esa protección es deliberada y no se sortea:
  la vía correcta es pedir acceso, no rodearlo.

## Acceso al código

El código fuente está en un repositorio privado. El motivo es el del aviso de
más abajo: el sistema opera sobre una base de datos con información deportiva
de jugadores menores de edad, y aunque el repositorio nunca ha contenido esos
datos, mantenerlo cerrado reduce la superficie de un descuido.

Si quieres revisarlo —por un proceso de selección, por interés técnico o para
hablar del método— pídeme acceso de lectura y te lo doy. Lo que hay dentro es
lo que describe la sección «Estructura del proyecto»: once módulos, el esquema
completo, 116 tests, la plantilla del informe y las herramientas de
diagnóstico.

## Aviso sobre datos personales

La competición es de categoría juvenil y los jugadores son menores de edad. La
base de datos guarda únicamente datos deportivos: ni fotografías, ni datos de
contacto, ni nada que no sea rendimiento sobre el campo.

Las actas traen incrustado el retrato de la licencia de cada jugador, y por
tanto esas imágenes están dentro del HTML crudo que se guarda en `bronze/`.
Nunca se extraen, nunca llegan a la base de datos y esa carpeta no se publica.
La única fotografía que aparece en un informe es la que se le pasa a mano al
generarlo, y queda incrustada en ese PDF concreto sin copiarse a ningún sitio.

Nada que contenga nombres sale publicado: ni la base de datos, ni los ficheros
de curación manual, ni los CSV de etiquetado, ni los PDF generados, ni el
fichero de Power BI. Y las capturas de pantalla de este repositorio no muestran
nombres de jugadores.

Los clips enlazan a vídeos que los propios clubes han publicado en abierto, y
el enlace apunta a la fuente original en lugar de redistribuir el material.

## Autoría y licencia

Proyecto de Roberto Peña Alcaide.

El texto y las imágenes de este repositorio se publican para su consulta y
evaluación, con todos los derechos reservados; ver `LICENSE`. El código fuente
no está incluido y no se publica bajo ninguna licencia.

Los commits de este repositorio y del privado van firmados.
