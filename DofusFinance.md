# Plan de desarrollo: Optimizador de recursos y recetas de Dofus

Programa automatizado que usa los datos de DofusDB para indicar **qué recetas conviene fabricar** según tu inventario y los precios que ingreses, y **cuánto pagar como máximo** por lo que te falte.

## 1. Alcance

**Incluye**

- Catálogo de recursos de los oficios de recolección y sus recetas (desde DofusDB).
- Ingreso de unidades y precio por unidad de cada recurso, y del precio final de cada producto.
- Recomendación de recetas más rentables según inventario, con detección de faltantes.
- Precio máximo y unidades máximas de compra para que una receta siga siendo rentable.
- Modo "solo precios": recomendaciones sin inventario, usando únicamente precios de recursos y de producto final.
- Automatización de la sincronización de datos y del recálculo.
- Ejecución 100 % local, para uso personal (sin nube ni acceso público).

**Fuera de alcance:** optimizador de equipamiento, clases, panoplias y builds.

## 2. Qué se sabe de la API

- Base: `https://api.dofusdb.fr` (beta: `api.beta.dofusdb.fr`). No es una API oficial de Ankama y puede cambiar sin aviso.
- Estilo de consulta observado: `/items?typeId[$in][]=1&$sort=-id&$skip=0`, con respuesta `{ total, limit, skip, data }`.
- Existen endpoints de `items` y `recipes` y de versión del juego (según clientes comunitarios).
- **DofusDB no entrega precios de mercado**: los precios siempre los ingresa el usuario.
- Por verificar en la Fase 0: esquema exacto de `recipes`, relación con oficios, límites de uso y términos.

## 3. Arquitectura

**El programa es de uso personal y se ejecuta solo en tu computador.** No hay servidor en la nube, cuentas de usuario, autenticación ni despliegue público. La única conexión a internet es la descarga de datos desde DofusDB.

| Capa | Propuesta |
| --- | --- |
| Lenguaje y lógica | Python, todo en un único proceso local (sin backend separado) |
| Base de datos | SQLite: un solo archivo en tu disco |
| Optimización | OR-Tools (CP-SAT) o PuLP |
| Interfaz | Streamlit ejecutado en `localhost` (más rápido de construir) o una app de escritorio con PySide/Tkinter |
| Tareas programadas | APScheduler dentro del programa, o el programador de tareas del sistema operativo |
| Empaquetado | Script de arranque o ejecutable con PyInstaller para abrirlo con doble clic |

Principios:

1. Los cálculos usan una **copia local** de los datos; nunca llaman a la API en vivo.
2. Los datos del usuario (inventario, precios) viven en tablas separadas y **no se borran** al re-sincronizar.
3. El JSON crudo de la API se guarda para poder reprocesar sin volver a descargar.
4. Si usas Streamlit, se enlaza solo a `127.0.0.1` para que nadie más en tu red pueda abrirlo.
5. **Copias de seguridad:** al ser un archivo local, el programa hace una copia automática de la base de datos (por ejemplo, diaria y antes de cada re-sincronización) y permite restaurarla.

### Modelo de datos

- `item(id, nombre, tipo, nivel)`
- `job(id, nombre)`
- `recipe(id, item_resultado_id, job_id, nivel_requerido)`
- `recipe_ingredient(recipe_id, item_id, cantidad)`
- `user_stock(item_id, unidades)`
- `user_price(item_id, precio_unitario, actualizado_en)` (aplica a recursos y a productos finales)
- `settings(tasa_venta, margen_minimo, presupuesto, usar_costo_oportunidad)`

## 4. Modelo de rentabilidad

Sea `t` la tasa de venta (configurable, 2 % por defecto, a confirmar en el juego).

```
Ingreso neto por craft = PrecioProducto × (1 − t)
Costo por craft        = Σ (cantidad_i × precio_i)
Ganancia por craft     = Ingreso neto − Costo
Margen %               = Ganancia / Costo
```

**Costo de oportunidad (opción activable):** los recursos que ya tienes se valoran a su precio de mercado, no a cero. Si se desactiva, se tratan como costo hundido. El sistema debe mostrar ambos resultados para que veas la diferencia.

## 5. Modos de recomendación

### Modo A: Con inventario

Decide cuántas veces fabricar cada receta cuando **varias recetas compiten por los mismos recursos**. Ordenar por margen y fabricar en ese orden no es óptimo; se resuelve como programa entero:

- `x_r` = número de crafts de la receta `r` (entero ≥ 0)
- `b_i` = unidades a comprar del recurso `i` (≥ 0)
- `u_i` = unidades de tu stock que se consumen (≤ stock_i)

```
Maximizar   Σ_r x_r · PrecioProducto_r · (1 − t)
          − Σ_i precio_i · b_i
          − Σ_i valor_oportunidad_i · u_i

Sujeto a    Σ_r cantidad_ir · x_r = u_i + b_i     para cada recurso i
            u_i ≤ stock_i
            Σ_i precio_i · b_i ≤ presupuesto       (opcional)
            b_i ≤ tope_compra_i                    (opcional)
            x_r ≤ tope_venta_r                     (opcional, saturación del mercado)
```

