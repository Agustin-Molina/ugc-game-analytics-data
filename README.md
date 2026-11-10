# UGC Game Analytics — datos

Datos crudos recolectados automáticamente por
[ugc-game-analytics](https://github.com/Agustin-Molina/ugc-game-analytics) con GitHub Actions.
No editar a mano: todo es **append-only** (cada corrida solo agrega archivos nuevos).

```
roblox/
  snapshots/date=YYYY-MM-DD/*.parquet         # horario: playing, visits, favoritos, votos
  chart_positions/date=YYYY-MM-DD/*.parquet   # posición en cada chart de explore
  events/date=YYYY-MM-DD/*.parquet            # cambios de nombre / descripción / update
  tier_history/date=YYYY-MM-DD/*.parquet      # cambios de tier (hot / warm / cold) y motivo
  games/date=YYYY-MM-DD/*.parquet             # dimensión como log de cambios
fortnite/
  snapshots/date=YYYY-MM-DD/*.parquet         # métricas day (con retención) y hour
  fetch_log/date=YYYY-MM-DD/*.parquet         # cada consulta, con o sin datos
  events/date=YYYY-MM-DD/*.parquet            # islas nuevas/eliminadas, cambios de título/tags
  games/date=YYYY-MM-DD/*.parquet             # dimensión como log de cambios
```

## Convenciones

- Parquet + zstd, particionado estilo Hive por fecha UTC de la corrida (`date=`).
- Todos los timestamps en UTC. `snapshot_at` = momento real de cada respuesta de la API.
- **Retención (`retention_d1`, `retention_d7`, Fortnite) es una fracción entre 0 y 1**, no un
  porcentaje: 0.12 = 12 % de los jugadores volvió al día siguiente. Null si la API no la
  informa (isla con < 5 jugadores únicos o día todavía no cumplido).
- Las métricas de Fortnite valen `null` cuando la isla tuvo < 5 jugadores únicos en el bucket
  (la API no las publica); esos buckets se descartan, pero cada consulta queda en `fetch_log`.
- Roblox: no todos los juegos se observan cada hora. Según su tier (columna `tier` de
  `games`, historial en `tier_history`) se consultan cada hora (hot), una vez por día (warm)
  o una vez por semana (cold); la cadencia real de cada juego sale de sus `snapshot_at`.
- `is_partial = true`: el bucket todavía no había cerrado al momento de la consulta.
- Fortnite: por el solapamiento de ventanas (cada isla se consulta cada 5 días con ventana de
  7) puede haber varias filas por (isla, interval, bucket_start); usar la de `snapshot_at`
  más reciente.

## Dimensiones (`games`): log append-only

Cada corrida agrega solo las filas nuevas o con algún atributo cambiado, con `valid_from`.
**Estado actual = última fila por id**:

```sql
-- Roblox (id = universe_id); para Fortnite usar fortnite/games y island_code
select *
from read_parquet('roblox/games/date=*/*.parquet', union_by_name = true)
qualify row_number() over (partition by universe_id order by valid_from desc) = 1;
```

El historial completo (una fila por cambio) sirve para ver cómo evolucionó cada juego.
Campos como "última vez visto" o "última actividad" no se guardan en la dimensión: se
derivan de `snapshots` / `fetch_log`.

## Uso rápido

```python
import duckdb
duckdb.sql("""
    select * from read_parquet('roblox/snapshots/**/*.parquet', hive_partitioning = true)
    limit 10
""")
```

Con el repo de código, `pipeline.db.connect('<este repo>')` crea vistas listas
(`roblox_games`, `fortnite_snapshots_latest`, …). Esquema completo de cada tabla: ver el
README del repo de código.

Fuentes: APIs públicas de Roblox y de Epic Games (Fortnite Data API); sujetas a sus términos.
