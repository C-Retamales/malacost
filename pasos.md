# Pasos: crear la librería de ML en Rust (proyecto independiente)

Guía paso a paso para crear, **en un repositorio aparte**, una librería de ML propia y publicable. Este documento es independiente de la app BioPipeline: la librería no debe saber nada de ella.

> Marcador de nombre: en todo el documento `<LIB>` es el nombre que elijas (ej. `<LIB>` = `midemo`). Reemplázalo al copiar los comandos.

---

## Parte A: Decisiones previas (antes de escribir código)

### A.1 Nombre
- Debe estar **libre en crates.io**: busca en https://crates.io/search?q=<LIB> y verifica que no exista.
- Conviene que esté libre también como organización/usuario en GitHub y como dominio de docs (docs.rs lo da gratis).
- Reglas del crate: minúsculas, números, `-` y `_`. Evita nombres genéricos o que se confundan con otros (`linfa`, `smartcore`, etc.).
- **Reserva el nombre pronto**: en crates.io se reserva publicando una primera versión (`0.0.1`) mínima.

### A.2 Licencia
Recomendación (estándar del ecosistema Rust): **`MIT OR Apache-2.0`** (doble licencia).
- Crea dos archivos: `LICENSE-MIT` y `LICENSE-APACHE` en la raíz.
- En cada `Cargo.toml`: `license = "MIT OR Apache-2.0"`.
- En el `README`, añade la sección estándar de licencia y contribución (ver B.7).
- Alternativas: solo MIT (más simple), o GPL/AGPL (si quieres obligar a que los derivados sean abiertos; limita su adopción en empresas). **Decídelo antes del primer commit público**, cambiarla después es complicado.
- Pon tu nombre (o el de la organización) y el año en el texto de `LICENSE-MIT`.

### A.3 Dónde alojarlo
- **Repositorio en GitHub** (recomendado). Opciones:
  - Cuenta personal: más rápido.
  - **Organización de GitHub** (gratis): mejor para una librería comunitaria, porque otros pueden ser co-mantenedores sin depender de tu cuenta personal.
- Visibilidad: puedes crearlo **privado** al inicio y hacerlo público cuando haya un primer modelo validado.
- Repositorio **separado** de BioPipeline. BioPipeline luego la consume como dependencia (por `git` primero, por crates.io después).

### A.4 Alcance de la v0.1
Define un alcance pequeño. Una librería con **2 o 3 modelos bien validados** vale más que 8 a medias. Propuesta:
- `0.1`: Ridge, ElasticNet, PLS + validación cruzada + métricas.
- `0.2`: GPR, SVR.
- `0.3`: Random Forest.
- `0.4`: boosting estilo XGBoost y MLP.

### A.5 Revisar el ecosistema (30–60 min, vale la pena)
Antes de escribir, mira qué existe para no duplicar y para aprender su API: `linfa`, `smartcore`, `friedrich` (GPR), `burn`, `gbdt`. Anota qué falta y cuál es tu diferencial (ej. PLS completo, GPR con optimización de hiperparámetros, API unificada, validación contra sklearn).

---

## Parte B: Preparar el entorno

### B.1 Herramientas
```powershell
# Rust (si no está instalado)
winget install Microsoft.VisualStudio.2022.BuildTools --override "--wait --passive --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
winget install Rustlang.Rustup
# Reabre la terminal, luego:
rustup default stable
rustup component add rustfmt clippy
```
Herramientas útiles de cargo (opcional, instálalas cuando las necesites):
```powershell
cargo install cargo-nextest      # runner de tests más rápido
cargo install cargo-deny         # auditoría de licencias y dependencias
cargo install cargo-audit        # vulnerabilidades conocidas
cargo install cargo-llvm-cov     # cobertura
cargo install cargo-release      # automatiza versiones y publicación
```

### B.2 Python para generar referencias (solo desarrollo, no se distribuye)
Se usa **solo para producir los resultados de referencia** contra los que se validan tus modelos.
```powershell
python -m venv .venv
.venv\Scripts\activate
pip install numpy scikit-learn xgboost pandas
```
`.venv/` va en `.gitignore`.

### B.3 Crear el repositorio
1. En GitHub: crear repo vacío `<LIB>` (sin README, sin licencia, para evitar conflictos).
2. En local, **fuera** de BioPipeline:
```powershell
mkdir <LIB>
cd <LIB>
git init -b main
```

