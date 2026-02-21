# Documentación Técnica – Pipeline CI/CD con Enfoque DevSecOps

**Diplomado de Desarrollo de Software – Módulo DevSecOps**  
**Estudiante:** Jaime Bacotich  
**Repositorio:** https://github.com/jaimebacotich/practica_devsecops  
**Base:** https://github.com/sancano22/practica2  

---

## 1. Arquitectura General del Sistema

El proyecto consiste en una aplicación web con arquitectura de microservicios:

| Componente | Tecnología | Puerto |
|---|---|---|
| `frontend` | React + Vite | 80 |
| `users-service` | Node.js + Express | 3001 |
| `academic-service` | Node.js + Express | 3002 |
| `api-gateway` | Node.js + http-proxy-middleware | 3000 |

Todos los servicios se construyen y ejecutan en contenedores Docker, orquestados con Docker Compose.

---

## 2. Diseño Global del Pipeline CI/CD

El pipeline sigue el flujo completo **Code → Test → Security → Build → Deploy** con enfoque **DevSecOps** ("shift-left security"), integrando seguridad en cada etapa del ciclo de vida del software.

```
┌─────────────┐   ┌──────────────┐   ┌────────────────┐   ┌──────────────┐   ┌─────────────┐
│  CÓDIGO     │──▶│  CALIDAD /   │──▶│   SEGURIDAD    │──▶│    BUILD     │──▶│   SMOKE     │
│  Checkout   │   │  TEST        │   │  SAST + SCA    │   │  Docker      │   │   TEST      │
│  Node setup │   │  ESLint      │   │  Semgrep       │   │  Trivy scan  │   │  API GW     │
│  .env setup │   │  Jest        │   │  Trivy fs      │   │  imágenes    │   │  Health     │
└─────────────┘   └──────────────┘   └────────────────┘   └──────────────┘   └─────────────┘
```

### Secuencia completa de etapas

| # | Etapa | Herramienta | Fase DevSecOps |
|---|---|---|---|
| 1 | Checkout del código | `actions/checkout@v4` | Dev |
| 2 | Setup Node.js v20 | `actions/setup-node@v4` | Dev |
| 3 | Crear archivos `.env` | bash `touch` | Dev |
| 4 | Install & Test – users-service | `npm ci` + ESLint + Jest | Dev / Sec |
| 5 | Install & Test – academic-service | `npm ci` + ESLint + Jest | Dev / Sec |
| 6 | Install & Test – api-gateway | `npm ci` + ESLint + Jest | Dev / Sec |
| 7 | Install & Test – frontend | `npm ci` + ESLint + Jest | Dev / Sec |
| 8 | SAST – users-service | Semgrep | Sec |
| 9 | SAST – academic-service | Semgrep | Sec |
| 10 | SAST – api-gateway | Semgrep | Sec |
| 11 | Validate env files | bash script | Sec |
| 12 | SCA – filesystem scan | Trivy `scan-type: fs` | Sec |
| 13 | Build Docker images | `docker build` | Ops |
| 14 | Trivy scan – users-service (imagen) | `aquasecurity/trivy-action` | Sec / Ops |
| 15 | Trivy scan – academic-service (imagen) | `aquasecurity/trivy-action` | Sec / Ops |
| 16 | Trivy scan – api-gateway (imagen) | `aquasecurity/trivy-action` | Sec / Ops |
| 17 | Run docker-compose | `docker compose up -d` | Ops |
| 18 | Smoke test API Gateway | `curl` health check | Ops |
| 19 | Shutdown services | `docker compose down` | Ops |

---

## 3. Selección y Ubicación de Herramientas

### 3.1 `npm ci` – Instalación Reproducible

