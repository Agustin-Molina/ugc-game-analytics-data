# UGC Game Analytics — datos

Datos crudos recolectados automáticamente por
[ugc-game-analytics](https://github.com/Agustin-Molina/ugc-game-analytics) con GitHub Actions.
No editar a mano: cada corrida agrega archivos nuevos (append-only), salvo las dimensiones
`games/games.parquet`, que se reescriben con el estado actual.

```
roblox/
  snapshots/date=YYYY-MM-DD/*.parquet         # horario: playing, visits, favoritos, votos
  chart_positions/date=YYYY-MM-DD/*.parquet   # posición en cada chart de explore
  events/date=YYYY-MM-DD/*.parquet            # cambios de nombre / descripción / update
  games/games.parquet                         # dimensión (estado actual)
fortnite/
  snapshots/date=YYYY-MM-DD/*.parquet         # métricas day (con retención) y hour
  fetch_log/date=YYYY-MM-DD/*.parquet         # cada consulta, con o sin datos
  events/date=YYYY-MM-DD/*.parquet            # islas nuevas/eliminadas, cambios de título/tags
  games/games.parquet                         # dimensión (estado actual)
```

- Formato: Parquet + zstd, particionado estilo Hive por fecha UTC de la corrida.
- Timestamps en UTC. `snapshot_at` = momento real de cada respuesta de la API.
- Fortnite: por el solapamiento de ventanas puede haber varias filas por
  (isla, interval, bucket_start); usar la de `snapshot_at` más reciente. `is_partial = true`
  marca buckets que todavía no habían cerrado.

Esquema completo y forma de uso: ver el README del repo de código.

```python
import duckdb
duckdb.sql("select * from read_parquet('roblox/snapshots/**/*.parquet', hive_partitioning=true) limit 10")
```

Fuentes: APIs públicas de Roblox y de Epic Games (Fortnite Data API); sujetas a sus términos.
