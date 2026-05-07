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

Para que el workflow pueda hablar con el cluster necesita un kubeconfig guardado como secret de GitHub. Esa configuracion la genera el equipo de plataforma desde el repo [k6_operator](https://github.com/Juancastro14/k6_operator) (ver `scripts/build-runner-kubeconfig.ps1` alli).

Resumen del bootstrap:
1. Plataforma aplica la Application `k6-tests-rbac` (crea ServiceAccount + Role + token Secret).
2. Plataforma genera el kubeconfig: `.\scripts\build-runner-kubeconfig.ps1 > runner-kubeconfig.yaml` y lo da en base64.
3. Configurar en este repo: **Settings -> Secrets and variables -> Actions -> New repository secret**:
   - Nombre: `KUBECONFIG_DATA`
   - Valor: el contenido base64 generado.

## Limites

- El workflow tiene timeout de 30 minutos. Si tu prueba es mas larga, ajusta `timeout-minutes` en [run-test.yml](.github/workflows/run-test.yml).
- Solo un test name por run number; si el mismo prefijo se repite en runs concurrentes, los nombres difieren por `github.run_number`.
- El runner image (`ghcr.io/juancastro14/k6-influxdb:latest`) lo construye y publica el repo de plataforma.