### B.4 Crear el workspace
Un workspace con varios crates, más una fachada que los reexporta.

`Cargo.toml` (raíz):
```toml
[workspace]
resolver = "2"
members = ["crates/*"]

[workspace.package]
version = "0.0.1"
edition = "2021"
license = "MIT OR Apache-2.0"
repository = "https://github.com/<USUARIO_U_ORG>/<LIB>"
rust-version = "1.75"          # MSRV: ajústalo a tu versión mínima real
authors = ["Tu Nombre <tu@correo>"]

[workspace.dependencies]
faer       = "*"   # fija la versión real al crear el proyecto
thiserror  = "*"
rand       = "*"
rand_chacha = "*"
rayon      = "*"
serde      = { version = "*", features = ["derive"], optional = true }
approx     = "*"   # comparaciones numéricas en tests
```
> Sustituye `"*"` por versiones concretas (`cargo add` las resuelve). Evita `*` en un crate que vas a publicar.

Crear los crates:
```powershell
cargo new crates/<LIB>-core   --lib
cargo new crates/<LIB>-linalg --lib
cargo new crates/<LIB>-linear --lib
cargo new crates/<LIB>-pls    --lib
cargo new crates/<LIB>        --lib   # fachada
```
(Los demás, `-kernel`, `-ensemble`, `-nn`, `-selection`, se crean cuando toquen.)

### B.5 Estructura final esperada
```
<LIB>/
├─ Cargo.toml
├─ README.md
├─ LICENSE-MIT
├─ LICENSE-APACHE
├─ CHANGELOG.md
├─ CONTRIBUTING.md
├─ CODE_OF_CONDUCT.md
├─ SECURITY.md
├─ .gitignore
├─ rustfmt.toml
├─ clippy.toml            (opcional)
├─ deny.toml              (cargo-deny)
├─ .github/
│  ├─ workflows/ci.yml
│  ├─ ISSUE_TEMPLATE/
│  └─ PULL_REQUEST_TEMPLATE.md
├─ crates/
│  ├─ <LIB>-core/
│  ├─ <LIB>-linalg/
│  ├─ <LIB>-linear/
│  ├─ <LIB>-pls/
│  └─ <LIB>/
├─ tests/reference/       # datasets + resultados de referencia (ver Parte D)
│  ├─ generate/           # scripts Python que los generan
│  └─ data/
├─ benches/
├─ examples/
└─ docs/
```

### B.6 `.gitignore` mínimo
```
/target
.venv/
*.pyc
.idea/
.vscode/
Cargo.lock      # solo si es SOLO librería; ver nota
```
Nota: en librerías se solía ignorar `Cargo.lock`, pero hoy se recomienda **commitearlo** si hay ejemplos/benchmarks/CI reproducibles. Elige y sé consistente.

### B.7 README: secciones mínimas
1. Nombre y una línea de qué es.
2. Estado (`experimental`, `0.x`).
3. Instalación (`cargo add <LIB>`).
4. Ejemplo mínimo que compila.
5. Modelos soportados y su estado de validación.
6. Comparación honesta con alternativas.
7. Cómo contribuir.
8. Licencia, con el texto estándar:
```
Licensed under either of Apache License, Version 2.0 or MIT license at your option.
Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in this crate by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.
```

---

## Parte C: Diseño de la librería

### C.1 Dependencias entre crates
```
<LIB> (fachada) → linear, pls, ... → core, linalg
core   : sin dependencias de modelos
linalg : sobre faer, aislado para poder cambiar de backend
```
Ningún crate depende de GUI, gráficos, ni de la app.

### C.2 Tipos y traits base (`<LIB>-core`)
Define primero, antes de cualquier modelo:
- **Error común** con `thiserror`: dimensiones incompatibles, no convergencia, datos con NaN, hiperparámetro inválido, matriz singular.
- **Traits**:
  ```rust
  pub trait Fit<X, Y> {
      type Fitted;
      type Error;
      fn fit(&self, x: &X, y: &Y) -> Result<Self::Fitted, Self::Error>;
  }
  pub trait Predict<X> {
      type Output;
      type Error;
      fn predict(&self, x: &X) -> Result<Self::Output, Self::Error>;
  }
  ```
  Patrón recomendado: **hiperparámetros** (struct configurable con builder) ⇒ `fit` ⇒ **modelo ajustado** (struct inmutable). Así no existe un modelo "a medio ajustar" ni estado inválido.