| Campo | Detalle |
|---|---|
| **Fase DevSecOps** | Dev – Integración continua |
| **Riesgo que mitiga** | Instalaciones inconsistentes entre entornos (funciona en local, falla en producción) |
| **Por qué es necesaria** | `npm install` puede resolver versiones distintas cada ejecución. `npm ci` garantiza exactamente las versiones del `package-lock.json`, asegurando reproducibilidad. |
| **Consecuencia de no usarla** | Dependencias distintas por entorno; bugs difíciles de reproducir; inconsistencias silenciosas. |

### 3.2 ESLint – Calidad de Código

| Campo | Detalle |
|---|---|
| **Fase DevSecOps** | Dev – Análisis estático de calidad |
| **Riesgo que mitiga** | Variables no usadas, código muerto, errores comunes de JavaScript, inconsistencias de estilo |
| **Configuración** | ESLint v9 con `@eslint/js` y `globals` para entorno Node.js |
| **Por qué es necesaria** | Detecta errores en tiempo de escritura antes de que lleguen al runtime. Garantiza estándares de código uniforme en equipo. |
| **Consecuencia de no usarla** | Deuda técnica acumulada, bugs por descuido (variables undefined, comparaciones erróneas), código difícil de mantener. |

### 3.3 Jest – Testing Automático

| Campo | Detalle |
|---|---|
| **Fase DevSecOps** | Dev – Testing continuo |
| **Riesgo que mitiga** | Regresiones funcionales; funcionalidades rotas que pasan desapercibidas |
| **Tests implementados** | Health check endpoint (`/health`) por servicio |
| **Por qué es necesaria** | El pipeline se detiene automáticamente si un test falla, impidiendo que código roto llegue a producción. |
| **Consecuencia de no usarla** | Cambios que rompen funcionalidad existente se despliegan sin detección; tiempo de debugging en producción. |

### 3.4 Semgrep – SAST (Static Application Security Testing)

| Campo | Detalle |
|---|---|
| **Fase DevSecOps** | Sec – Seguridad del código fuente |
| **Riesgo que mitiga** | Vulnerabilidades en el código: inyecciones, uso inseguro de eval(), secrets hardcodeados, XSS, CSRF, SSRF |
| **Reglas aplicadas** | `p/javascript` (mejores prácticas JS) + `p/nodejsscan` (vulnerabilidades específicas Node.js) |
| **Por qué es necesaria** | Detecta vulnerabilidades de seguridad *antes del despliegue*, cuando el costo de corrección es mínimo. Un sistema funcional puede tener graves vulnerabilidades de seguridad. |
| **Consecuencia de no usarla** | Vulnerabilidades llegan a producción; exposición a ataques de inyección, robo de datos, escalación de privilegios. |

### 3.5 Trivy (Filesystem) – SCA (Software Composition Analysis)

| Campo | Detalle |
|---|---|
| **Fase DevSecOps** | Sec – Seguridad de dependencias |
| **Riesgo que mitiga** | Librerías npm con CVEs conocidos (ej: `cross-spawn` RCE, `tar` path traversal, `minimatch` ReDoS) |
| **Configuración** | `scan-type: fs`, severity `CRITICAL,HIGH`, `exit-code: 1` |
| **Por qué es necesaria** | El 85% de las aplicaciones modernas usan librerías de terceros. Una dependencia vulnerable es un vector de ataque real aunque el código propio esté limpio. |
| **Consecuencia de no usarla** | Dependencias con CVEs conocidos en producción; explotación mediante supply-chain attacks. |

### 3.6 Docker Build + Docker Buildx – Containerización

| Campo | Detalle |
|---|---|
| **Fase DevSecOps** | Ops – Build de artefactos |
| **Riesgo que mitiga** | Inconsistencias de entorno entre desarrollo y producción |
| **Por qué es necesaria** | Garantiza que la misma imagen que se prueba es la que se despliega (immutable artifact). Docker Buildx permite builds multi-plataforma. |
| **Consecuencia de no usarla** | "Funciona en mi máquina" – entornos inconsistentes, bugs que solo aparecen en producción. |

