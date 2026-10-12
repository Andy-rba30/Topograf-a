# Validación del lado Revit (Revit 2027, rama `feature/coordenadas-compartidas` de revit-mcp)

Complementa `civil3d-mcp/herramientas-dev/VALIDACION_14.md`, que lleva el flujo completo (pasos 0 a 10). Aquí están
las comprobaciones que solo tocan Revit, para hacerlas con un **proyecto nuevo de prueba** (el paso 3 cambia las
coordenadas compartidas del modelo) y la extensión recargada en pyRevit.

Anota en cada paso la respuesta literal de la herramienta. No marques OK lo que no viste.

1. **Arranque.** `get_revit_status` responde `active`. Pide la lista de herramientas: 50, con `get_project_coordinates`
   y `set_project_coordinates`, y `link_file` con el parámetro `placement`.

2. **Lectura inicial.** `get_project_coordinates` (sin argumentos). Esperado: `Shared Coordinates Set: False`,
   `Internal Origin` con east, north y elevation en 0 y `angle_degrees` 0, `Project Base Point` y `Survey Point` con
   `internal` y `shared`, `Active Location` con el nombre de la ubicación activa y `Locations` con la lista. Si
   `Project Base Point` o `Survey Point` salen `None`, anótalo: `BasePoint.GetProjectBasePoint` no respondió.

3. **Fijar coordenadas.** `set_project_coordinates` con `east=500000`, `north=8500500`, `elevation=2420`,
   `units="m"` (o los valores reales de tu superficie). Esperado: mensaje "Shared coordinates set on location ...".
   Si dice "(activated instead of the read-only one)", la ubicación activa era la "Internal" de Revit 2018+ y el
   código activó otra: anótalo, es comportamiento previsto. Luego `get_project_coordinates`: `Shared Coordinates Set:
   True` e `Internal Origin` igual a lo enviado (tolerancia 0,001). En Revit, Gestionar → Coordenadas → Especificar
   coordenadas en un punto, pulsando sobre el origen interno (vista de plano con el origen interno visible), debe
   mostrar esos valores. Comprueba que en el historial de deshacer hay **una** entrada "Set project coordinates via
   MCP".

4. **Unidades.** `get_project_coordinates` con `units="mm"`: los mismos valores multiplicados por 1000. Con
   `units="ft"`, divididos por 0,3048. Con `units="in"`: error "units 'in' not supported. Use: m, mm, ft".

5. **Posiciones rechazadas.** `link_file` con un DWG cualquiera y `placement="manual"`: error "placement 'manual' not
   supported. Use: origin, shared, center, site". No debe quedar ningún vínculo.

6. **Vínculo sin coordenadas (en otro proyecto nuevo, sin el paso 3).** `link_file` con el DWG georreferenciado y
   `placement="shared"`: error que empieza por "placement 'shared' needs shared coordinates". Luego el mismo DWG con
   `placement="origin"`: el vínculo se crea y el mensaje trae "WARNING: geometry is N km from the internal origin,
   beyond the 33 km" con N del orden de miles. Revit **no** debe haber mostrado ningún cuadro (la extensión los
   suprime); si lo mostró, anótalo. Borra ese vínculo.

7. **Vínculo por coordenadas compartidas (decisivo).** En el proyecto del paso 3: `link_file` con el DWG y
   `placement="shared"`. Esperado: mensaje "Linked file '...' (placement: shared). Farthest corner N m from the internal
   origin; centre at shared E=... N=... elevation=..." con N menor de 16 000 y sin "WARNING". En Revit: vista 3D con la
   superficie junto al origen interno; Gestionar → Vínculos → la posición dice "Compartido"; Gestionar → Coordenadas →
   Informar coordenadas compartidas sobre un punto de la superficie da valores UTM coherentes con Civil 3D.

8. **Vínculo RVT (opcional).** `link_file` con un `.rvt` y `placement="shared"`: debe crear el vínculo sin error aunque
   el RVT no tenga coordenadas compartidas (Revit lo coloca donde pueda); con `placement="center"` y `"site"` también.

9. **Comprobación cruzada con Civil 3D.** Paso 10 de `VALIDACION_14.md`: la cota de un mismo punto coincide
   (tolerancia 0,05 m).

Informe: tabla paso / OK o FALLO / respuesta literal, versión de Revit y de pyRevit, y capturas de los cuadros de
Revit si apareció alguno en los pasos 6 o 7.