- **Capacidades opcionales** como traits separados: `PredictWithUncertainty`, `FeatureImportance`.
- **Observador de progreso y cancelación** (opcional): trait `FitObserver { fn on_step(..); fn is_cancelled(&self) -> bool }`, para que una GUI pueda mostrar curvas sin que la librería la conozca.
- **Semilla** explícita en todo lo aleatorio (`ChaCha8Rng`, para resultados idénticos entre plataformas).

### C.3 Decisión sobre tipos de matriz
Elige una y documéntala:
- **Opción 1:** usar `faer::Mat<f64>` directamente (más simple, rápido).
- **Opción 2:** `ndarray` (más popular, mejor interoperabilidad con `linfa`).
- **Opción 3:** genérica con traits propios (flexible, más complejo).

Recomendación: empezar con una sola, ocultarla tras tipos propios (`type Matrix = ...`) y aislar la conversión en `linalg`. Esto deja abierta la puerta a otras sin romper la API.

### C.4 Capa `linalg`
Funciones que los modelos necesitan, en un solo sitio: Cholesky (con y sin jitter), solve, QR, SVD, autovalores simétricos, producto matriz-vector y matriz-matriz, normas. No escribas un BLAS propio.

### C.5 Reglas de calidad desde el inicio
- `#![forbid(unsafe_code)]` salvo excepción justificada.
- `#![deny(missing_docs)]` en los crates publicados.
- Sin `unwrap()` ni `panic!` en rutas alcanzables por el usuario.
- Cálculos en `f64`.
- Toda la aleatoriedad pasa por una semilla.

---

## Parte D: Validación contra referencias (el paso que más importa)

Un modelo "que corre" no es un modelo correcto. Antes de implementar cada modelo, prepara su test de paridad.

### D.1 Generar referencias
En `tests/reference/generate/` un script Python por modelo:
1. Fija una semilla y crea (o carga) un dataset pequeño, y guárdalo en `tests/reference/data/*.csv`.
2. Ajusta el modelo equivalente en scikit-learn (o XGBoost).
3. Guarda **hiperparámetros exactos, coeficientes/predicciones y métricas** en JSON junto al dataset.
4. Registra las versiones de Python, numpy y sklearn usadas.

### D.2 Test de paridad en Rust
El test de integración carga el CSV y el JSON, ajusta tu modelo con los mismos hiperparámetros y compara con `approx` usando una **tolerancia declarada** (ej. `1e-8` para Ridge, `1e-4` para modelos iterativos). Si no coincide, hay que entender *por qué* (a veces la referencia define distinto el término de regularización o el escalado).

### D.3 Tipos de tests por modelo
| Test | Qué comprueba |
|---|---|
| Solución conocida | Datos sintéticos con respuesta analítica |
| Paridad | Coincide con sklearn/XGBoost |
| Propiedades | Ej. Ridge con λ→0 se acerca a OLS; ElasticNet con l1_ratio=0 equivale a Ridge |
| Bordes | Datos con una variable, n < p, colinealidad, constante, NaN, dimensiones malas |
| Determinismo | Misma semilla ⇒ mismo resultado |
| Serialización | Guardar y cargar da las mismas predicciones |

Documenta en el README qué modelos pasan qué tests. Esa tabla es la que da confianza a quien la use.

---

## Parte E: Plan de implementación por hitos

Cada hito termina con: código, tests, documentación con ejemplo, entrada en `CHANGELOG`, y CI en verde.

### Hito 0: esqueleto
- Workspace creado, CI corriendo (E.0.1), `README` y licencias.
- `core` con `Error` y traits `Fit`/`Predict`.
- `linalg` con envoltorios de `faer` (solve, Cholesky) y tests.
- **Publicar `0.0.1` mínima para reservar el nombre** (ver Parte G).

### Hito 1: Ridge (primer modelo extremo a extremo)
- Hiperparámetros: `alpha`, `fit_intercept`, `normalize/scale` (decide y documenta).
- Solución cerrada vía Cholesky (o SVD si es mal condicionado).
- Test de paridad con `sklearn.linear_model.Ridge`.
- Ejemplo en `examples/ridge.rs` y doctest.
- Sirve para validar **toda la tubería**: API, errores, tests, docs, CI.