### 3.7 Trivy (Image Scan) – Seguridad de Contenedores

| Campo | Detalle |
|---|---|
| **Fase DevSecOps** | Sec / Ops – Seguridad de la imagen |
| **Riesgo que mitiga** | Vulnerabilidades en el sistema operativo base de la imagen Docker (librerías del SO, capas intermedias) |
| **Configuración** | `image-ref` por servicio, severity `CRITICAL,HIGH`, `exit-code: 1` |
| **Por qué es necesaria** | Una imagen puede contener vulnerabilidades a nivel de SO (ej: OpenSSL, glibc) aunque el código de la app esté limpio. |
| **Consecuencia de no usarla** | Contenedores con CVEs conocidos en producción; escape de contenedor, escalación de privilegios a nivel de host. |

### 3.8 Curl Health Check – Smoke Test

| Campo | Detalle |
|---|---|
| **Fase DevSecOps** | Ops – Verificación post-despliegue |
| **Riesgo que mitiga** | Despliegues silenciosamente rotos donde los servicios no responden |
| **Por qué es necesaria** | Valida que el sistema completo es funcional después del build, no solo que los tests unitarios pasan. |
| **Consecuencia de no usarla** | Imágenes "correctas" que no arrancan correctamente en producción. |

---

## 4. Automatización y Security Gates

El pipeline actúa como un **sistema de control automático**. Cada etapa es un gate: si falla, el código **no avanza**.

```
Código push ──▶ [Gate 1: Lint] ──▶ [Gate 2: Tests] ──▶ [Gate 3: SAST] 
              ──▶ [Gate 4: SCA] ──▶ [Gate 5: Docker Build] ──▶ [Gate 6: Trivy Image]
              ──▶ [Gate 7: Smoke Test] ──▶ ✅ Aprobado
```

| Gate | Herramienta | Condición de bloqueo |
|---|---|---|
| Calidad de código | ESLint | Cualquier error de lint |
| Tests | Jest | Cualquier test que falle |
| SAST | Semgrep (`--error`) | Cualquier vulnerabilidad detectada |
| SCA | Trivy fs (`exit-code: '1'`) | CVE CRITICAL o HIGH en dependencias |
| Imagen Docker | Trivy image (`exit-code: '1'`) | CVE CRITICAL o HIGH en la imagen |

### Gates de seguridad explícitos en el YAML

```yaml
# SAST – bloquea si hay vulnerabilidades en código
semgrep --config=p/javascript --config=p/nodejsscan ... --error

# SCA – bloquea si hay CVEs en dependencias
exit-code: '1'
severity: 'CRITICAL,HIGH'

# Imagen Docker – bloquea si hay CVEs en la imagen
exit-code: '1'
severity: 'CRITICAL,HIGH'
```

El pipeline opera bajo el principio **"fail fast"**: detiene la ejecución en el primer gate fallido, ahorrando tiempo y evitando despliegues inseguros o rotos.

---

## 5. Principios DevSecOps Aplicados

| Principio | Implementación |
|---|---|
| **Shift-Left Security** | Seguridad se verifica antes del build, no después |
| **Fail Fast** | `exit-code: 1` en todos los scans de seguridad |
| **Reproducibilidad** | `npm ci` en vez de `npm install` |
| **Inmutabilidad** | Imágenes Docker como artefactos inmutables |
| **Automatización total** | Todo ejecutado por GitHub Actions, sin intervención manual |
| **Principio de mínima confianza** | Validación de `.env` para detectar secretos expuestos accidentalmente |

---

## 6. Evidencia de Ejecución

El pipeline se ejecuta automáticamente en cada `push` o `pull request` a la rama `main`.

- **Repositorio:** https://github.com/jaimebacotich/practica_devsecops
- **GitHub Actions:** https://github.com/jaimebacotich/practica_devsecops/actions
- **Pipeline file:** `.github/workflows/devsecops.yml`
