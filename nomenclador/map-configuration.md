# Configuración del Mapa - Nomenclador

## Cambios Realizados (10 de Diciembre, 2025)

### 1. Mapa Base - IGN Argentina
Se actualizó el mapa base para utilizar los servicios del Instituto Geográfico Nacional (IGN) de Argentina.

**Archivo modificado**: `src/services/constants.ts`
- **Antigua URL**: OpenStreetMap (`https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png`)
- **Nueva URL**: IGN Argentina TMS (`https://wms.ign.gob.ar/geoserver/gwc/service/tms/1.0.0/mapabase_gris@EPSG%3A3857@png/{z}/{x}/{-y}.png`)
- **Servicio**: TMS (Tile Map Service) del IGN
- **Capa**: mapabase_gris (mapa base en escala de grises de Argentina)
- **Proyección**: EPSG:3857 (Web Mercator)
- **Opción TMS**: true (necesaria para invertir correctamente las coordenadas Y)
- **Atribución actualizada**: Instituto Geográfico Nacional (pero removida de la visualización)

### 2. Deshabilitación del Zoom
Se deshabilitaron todas las interacciones de zoom en el mapa para evitar que los usuarios cambien el nivel de zoom.

**Configuraciones aplicadas**:
```typescript
{
  zoomControl: false,          // Oculta los botones +/- de zoom
  scrollWheelZoom: false,      // Desactiva zoom con la rueda del ratón
  doubleClickZoom: false,      // Desactiva zoom con doble clic
  touchZoom: false,            // Desactiva zoom con gestos táctiles
  dragging: true,              // Mantiene la capacidad de arrastrar el mapa
}
```

**Archivo modificado**: `src/components/map/LeafletMap.tsx`
- Se actualizó la inicialización del mapa para aplicar las opciones de zoom desde MAP_CONFIG
- El mapa ahora respeta las configuraciones de zoom definidas en constants.ts

### 3. Remoción de Atribuciones
Se removió la caja de atribución que aparecía en la esquina inferior derecha del mapa.

**Configuración aplicada**:
```typescript
{
  attributionControl: false,   // Oculta el control de atribución (Leaflet/OpenStreetMap links)
}
```

**Archivo modificado**: `src/components/map/LeafletMap.tsx`
- Se agregó `attributionControl: false` en las opciones de inicialización del mapa
- Se removió la propiedad `attribution` del tileLayer (ya no es necesaria)

### 4. Comportamiento del Mapa
- **Pan/Arrastrar**: ✅ HABILITADO - Los usuarios pueden mover el mapa
- **Zoom con rueda**: ❌ DESHABILITADO
- **Zoom con doble clic**: ❌ DESHABILITADO
- **Zoom táctil**: ❌ DESHABILITADO
- **Controles de zoom (+/-)**: ❌ DESHABILITADOS
- **Atribuciones (Leaflet/OSM)**: ❌ REMOVIDAS
- **Zoom automático**: ✅ FUNCIONAL - Al seleccionar una unidad geoestadística, el mapa ajusta automáticamente los límites con `fitBounds`

### Notas Técnicas
- El mapa del IGN utiliza el sistema de coordenadas EPSG:3857 (Web Mercator)
- Se utiliza el servicio TMS del IGN con la capa `mapabase_gris` (mapa en escala de grises)
- La URL utiliza `{-y}` para invertir las coordenadas Y según el estándar TMS
- La opción `tms: true` debe estar configurada en Leaflet para que interprete correctamente las coordenadas
- El zoom programático (fitBounds) sigue funcionando cuando se carga una geometría
- Los usuarios solo pueden navegar mediante arrastre (pan)
- No se muestra información de atribución en la interfaz (limpio y minimalista)

### Troubleshooting
**Problema**: Error HTTP 400 - Invalid character found in the request target
**Solución**: Se cambió de servicio WMTS a TMS porque el servicio WMTS tenía problemas con la codificación de caracteres en la URL. El servicio TMS es más simple y compatible con Leaflet.

**Problema**: Mapa se ve gris o no carga tiles
**Causas posibles**:
1. Problema de conectividad con los servicios del IGN
2. CORS no configurado correctamente
3. Servicio temporalmente no disponible
4. URL del servicio incorrecta

**Solución aplicada**: Uso del servicio TMS del IGN con la capa `mapabase_gris` que es estable y funcional.

### URL del Servicio IGN (TMS)
```
https://wms.ign.gob.ar/geoserver/gwc/service/tms/1.0.0/mapabase_gris@EPSG%3A3857@png/{z}/{x}/{-y}.png
```

### Capas Disponibles del IGN
- `mapabase_gris`: Mapa base en escala de grises
- `cabase_raster`: Mapa base a color (alternativa si se prefiere color)
- `mapabase_topo`: Mapa topográfico

Para cambiar de capa, simplemente reemplazar `mapabase_gris` en la URL con el nombre de la capa deseada.
