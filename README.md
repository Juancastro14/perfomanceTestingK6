# perfomanceTestingK6

Repositorio del **equipo QA de BHD** para correr pruebas de carga distribuidas con k6.

## Como lanzar una prueba (QA)

1. Ir a la pestaña **Actions** -> **Run k6 load test** -> click en **Run workflow**.
2. Completar los inputs:
   - **suite**: que script correr (`load-test` por ahora; se agregaran mas).
   - **parallelism**: cuantos pods runner (los VU del script se dividen entre ellos).
   - **target_url**: URL bajo prueba.
   - **test_name_prefix**: prefijo para el TestRun. El run number se le suma para evitar colisiones (ej. `qa-load-42`).
   - **namespace**: por ahora solo `compute-resources-area-calidad-dev`.
3. Click **Run workflow**.

El workflow:
- Crea un ConfigMap con el script.
- Aplica el TestRun en el cluster.
- Espera a que los pods arranquen.
- Hace tail de los logs en vivo.
- Al final imprime un resumen y limpia los recursos.
- Las metricas van a InfluxDB y se pueden ver en Grafana filtrando por `testid=<nombre-del-testrun>`.

## Estructura

```
perfomanceTestingK6/
├── scripts/                              # los k6 scripts (.js)
│   └── load-test.js
├── manifests/
│   └── testrun.template.yaml             # plantilla del TestRun (envsubst)
├── .github/workflows/
│   └── run-test.yml                      # workflow que QA dispara desde la UI
└── README.md
```

## Agregar una nueva suite

1. Crear el script `scripts/<nombre-suite>.js`.
2. Editar [.github/workflows/run-test.yml](.github/workflows/run-test.yml) y agregar el nombre al input `suite.options`.
3. Push a `main`. La nueva suite aparece en el dropdown del Run workflow.

Convencion: cada script debe leer `__ENV.TARGET_URL` y declarar sus stages/scenarios. La carga (VU) se divide automaticamente por k6-operator entre los pods (`parallelism`).

## Bootstrap (una sola vez)

Hay tres piezas para configurar antes del primer run:

### 1) Self-hosted runner

El cluster vive como `kind` local en la maquina del equipo de plataforma — los runners hosteados por GitHub no llegan a `127.0.0.1`. Por eso el workflow usa `runs-on: self-hosted` y el runner se instala en esa misma maquina.

Pasos (los hace plataforma):

1. En este repo: **Settings -> Actions -> Runners -> New self-hosted runner**.
2. Elegir el OS (Windows en este caso) y seguir los comandos que da GitHub:
   - Bajar el `actions-runner` ZIP.
   - `./config.cmd --url https://github.com/Juancastro14/perfomanceTestingK6 --token <TOKEN>`
   - `./run.cmd` (o instalar como servicio: `./svc.sh install` en Linux, `./svc.cmd install` en Windows).
3. Verificar en Settings -> Actions -> Runners que aparezca `Idle`.

> Cuando el cluster se mude a un AKS publico, cambiar `runs-on: self-hosted` por `ubuntu-latest` en [run-test.yml](.github/workflows/run-test.yml) y desmontar el runner.

### 2) Kubeconfig como secret

El workflow necesita un kubeconfig para hablar con el cluster. Lo genera plataforma desde el repo [k6_operator](https://github.com/Juancastro14/k6_operator):

```powershell
.\scripts\build-runner-kubeconfig.ps1 -Base64 > runner-kubeconfig.b64.txt
```

Despues, en este repo: **Settings -> Secrets and variables -> Actions -> New repository secret**:
- Nombre: `KUBECONFIG_DATA`
- Valor: el contenido (en base64) del archivo generado.

### 3) Imagen del runner k6 publica en GHCR

El TestRun usa `ghcr.io/juancastro14/k6-influxdb:latest`. La buildea el CI del repo [k6_operator](https://github.com/Juancastro14/k6_operator) en cada push a `main` que toque `docker/**`. Despues del primer build, hacer la package publica una sola vez:
- https://github.com/Juancastro14?tab=packages -> `k6-influxdb` -> Package settings -> **Change visibility** -> Public.

## Limites

- El workflow tiene timeout de 30 minutos. Si tu prueba es mas larga, ajusta `timeout-minutes` en [run-test.yml](.github/workflows/run-test.yml).
- Solo un test name por run number; si el mismo prefijo se repite en runs concurrentes, los nombres difieren por `github.run_number`.
- El runner image (`ghcr.io/juancastro14/k6-influxdb:latest`) lo construye y publica el repo de plataforma.
