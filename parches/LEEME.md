# Parches

Los mismos commits que están publicados en las ramas `feature/coordenadas-compartidas` (revit-mcp) y
`feature/extension-superficie` (civil3d-mcp), por si esas ramas se rehacen. Se aplican sobre `main` de cada repo:

```powershell
cd <repo revit-mcp>;   git am <ruta>\parches\revit-mcp\*.patch
cd <repo civil3d-mcp>; git am <ruta>\parches\civil3d-mcp\*.patch
```