### Hito 2: ElasticNet
- Coordinate descent con soft-thresholding, criterio de convergencia por tolerancia y máximo de iteraciones, warm start.
- Opcional: camino de regularización (`alpha` en rejilla).
- Paridad con `sklearn.linear_model.ElasticNet`.

### Hito 3: PLS
- NIPALS (y/o SIMPLS). Soporta `n_components`, centrado y escalado.
- Expone scores, loadings, weights y coeficientes.
- Paridad con `sklearn.cross_decomposition.PLSRegression` (cuidado con las diferencias de convención: signos y escalado).

### Hito 4: selección de modelos y métricas
- K-fold, K-fold repetido, por grupos.
- Métricas: RMSE, MAE, R², Q².
- Búsqueda: grid y random sobre hiperparámetros.
- **Publicar `0.1.0`.**

### Hito 5: GPR
- Kernels (RBF, Matérn, constante, ruido, y sumas/productos).
- Cholesky con jitter, predicción con media y desviación.
- Log-verosimilitud marginal y optimización de hiperparámetros (L-BFGS) con varios reinicios.
- Paridad con `sklearn.gaussian_process.GaussianProcessRegressor`.

### Hito 6: SVR RBF
- ε-SVR con solver SMO, cache de kernel y shrinking.
- Paridad con `sklearn.svm.SVR` (que usa libsvm).
- Benchmarks de escalado con n.

### Hito 7: Random Forest
- Árbol CART de regresión, bagging, `max_features`, OOB.
- Paralelismo con `rayon`.
- Paridad **estadística** (no exacta, es aleatorio): R² e importancias parecidos, no idénticos.

### Hito 8: boosting estilo XGBoost
- Gradiente y hessiano, histogramas, regularización L1/L2, `learning_rate`, `max_depth`, submuestreo.
- Es el más costoso: planifícalo por sub-hitos (primero un árbol, luego boosting simple, luego histogramas).

### Hito 9: MLP
- Capas densas, activaciones, Adam, L2, early stopping, semilla.
- CPU primero. GPU es una decisión posterior.

### Orden recomendado
Ridge → ElasticNet → PLS → métricas/CV → (publicar 0.1) → GPR → SVR → RF → boosting → MLP.

---

## Parte F: Integración continua (CI) y calidad

