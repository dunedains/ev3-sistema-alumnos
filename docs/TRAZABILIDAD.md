# Registro de trazabilidad - Sistema Alumnos

Cada fila la agrega Jenkins al terminar un build (`scripts/trazabilidad.py`). Copia del artefacto `reports/TRAZABILIDAD.md`
del build #6 (primer build en verde con todas las etapas de seguridad). Los builds siguientes agregan su fila en Jenkins.

| Build | Fecha (UTC) | Versión | Commit | Mensaje del commit | Pruebas | Bandit H/M/L | pip-audit | ZAP H/M/L | Resultado | Producción |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 2026-11-23 19:35 | 1.0.1+9b3d738 | `9b3d738` | Registro de seguridad (plantilla) | 9/9 OK | n/a | n/a | n/a | SUCCESS | 1.0.1+9b3d738 |
| 2 | 2026-11-23 19:37 | 1.0.2+eecf6a3 | `eecf6a3` | Pipeline: etapas SAST (Bandit), SCA (pip-audit), Deploy Staging, DAST  | 9/9 OK | 2/2/1 | 19 vuln. | 3/3/1 | UNSTABLE | no desplegado |
| 3 | 2026-11-23 19:42 | 1.0.3+8ddbc2d | `8ddbc2d` | VULN-07: cabeceras de seguridad HTTP: CSP, X-Frame-Options, nosniff, R | 9/9 OK | 0/0/0 | 19 vuln. | 0/1/0 | UNSTABLE | no desplegado |
| 4 | 2026-11-23 19:45 | 1.0.4+ec920df | `ec920df` | DEP-01: habilitar Dependabot para dependencias pip (requirements.txt) | 9/9 OK | 0/0/0 | 19 vuln. | 0/0/0 | UNSTABLE | no desplegado |
| 5 | 2026-11-23 19:57 | 1.0.5+9b3cc97 | `9b3cc97` | DOC-01: registro de seguridad (revisiones, VULN-01..09, resultados ZAP | 9/9 OK | 0/0/0 | 19 vuln. | 0/0/0 | UNSTABLE | no desplegado |
| 6 | 2026-11-23 20:00 | 1.0.6+b14dde5 | `b14dde5` | DEP-02: merge PR #1 de Dependabot (Flask 3.1.3, Werkzeug 3.1.9, Jinja2 | 9/9 OK | 0/0/0 | 0 vuln. | 0/0/0 | SUCCESS | 1.0.6+b14dde5 |
