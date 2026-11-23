# Registro de seguridad · Sistema Alumnos

Estudiante: Felipe   Repositorio: https://github.com/dunedains/ev3-sistema-alumnos   Fecha: 23-11-2026

Completa este registro a medida que avanzas. Cada fila debe poder seguirse hasta un **build de Jenkins**
y un **commit** (usa el identificador `VULN-xx` en el mensaje del commit).

Pipeline: `sistema-alumnos-pipeline` (Jenkins) → Build → Test → Security Review (SAST) → Dependency Management (SCA)
→ Deploy Staging → DAST - OWASP ZAP → **[gate]** → Deploy Producción. Cualquier hallazgo Medium/High deja el build
UNSTABLE y el gate impide desplegar en producción.

## 1. Revisiones de seguridad realizadas

| # | Etapa del SDLC | Herramienta / técnica | Build | Resultado (resumen) |
|---|---|---|---|---|
| R1 | Código (SAST) | Bandit en el pipeline (etapa *Security Review (SAST)*) | #2 | 2 hallazgos High / 2 Medium / 1 Low (B324 MD5, B201 debug, B608 SQL, B104 0.0.0.0, B105 secreto). Tras VULN-01..07: #3 = 0/0/0 |
| R2 | Dependencias (SCA) | pip-audit en el pipeline (etapa *Dependency Management*) | #2 | 19 vulnerabilidades en 4 paquetes (Flask 2, Werkzeug 10, Jinja2 5, gunicorn 2). Ver sección 5 |
| R3 | Pruebas dinámicas (DAST) | OWASP ZAP 2.16.1 sobre staging (spider + pasivo + activo) | #2 | 3 High / 3 Medium / 1 Low. Tras correcciones: #3 = 0/1/0, #4 = 0/0/0 |
| R4 | Revisión manual | Lectura de `app.py` | — | Confirma los hallazgos y agrega: errores SQL expuestos (CWE-209), login sin CSRF (CWE-352) y sin límite de intentos (CWE-307) |
| R5 | Producción | Grafana + Prometheus (dashboard *Sistema Alumnos · Producción (seguridad)* y 2 reglas de alerta) | — | Incidente de fuerza bruta + inyección detectado el 23-11 16:52 (ver sección 4) |

## 2. Vulnerabilidades identificadas y corregidas

| ID | Vulnerabilidad (CWE) | Detectada por | Ubicación | Riesgo | Corrección aplicada | Commit | Build que lo verifica |
|---|---|---|---|---|---|---|---|
| VULN-01 | Inyección SQL (CWE-89) | Bandit B608 + ZAP *SQL Injection* + manual | `app.py:205` (`/alumnos/buscar`) | Alto | Consulta parametrizada `LIKE ?` con el valor como parámetro | `f41732e` | #3 (Bandit 0, ZAP sin SQLi) |
| VULN-02 | XSS reflejado (CWE-79) | ZAP *XSS (Reflected)* y *(DOM Based)* + manual | `app.py:154` (`/hello`) | Alto | `markupsafe.escape()` sobre el parámetro `name` | `a64e634` | #3 (ZAP sin XSS) |
| VULN-03 | Credenciales/clave secreta embebidas (CWE-798) | Bandit B105 + manual | `app.py:19-23` | Alto | `SECRET_KEY` y hash del admin desde variables de entorno; clave aleatoria si falta | `f029477` | #3 |
| VULN-04 | Hash débil MD5 sin sal (CWE-327/916) | Bandit B324 (High) | `app.py:221` (`/login`) | Alto | `werkzeug.security.check_password_hash` (scrypt/pbkdf2 con sal) + `hmac.compare_digest` | `56c6a0f` | #3 |
| VULN-05 | Modo debug de Flask y bind 0.0.0.0 (CWE-489/605) | Bandit B201 (High) y B104 | `app.py:241` | Alto | `debug=False` y `host="127.0.0.1"` al ejecutar directo (en despliegue: gunicorn) | `f6f3163` | #3 |
| VULN-06 | Exposición de errores SQL (CWE-209) | Manual (respuesta 500 con la consulta) | `app.py:208-209` | Medio | Mensaje genérico al cliente; detalle solo en el log | `1f74122` | #3 |
| VULN-07 | Falta de cabeceras de seguridad (CWE-693/1021) | ZAP *CSP Header Not Set*, *Missing Anti-clickjacking*, *X-Content-Type-Options* | Todas las rutas | Medio | `after_request` con CSP, X-Frame-Options DENY, nosniff, Referrer-Policy, Permissions-Policy | `8ddbc2d` | #3 (ZAP 0/1/0) |
| VULN-08 | Ausencia de token anti-CSRF (CWE-352) | ZAP *Absence of Anti-CSRF Tokens* | Formulario `/` → `POST /login` | Medio | Token `csrf_token` en sesión + validación con `hmac.compare_digest`; cookie HttpOnly + SameSite=Lax | `9b9bd6a` | #4 (ZAP 0/0/0) |
| VULN-09 | Sin límite de intentos de login / fuerza bruta (CWE-307) | Manual + Grafana (incidente) | `app.py` `/login` | Alto | Máx. 5 fallidos por IP en 5 min → HTTP 429 y métrica `peticiones_sospechosas_total{tipo="fuerza_bruta"}` | `dd5bb6f` | #4 |