### F.1 `.github/workflows/ci.yml`: qué debe hacer
1. **Matriz de SO:** Windows, Linux, macOS.
2. **Matriz de Rust:** `stable` y tu MSRV (`rust-version`).
3. Pasos: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --all`, `cargo doc --no-deps`.
4. Trabajos adicionales (pueden ir aparte): `cargo deny check`, `cargo audit`, cobertura.

### F.2 Archivos de comunidad
- `CONTRIBUTING.md`: cómo compilar, cómo correr tests, estilo, qué se espera en un PR (tests de paridad incluidos).
- `CODE_OF_CONDUCT.md`: Contributor Covenant.
- `SECURITY.md`: cómo reportar vulnerabilidades en privado.
- Plantillas de issue (bug, feature) y de PR.
- Etiquetas: `good first issue`, `help wanted`, `numerical`, `performance`.

### F.3 Benchmarks
- `criterion` en `benches/`. Mide `fit` y `predict` según n y p.
- Publica los resultados **incluyendo comparaciones donde pierdas** frente a sklearn/XGBoost. La honestidad construye credibilidad.
- No corras benchmarks en el CI normal (son ruidosos); hazlo bajo demanda.

### F.4 Documentación
- `rustdoc` con ejemplos ejecutables en cada tipo público.
- Una página por modelo: algoritmo, ecuaciones, referencias bibliográficas, hiperparámetros, límites conocidos.
- Se publica automáticamente en **docs.rs** al subir a crates.io.

---

## Parte G: Publicación en crates.io

### G.1 Preparación
1. Crear cuenta en https://crates.io (se inicia sesión con GitHub).
2. Generar un token de API (Account Settings → API Tokens) y guardarlo con `cargo login`. **No lo subas nunca al repo.**
3. Cada crate publicable necesita en su `Cargo.toml`: `description`, `license`, `repository`, `readme`, `keywords` (máx. 5), `categories`.
4. Si usas dependencias entre crates del workspace, indica **ruta y versión**:
   ```toml
   <LIB>-core = { path = "../<LIB>-core", version = "0.1.0" }
   ```

### G.2 Orden de publicación
Se publica de las **hojas hacia arriba**, porque cada crate debe encontrar sus dependencias ya publicadas:
`<LIB>-core` → `<LIB>-linalg` → `<LIB>-linear` → `<LIB>-pls` → `<LIB>` (fachada).

### G.3 Comprobaciones antes de publicar
```powershell
cargo publish --dry-run -p <LIB>-core     # simula, sin publicar
cargo package --list -p <LIB>-core        # revisa qué archivos entran
```
Una publicación **no se puede borrar**; solo se puede "yank" (retirar de nuevas resoluciones). Revisa dos veces.

### G.4 Versionado
- **SemVer.** En `0.x`, un cambio incompatible sube la versión menor (`0.1` → `0.2`).
- Actualiza `CHANGELOG.md` en cada versión (formato Keep a Changelog).
- Crea una etiqueta de git por versión (`v0.1.0`) y una *release* en GitHub.
- `cargo-release` automatiza todo esto.

### G.5 Después de publicar
- Verifica que docs.rs compiló la documentación.
- Anuncia en: r/rust, users.rust-lang.org, This Week in Rust (envío de PR a su repo), y la comunidad de `rust-ml`.
- Prepara respuestas a "¿por qué no usar `linfa`/`smartcore`?" (tu comparación honesta del A.5).

---

## Parte H: Conectar con BioPipeline (más adelante)

1. Mientras la librería no esté publicada, BioPipeline la usa por Git:
   ```toml
   <LIB> = { git = "https://github.com/<USUARIO_U_ORG>/<LIB>", tag = "v0.1.0" }
   ```
2. Desarrollo simultáneo: dependencia por **ruta local** en un `[patch]` o directamente `path = "../<LIB>"`, solo en tu máquina (no lo commitees).
3. Una vez en crates.io: `<LIB> = "0.1"`.
4. En BioPipeline, el crate `ml` solo adapta la librería al `trait Model` de la app (ver `arquitectura.md`). La librería no cambia por las necesidades de la GUI.

---

## Parte I: Checklist resumida

**Antes de empezar**
- [ ] Nombre elegido y libre en crates.io y GitHub
- [ ] Licencia decidida (`MIT OR Apache-2.0` recomendado)
- [ ] Repo/organización en GitHub creado (puede ser privado)
- [ ] Rust, Build Tools y Python instalados

**Hito 0**
- [ ] Workspace, `LICENSE-*`, `README`, `.gitignore`, `rustfmt.toml`
- [ ] CI con matriz de SO
- [ ] `core` (errores y traits) y `linalg`
- [ ] `0.0.1` publicada para reservar el nombre

**Hito 1 en adelante, por cada modelo**
- [ ] Referencia generada con sklearn/XGBoost y guardada en `tests/reference`
- [ ] Implementación
- [ ] Test de paridad con tolerancia declarada
- [ ] Tests de propiedades y de bordes
- [ ] Ejemplo + doctest + página de documentación
- [ ] Entrada en `CHANGELOG`
- [ ] Tabla de validación del `README` actualizada

**Lanzamiento `0.1.0`**
- [ ] `cargo publish --dry-run` de todos los crates
- [ ] Publicación en orden (G.2)
- [ ] Etiqueta y *release* en GitHub
- [ ] Documentación visible en docs.rs
- [ ] Anuncio

---

## Riesgos conocidos

| Riesgo | Mitigación |
|---|---|
| Resultados numéricos "casi correctos" | Tests de paridad desde el primer modelo, tolerancias declaradas |
| Rendimiento inferior a sklearn/XGBoost | Benchmarks honestos; optimizar solo lo medido; usar `faer` |
| Alcance demasiado grande | Publicar `0.1` con 3 modelos; el resto en hitos posteriores |
| Cambios de API tras publicar | Mantener `0.x` hasta estabilizar; marcar tipos como `#[non_exhaustive]` cuando proceda |
| Nombre ocupado a último momento | Reservar con `0.0.1` temprano |
| Licencia equivocada | Decidirla antes del primer commit público |
| Mantenimiento en solitario | Organización de GitHub, `CONTRIBUTING`, etiquetas para nuevos colaboradores |