Salida: plan de fabricación (receta → cantidad de crafts), lista de compras y ganancia total esperada.

### Modo B: Solo precios (sin inventario)

Se fija `stock = 0`. Cada receta se evalúa únicamente con el precio por unidad de sus recursos y el precio final del producto.

- Sin presupuesto: ranking por ganancia por craft y por margen %.
- Con presupuesto: el mismo modelo del Modo A, que elige la mejor combinación de recetas para el capital disponible.

Este modo responde a "¿qué vale la pena fabricar hoy?" aunque no tengas nada.

## 6. Faltantes: precio máximo y unidades máximas

Para cada receta que no puedes completar con tu stock:

```
Faltante_i     = N × cantidad_i − stock_i
Precio_max_i   = (PrecioProducto × (1 − t) − Σ costo de los demás ingredientes − margen_mínimo) / cantidad_i
```

`Precio_max_i` es el punto de equilibrio: comprar por encima de ese valor deja la receta sin ganancia. **Unidades máximas a comprar** se limita por:

1. Faltante real para completar los crafts planeados.
2. Presupuesto disponible (si se define).
3. Escalones de precio del mercado (lotes x1 / x10 / x100 / x1000), si los ingresas: se compra hasta donde el precio del escalón siga por debajo de `Precio_max_i`.

Con un solo precio fijo, comprar es rentable o no lo es; los puntos 2 y 3 son los que dan un límite real.

## 7. Automatización

- **Sincronización programada:** al abrir el programa (y una vez al día si queda abierto) consulta la versión del juego; si cambió, re-sincroniza items y recetas. Si no hay internet, sigue funcionando con los datos ya guardados.
- **Recálculo automático** cada vez que cambias un stock o un precio.
- **Alertas locales** dentro de la interfaz o como notificación de escritorio: una receta pasa a ser rentable o deja de serlo, o un precio lleva más de X días sin actualizarse.
- **Copias de seguridad automáticas** de la base de datos.
- **Importar/exportar CSV** de inventario y precios para cargar datos en bloque.
- **Historial de precios** (opcional): guardar cada cambio para ver tendencias y márgenes en el tiempo.

## 8. Fases

| Fase | Contenido | Duración |
| --- | --- | --- |
| 0. Exploración | Esquemas de `items`, `recipes`, oficios; límites y términos de uso | 2–3 días |
| 1. Datos | Sincronización completa e incremental, modelo local | 1 semana |
| 2. Catálogo y entrada de datos | Lista de recursos por oficio, edición de stock y precios, CSV | 1 semana |
| 3. Motor básico | Fórmulas de rentabilidad, ranking, Modo B sin presupuesto | 1 semana |
| 4. Optimizador | Modelo entero del Modo A, presupuesto, topes | 1–2 semanas |
| 5. Faltantes | Precio máximo, unidades máximas, escalones de precio | 1 semana |
| 6. Automatización | Tareas programadas, alertas locales, copias de seguridad, historial | 1 semana |
| 7. Interfaz, empaquetado y pruebas | Pantallas, arranque con doble clic, validación con casos reales | 1 semana |

Al ser local, no hay fase de despliegue, servidores ni seguridad web. Total estimado: **6–8 semanas** a tiempo parcial. Un MVP útil (Modo B con ranking) queda listo en la semana 3.

## 9. Criterios de aceptación

- Todas las recetas de los oficios de recolección se cargan y coinciden con DofusDB.
- Con stock 0 y solo precios, el sistema entrega un ranking correcto (verificado a mano en 10 recetas).
- Con dos recetas que comparten un recurso escaso, el plan del optimizador supera al de "ordenar por margen".
- Para cada faltante se muestra precio máximo y unidades máximas coherentes con la fórmula.
- Tras una re-sincronización, tu stock y tus precios siguen intactos.

## 10. Riesgos

| Riesgo | Mitigación |
| --- | --- |
| API no oficial que cambia o cae | Base local, JSON crudo guardado, capa adaptadora aislada |
| Precios desactualizados | Marca de tiempo por precio y alertas de antigüedad |
| Tasa de venta u otras reglas distintas a las supuestas | Parámetros configurables y validados en el juego |
| Recetas con ingredientes que a su vez se fabrican | Fase posterior: comparar comprar vs. fabricar el intermedio |
| Límites de peticiones | Pausas, paginación y sincronización incremental (el uso personal genera muy poco tráfico) |
| Pérdida de datos al estar todo en un archivo local | Copias automáticas de la base de datos y exportación a CSV |

## 11. Supuestos y decisiones pendientes

1. **Límite de compra:** ¿por presupuesto, por escalones de precio, o ambos?
2. **Costo de oportunidad** activado o desactivado por defecto.
3. **Interfaz local:** Streamlit en `localhost` (rápido de construir, se abre en el navegador) o una app de escritorio (más integrada, más trabajo).
4. **Sistema operativo** donde lo usarás (Windows, macOS o Linux), para definir el empaquetado y el arranque automático.
5. **Recetas anidadas:** ¿entran en la primera versión o después?