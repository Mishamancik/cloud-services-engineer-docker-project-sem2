





### Горизонтальное масштабирование
Для горизотального масштабирования подходит только stateless backend, так как для frontend просто собираются статичные файлы. Для runtime во frontend используется nginx, который работает как точка входа и обеспечивает балансировку запросов к множественным backend-инстансам.

Чтобы масштабировать backend, используйте флаг в команде запуска:
``` 
docker compose up -d --scale backend=3 
```


### Сканирование образов
Backend
```
Report Summary

┌────────────────────────────────────┬──────────┬─────────────────┬─────────┐
│               Target               │   Type   │ Vulnerabilities │ Secrets │
├────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ momo-backend:4.0.1 (alpine 3.23.4) │  alpine  │        0        │    -    │
├────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ app/main                           │ gobinary │       79        │    -    │
└────────────────────────────────────┴──────────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)


app/main (gobinary)
===================
Total: 79 (UNKNOWN: 0, LOW: 1, MEDIUM: 44, HIGH: 30, CRITICAL: 4)
```

Frontend
```
Report Summary

┌─────────────────────────────────────┬────────┬─────────────────┬─────────┐
│               Target                │  Type  │ Vulnerabilities │ Secrets │
├─────────────────────────────────────┼────────┼─────────────────┼─────────┤
│ momo-frontend:4.0.2 (alpine 3.23.4) │ alpine │        0        │    -    │
└─────────────────────────────────────┴────────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)
```


## Проверка docker-bench-security
Некоторые результаты с WARN-статусом
```
[WARN]       * Container running with root FS mounted R/W: momo-backend-1
[WARN]       * Container running with root FS mounted R/W: momo-frontend-1
[WARN]       * PIDs limit not set: momo-backend-1
[WARN]       * PIDs limit not set: momo-frontend-1
[WARN]       * Port being bound to wildcard IP: 0.0.0.0 in momo-frontend-1
[WARN]       * Privileges not restricted: momo-backend-1
[WARN]       * Privileges not restricted: momo-frontend-1
[WARN]      * No SecurityOptions Found: momo-backend-1
[WARN]      * No SecurityOptions Found: momo-frontend-1
[WARN]      * Port in use: 8080 in momo-frontend-1
```
