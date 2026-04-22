# Docker Guide

## Services and Ports

| Service | Container Name | Port |
|---------|---------------|------|
| API server | `api` | **3742** |
| PostgreSQL | `db` | **5499** |
| Redis | `cache` | 6380 |
| Mailhog | `mail` | 8025 |

## Starting the Stack

```bash
docker compose up -d
docker compose logs -f api
```

## Common Troubleshooting

**Container won't start**: `docker compose down && docker volume rm projectname_pgdata && docker compose up -d`

**Port conflict**: Check nothing is bound to 3742 or 5499 with `lsof -i :3742`.

**Hot reload not working**: Ensure `CHOKIDAR_USEPOLLING=true` is set in `.env.local`.

## Native Modules

After `npm install`, rebuild native modules inside the container:
```bash
docker exec api npm rebuild better-sqlite3
```
