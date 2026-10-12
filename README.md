# Topografía: superficies de Civil 3D 2027 en Revit 2027 sin el aviso "muy lejos del origen"

Este repositorio documenta la solución al aviso que Revit muestra cada vez que se vincula una superficie de Civil 3D,
aunque los dos archivos estén georreferenciados, y recoge los cambios hechos en los dos conectores MCP del autor:

| Repositorio | Rama | Qué añade |
|---|---|---|
| [Andy-rba30/revit-mcp](https://github.com/Andy-rba30/revit-mcp) | **fusionado en `main`** (PR #24, versión 0.6.5; rama `feature/coordenadas-compartidas-main`) | `shared_coordinates`, `units` y `at_point_mm` en `set_project_location`; `placement` y `extents` en `link_file`; `shared_coordinates_set` y `site` en la ubicación del proyecto; sección 2f de `INSTRUCCIONES_AGENTE.md` |
| [Andy-rba30/civil3d-mcp](https://github.com/Andy-rba30/civil3d-mcp) | **fusionado en `main`** (PR #4, versión 1.4.0) | `extension_superficie` y la sección 7 de `INSTRUCCIONES_AGENTE.md` con el flujo completo |

Nota: el PR #23 de revit-mcp (rama `feature/coordenadas-compartidas`) se cerró sin fusionar porque partía de `migracion-mcp2`, 70 commits por detrás de `main`, con herramientas distintas. Sobre `main` las herramientas ya existían con otros nombres (`set_project_location`, `get_revit_model_info(include=["location"])`, `import_from_civil`), así que allí la solución se integró en ellas: los nombres `get_project_coordinates` y `set_project_coordinates` de este documento corresponden en `main` a `get_revit_model_info(include=["location"])` y a `set_project_location(shared_coordinates=...)`. Los parches de `parches/` son los del PR #23 y el PR #4; el commit fusionado en revit-mcp es el de la rama `feature/coordenadas-compartidas-main`.

## El problema

Revit tiene un origen interno que no se puede mover. Autodesk pide que toda la geometría quede a menos de 16 km
(10 millas) de ese origen; a partir de 33 km (20 millas) Revit avisa y, al vincular un CAD, cambia la posición a
"centro con centro" por su cuenta, con lo que la superficie deja de estar georreferenciada.

Una superficie de Civil 3D en UTM (por ejemplo E 500 000, N 8 500 000) está a miles de kilómetros del 0,0. Que Revit
esté "georreferenciado" significa otra cosa: que sus coordenadas compartidas (punto de replanteo, punto base) tienen un
desplazamiento respecto al origen interno. Si ese desplazamiento sigue en 0,0,0, Revit lee el WCS del DWG como si el
origen interno fuera UTM 0,0.

En `revit-mcp`, además, `link_file` fijaba siempre `ImportPlacement.Origin` (origen con origen), sin forma de cambiarlo:
el aviso estaba garantizado con cualquier DWG georreferenciado.

## La solución: vincular por coordenadas compartidas

Es el flujo que Revit mismo usa (Gestionar → Coordenadas → Especificar coordenadas en un punto, y después Vincular CAD
con posición "Automático - Por coordenadas compartidas"), llevado a herramientas MCP para que el agente lo haga solo:

| Herramienta | MCP | Qué hace |
|---|---|---|
| `extension_superficie(superficie)` | Civil 3D | Mínimos y máximos de X, Y y cota, centro, `origen_sugerido` (centro redondeado a 100 m; cota a 10 m), número de puntos, sistema de coordenadas y unidades del dibujo, y avisos si la superficie mide más de 16 km o de 33 km |
| `get_project_coordinates(units)` | Revit | Posición del origen interno en coordenadas compartidas (`internal_origin`), `shared_coordinates_set`, punto base, punto de replanteo, sitio (latitud, longitud, sistema GIS) |
| `set_project_coordinates(east, north, elevation, angle_degrees, units, point)` | Revit | `ProjectLocation.SetProjectPosition`: da a un punto interno (por defecto el origen) sus coordenadas reales. Una vez por modelo |
| `link_file(file_path, placement="shared")` | Revit | Vincula el DWG "por coordenadas compartidas". Con `shared` y sin coordenadas fijadas responde error y explica qué falta. La respuesta dice a qué distancia del origen quedó el vínculo (`extents`) y avisa a partir de 16 km |

### Flujo del agente

El agente lo recibe como instrucciones del servidor MCP de Civil 3D (`PuenteMcp/INSTRUCCIONES_AGENTE.md`, sección 7).

1. Civil 3D: `listar_superficies` → `extension_superficie("EG")`. Anotar `origen_sugerido`, `sistema_coordenadas.codigo`
   y `avisos`. Un aviso de más de 33 km no lo arregla ninguna posición: hay que recortar.
2. Revit: `get_project_coordinates` → si `shared_coordinates_set` es `false`, el modelo nunca se georreferenció.
3. Revit: `set_project_coordinates(east=E0, north=N0, elevation=Z0, units="m")` con `origen_sugerido`. `angle_degrees=0`
   si el DWG está alineado al norte de cuadrícula (lo habitual).
4. Civil 3D: `guardar_dibujo` si hay cambios sin guardar; Revit lee el DWG de disco.
5. Revit: `link_file(file_path="C:\...\topografia.dwg", placement="shared")`. Comprobar `extents.far_from_origin=false` y
   que `center_shared_m` coincide con `centro` del paso 1.
6. Revit: Toposolid con "Crear a partir de importación" sobre el vínculo (manual, o con `execute_revit_code`).

Si el DWG ya estaba vinculado "origen con origen", hay que volver a vincularlo tras el paso 3: el vínculo viejo no se
recoloca solo.

### Ejemplo de respuestas

```json
// extension_superficie {"superficie": "EG"}
{"superficie": "EG", "tipo_superficie": "TIN", "n_puntos": 184230,
 "min_x": 498120.4, "max_x": 501880.9, "min_y": 8498910.2, "max_y": 8502105.7, "min_z": 2381.1, "max_z": 2466.8,
 "ancho_x": 3760.5, "alto_y": 3195.5, "diagonal": 4934.6,
 "centro": {"x": 500000.65, "y": 8500507.95, "z": 2423.95},
 "origen_sugerido": {"este": 500000, "norte": 8500500, "cota": 2420},
 "sistema_coordenadas": {"codigo": "UTM84-18S", "unidades": "Meters", "nota": null},
 "avisos": []}
```

```json
// set_project_coordinates {"east": 500000, "north": 8500500, "elevation": 2420, "units": "m"}
"Shared coordinates set on location 'Site': internal point (0.0, 0.0, 0.0) mm is now E=500000 N=8500500
 elevation=2420 m with true north at 0 degrees. Internal origin is now E=500000.0 N=8500500.0 elevation=2420.0 m.
 Link georeferenced CAD files with placement='shared'."
```

```json
// link_file {"file_path": "C:\\Proyectos\\topografia.dwg", "placement": "shared"}
"Linked file 'topografia.dwg' (placement: shared). Farthest corner 2512.3 m from the internal origin;
 centre at shared E=500000.65 N=8500507.95 elevation=2423.95 m"
```

## Qué está probado y qué no

Probado en esta sesión, sin Civil 3D ni Revit:

- `revit-mcp`: 33 pruebas nuevas de `coordinates_core` (conversiones, nombres de posición, distancia al origen,
  bandera de coordenadas compartidas) con pytest; sintaxis de los módulos IronPython; el puente arranca y registra las
  50 herramientas con los parámetros esperados.
- `civil3d-mcp`: las 32 pruebas del puente con pytest (dos ajustadas a la versión 1.4.0). El C# se escribió contra los
  sustitutos de `herramientas-dev/CompilarSinCivil` (con los tipos nuevos añadidos), pero **no se compiló aquí**: no hay
  `dotnet` en el contenedor. La integración continua del repo lo compila en el PR.

No probado: la ejecución real en Civil 3D 2027 y Revit 2027. El prompt para el agente local está en
`civil3d-mcp/herramientas-dev/VALIDACION_14.md` (pasos 0 a 10, los decisivos son el 2 y el 9) y la lista de
comprobaciones del lado Revit en [`docs/validacion.md`](docs/validacion.md).

Miembros de la API que esa validación decide (anotados como `por verificar` en
`herramientas-dev/miembros_por_verificar_civil3d.md`):

- Civil 3D: `Surface.GetGeneralProperties()` y `GeneralSurfaceProperties.MinimumCoordinateX`/`...Y`/`MinimumElevation`/
  `MaximumElevation`/`NumberOfPoints`; `CivilDocument.Settings.DrawingSettings.UnitZoneSettings.CoordinateSystemCode` y
  `.DrawingUnits`.
- Revit: `ProjectLocation.GetProjectPosition`/`SetProjectPosition`, `Creation.Application.NewProjectPosition`,
  `BasePoint.GetProjectBasePoint`/`GetSurveyPoint`, `SiteLocation.GeoCoordinateSystemId`, `ImportPlacement.Shared` en
  `DWGImportOptions` y `RevitLinkInstance.Create(doc, id, placement)`. Todos están en la documentación de la API 2027;
  el que puede sorprender es `SetProjectPosition` sobre la ubicación "Internal" de solo lectura, que el código ya
  contempla (activa otra ubicación o duplica una).

## Decisiones y límites

- **`link_file` sigue con `placement="origin"` por defecto**, para no cambiar el comportamiento de quien ya lo usa. Con
  un CAD georreferenciado la respuesta avisa de la distancia y dice qué hacer; el agente, además, lee el flujo en la
  descripción de la herramienta y en las instrucciones del MCP de Civil 3D.
- **Metros por defecto** en las dos herramientas de coordenadas, aunque el resto del MCP de Revit trabaja en mm: las
  coordenadas topográficas llegan de Civil 3D en unidades de dibujo. Admite `units="mm"` y `"ft"`.
- **Extensión del DWG.** El límite de 33 km también aplica a la extensión del archivo. Un corredor de 40 km da el aviso
  aunque se centre bien; `extension_superficie` lo avisa y entonces hay que recortar la superficie o el DWG, o pasar al
  camino B (puntos a Toposolid por API, no implementado).
- **UTM es cuadrícula, no terreno.** Las coordenadas compartidas de Revit son un desplazamiento plano sin factor de
  escala. Para sitios de pocos kilómetros son centímetros; para obras lineales largas conviene trabajar en coordenadas
  locales de Civil 3D.
- **`angle_degrees`** es el ángulo del norte de proyecto al norte verdadero. Si el DWG está en norte de cuadrícula y el
  modelo de Revit también, es 0. Si el edificio se modeló girado respecto al norte, se indica aquí.
- **Versión** de `civil3d-mcp` subida a 1.4.0 (plugin, bundle y puente) siguiendo la práctica del repo; `probar_servidor.py`
  acepta ahora 1.3.x y 1.4.x en `/ping`.

## Instalar las dos ramas

Civil 3D (con Civil 3D cerrado):

```powershell
git fetch origin feature/extension-superficie
git checkout feature/extension-superficie
powershell -ExecutionPolicy Bypass -File .\ArbaMcp\instalar.ps1
# reiniciar el puente (PuenteMcp\main.py) y el cliente MCP
```

Revit (la extensión de pyRevit es el propio repo):

```powershell
git fetch origin feature/coordenadas-compartidas
git checkout feature/coordenadas-compartidas
# pyRevit → Reload; reiniciar el puente (uv run main.py) y el cliente MCP
```
