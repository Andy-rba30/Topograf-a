# Parches

Los mismos commits que están publicados en las ramas `feature/coordenadas-compartidas` (revit-mcp) y
`feature/extension-superficie` (civil3d-mcp), por si esas ramas se rehacen. Se aplican sobre la rama por defecto de
cada repo: `migracion-mcp2` en revit-mcp y `main` en civil3d-mcp:

```powershell
cd <repo revit-mcp>;   git am <ruta>\parches\revit-mcp\*.patch
cd <repo civil3d-mcp>; git am <ruta>\parches\civil3d-mcp\*.patch
```