## 3. Resultados de OWASP ZAP

Build #2 (código vulnerable): **3 High / 3 Medium / 1 Low**. Build #3: 0/1/0. Build #4: **0/0/0** (solo 2 informativas).

| Alerta de ZAP | Riesgo | URL de ejemplo | VULN relacionada | Cómo se mitiga | ¿Mitigada? Build |
|---|---|---|---|---|---|
| Cross Site Scripting (Reflected) | High | `/hello?name=</h1><scrIpt>alert(1);</scRipt><h1>` | VULN-02 | Escapar la salida (`escape`) + CSP | Sí · #3 |
| Cross Site Scripting (DOM Based) | High | `/hello?name=<script>alert(5397)</script>` | VULN-02 | Escapar la salida + CSP `default-src 'none'` | Sí · #3 |
| SQL Injection | High | `/alumnos/buscar?nombre='` | VULN-01 / VULN-06 | Consulta parametrizada y error genérico | Sí · #3 |
| Content Security Policy (CSP) Header Not Set | Medium | `/` (5 URL) | VULN-07 | Cabecera `Content-Security-Policy` en todas las respuestas | Sí · #3 |
| Missing Anti-clickjacking Header | Medium | `/` (3 URL) | VULN-07 | `X-Frame-Options: DENY` + `frame-ancestors 'none'` | Sí · #3 |
| Absence of Anti-CSRF Tokens | Medium | `/` (formularios) | VULN-08 | Token anti-CSRF por sesión validado en `/login` | Sí · #4 |
| X-Content-Type-Options Header Missing | Low | `/` (6 URL) | VULN-07 | `X-Content-Type-Options: nosniff` | Sí · #3 |

## 4. Monitoreo e incidentes

Dashboard: *EV3 SecureDev › Sistema Alumnos · Producción (seguridad)* (Prometheus `http://prometheus:9090`, refresco 5 s).
Reglas (evaluación cada 10 s): **Fuerza bruta en /login** (> 5 logins fallidos/bloqueados en 2 min) y
**Patrones de ataque XSS/SQLi** (> 2 peticiones sospechosas en 2 min).

| Hora | Panel / métrica de Grafana | Qué se observó | Entorno | Clasificación | Acción de respuesta | Verificación |
|---|---|---|---|---|---|---|
| 23-11 16:52 | *Logins fallidos (5 min)* / `login_fallidos_total` | 30 logins fallidos en < 1 min desde una sola IP; 1 login exitoso al final (clave `admin` adivinada). Alerta **Fuerza bruta** en *Firing* | Producción 1.0.1+9b3d738 | Incidente **alto**: fuerza bruta exitosa sobre la cuenta admin (CWE-307 + CWE-798/327) | Desplegar la versión corregida (VULN-03/04/09) por el pipeline; credencial del admin fuera del código | Ver fila de verificación (sección 6) |
| 23-11 16:52 | *Peticiones sospechosas por tipo* / `peticiones_sospechosas_total{tipo}` | Picos de `sqli`, `xss` y `path_traversal`; la inyección SQL devolvió el esquema de la BD. Alerta **XSS/SQLi** en *Firing* | Producción 1.0.1+9b3d738 | Incidente **alto**: explotación de inyección SQL y XSS reflejado | Desplegar VULN-01/02/06/07 (gate del pipeline) | Ver sección 6 |
