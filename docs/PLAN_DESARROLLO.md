# Plan de desarrollo — App de seguimiento de actividad física

## 1. Visión

App móvil que permite al usuario:

1. Registrar su perfil físico (edad, sexo, peso, altura, nivel de actividad, FC en reposo).
2. Elegir un tipo de ejercicio e iniciar una sesión.
3. Vincular su smartwatch y ver su **ritmo cardiaco en tiempo real** durante la actividad.
4. Obtener una estimación de **calorías quemadas** basada en su perfil y en su frecuencia cardiaca.
5. Consultar el historial y su progreso.

### Dispositivos objetivo

| Dispositivo | Sistema | Notas |
|---|---|---|
| **Nothing Phone (3)** | Android (Nothing OS, con Google Play Services) | Health Connect viene integrado en el sistema. |
| **Samsung Galaxy Watch6 Classic** | Wear OS (One UI Watch) | Se vincula al Nothing con la app *Galaxy Wearable* + *Samsung Health*. |

Sin iOS por ahora: todo el proyecto es **Android + Wear OS**.

---

## 2. Decisiones técnicas

Como el reloj es Wear OS y la app del reloj **tiene que ser nativa** (React Native/Flutter no tienen soporte real en Wear OS), lo más eficiente es hacer **todo en Kotlin** y compartir código entre teléfono y reloj.

| Área | Elección | Motivo |
|---|---|---|
| Lenguaje | **Kotlin** | Nativo en Android y Wear OS. |
| UI teléfono | Jetpack Compose + Material 3 | |
| UI reloj | Compose for Wear OS (pantalla redonda, bisel giratorio del Watch6 Classic con `rotaryScrollable`) | |
| FC en el reloj | **Health Services — `ExerciseClient`** | API oficial de Wear OS para sesiones de ejercicio; FC ~1 lectura/s, funciona con pantalla apagada. |
| Reloj ⇄ teléfono | **Wearable Data Layer API** (`MessageClient` para FC en vivo, `DataClient` para el resumen) | Usa la conexión Bluetooth que ya tiene el reloj con el teléfono. |
| Datos de salud | **Health Connect** (`androidx.health.connect:connect-client`) | Leer peso, FC en reposo e historial de Samsung Health; escribir los entrenamientos. |
| Persistencia | Room (SQLite) | Funciona sin conexión. |
| Arquitectura | MVVM + Hilt + Coroutines/Flow | |
| Gráficas | Vico | Gráficas en Compose. |
| Backend (fase 4) | Supabase (Postgres + Auth) | Respaldo y sincronización, opcional. |
| Pruebas | JUnit + Turbine (Flows), Compose UI tests | |
| IDE | Android Studio (con emulador de Wear OS) | |

---

## 3. Integración con el Galaxy Watch6 Classic

### Lo que **no** sirve para FC en vivo
- **Bluetooth estándar de FC (servicio `0x180D`)**: el Galaxy Watch no emite su ritmo cardiaco así de forma nativa.
- **Health Connect / Samsung Health**: Samsung Health sincroniza con Health Connect con retraso (minutos). Sirve para el historial, no para el ritmo durante la actividad.

### Lo que sí: app propia en el reloj
```
[Galaxy Watch6]                                   [Nothing Phone 3]
 App Wear OS                                        App Android
  ExerciseClient (Health Services)                   WearableListenerService
   └─ FC cada ~1 s ──► MessageClient ──Bluetooth──►   └─ calcula calorías, zonas
  Pantalla: FC, zona, tiempo                         Pantalla en vivo + gráfica
  Controles: iniciar / pausar / terminar  ◄──────►   Controles espejo
                                                     Al terminar: Room + Health Connect
```

- El **reloj es la fuente de verdad durante la sesión**: guarda las muestras localmente, así no se pierde nada si el teléfono se desconecta, y las reenvía al reconectar.
- Ambas apps comparten el mismo `applicationId` y firma (requisito del Data Layer).
- Se puede iniciar la sesión desde el reloj o desde el teléfono.
- `ExerciseClient` también entrega sus propias calorías estimadas: las guardaremos para comparar con nuestro cálculo.
- **Samsung Health Sensor SDK** (acceso a sensores en bruto, IBI/HRV): opcional y más adelante; para publicar requiere aprobación de Samsung. No hace falta para el objetivo actual.

### Permisos
- Reloj: `BODY_SENSORS` (o `health.READ_HEART_RATE` en Wear OS 5+), `ACTIVITY_RECOGNITION`, `FOREGROUND_SERVICE_HEALTH`.
- Teléfono: permisos de Health Connect (lectura de peso/FC, escritura de ejercicio), `POST_NOTIFICATIONS`.

### Entorno de pruebas en tu hardware
1. Reloj: *Ajustes → Información del reloj → Software* → tocar 7 veces "Versión de software" para activar opciones de desarrollador.
2. Activar **Depuración ADB** y **Depuración por Wi-Fi** en el reloj; conectar con `adb pair` / `adb connect` desde Android Studio.
3. Teléfono: activar depuración USB en el Nothing Phone.
4. Al principio se puede usar el emulador de Wear OS con datos de FC sintéticos (Health Services los simula).

---

## 4. Cálculo de calorías

Módulo Kotlin puro (`:core:domain`), sin dependencias de Android, compartido por ambas apps y con pruebas unitarias.

### 4.1 Con frecuencia cardiaca (método principal) — Keytel et al., 2005
Por minuto, con FC = frecuencia media del intervalo, P = peso (kg), E = edad (años):

- **Hombres:** `kcal/min = (-55.0969 + 0.6309·FC + 0.1988·P + 0.2017·E) / 4.184`
- **Mujeres:** `kcal/min = (-20.4022 + 0.4472·FC - 0.1263·P + 0.0740·E) / 4.184`

Se integra cada 5–10 s. Válido aprox. para FC entre 90 y 150 lpm; fuera de ese rango (o si se pierde la señal) se usa el método MET.

### 4.2 Sin frecuencia cardiaca — método MET corregido
- `kcal = MET × 3.5 × P / 200 × minutos`
- **Corrección por perfil:** metabolismo basal con **Mifflin-St Jeor** (peso, altura, edad, sexo) y `MET_corregido = MET × 3.5 / VO2_reposo_individual`, donde `VO2_reposo = (TMB / 1440 / 5) / P × 1000` (ml/kg/min).
- Valores MET del *Compendium of Physical Activities* (Ainsworth). Ejemplos:

| Ejercicio | MET |
|---|---|
| Caminar 5 km/h | 3.5 |
| Correr 8 km/h | 8.3 |
| Correr 10 km/h | 9.8 |
| Bicicleta moderada (16–19 km/h) | 6.8 |
| Natación moderada | 5.8 |
| Pesas / fuerza moderada | 3.5 |
| HIIT | 8.0 |
| Yoga | 2.5 |

### 4.3 Datos derivados
- **FC máxima:** Tanaka `208 − 0.7 × edad` (editable).
- **FC en reposo:** leída de Health Connect (Samsung Health la mide) o introducida a mano.
- **Zonas de FC:** Karvonen (Z1 50–60 % … Z5 90–100 %), con vibración en el reloj al cambiar de zona.

> Mostrar siempre que las calorías son una **estimación**, no un dato médico.

---

## 5. Modelo de datos (simplificado)

```
UserProfile   { id, birthDate, sex, weightKg, heightCm, restingHr?, maxHrOverride?, units }
ExerciseType  { id, name, icon, met, category, healthServicesType, usesGps }
Workout       { id, exerciseTypeId, startedAt, endedAt, durationSec, avgHr, maxHr,
                calories, calorieMethod ('hr'|'met'|'mixed'), watchCalories?,
                distanceM?, source ('watch'|'phone'), notes? }
HrSample      { workoutId, timestamp, bpm }
WeightLog     { date, weightKg }
```

---

## 6. Pantallas

**Teléfono**
1. Onboarding / Perfil (con opción de importar peso y altura desde Health Connect).
2. Inicio — resumen semanal, estado del reloj, botón "Empezar actividad".
3. Selección de ejercicio — lista con búsqueda y favoritos.
4. Sesión en curso — cronómetro, FC grande con color de zona, gráfica en vivo, calorías.
5. Resumen de sesión — FC media/máx, tiempo por zona, calorías (nuestras vs. del reloj).
6. Historial y estadísticas.
7. Ajustes — permisos, Health Connect, exportación.

**Reloj**
1. Lista de ejercicios (navegable con el bisel).
2. Sesión: FC, zona, tiempo, calorías; pausar/terminar.
3. Resumen breve.
4. *Tile* y *complicación* de Wear OS para iniciar rápido (fase 5).

---

## 7. Estructura del proyecto (Gradle multi-módulo)

```
app/              # app del teléfono (Compose)
wear/             # app del reloj (Compose for Wear OS + Health Services)
core/
  domain/         # calories.kt, heartRateZones.kt, metTable.kt  ← Kotlin puro + tests
  data/           # Room, repositorios, Health Connect
  wearcomm/       # rutas y modelos de mensajes del Data Layer (compartido)
docs/
```

---

## 8. Fases y entregables

### Fase 0 — Preparación (≈1 semana)
- Proyecto Android Studio con módulos `app`, `wear`, `core:*`.
- CI con GitHub Actions (build + tests).
- Activar depuración en el Nothing Phone y en el Galaxy Watch6; desplegar un "hola mundo" en ambos.
- **Criterio de salida:** ambas apps instaladas desde Android Studio en tus dispositivos.

### Fase 1 — App del teléfono sin reloj (≈2 semanas)
- Perfil y persistencia local (Room).
- Catálogo de ejercicios con MET.
- Sesión con cronómetro y cálculo MET corregido.
- Resumen e historial básico.
- Tests unitarios del motor de calorías.
- **Criterio de salida:** registrar una sesión y verla en el historial.

### Fase 2 — App del reloj con FC en vivo (≈3 semanas)
- App Wear OS con `ExerciseClient`: iniciar, pausar, terminar, FC en vivo.
- Servicio en primer plano para que la sesión siga con la pantalla apagada.
- Envío de FC al teléfono por `MessageClient`; resumen final por `DataClient`.
- Teléfono: FC en vivo, gráfica, zonas, calorías por Keytel con respaldo MET.
- Reconexión: muestras guardadas en el reloj y reenviadas.
- **Criterio de salida:** sesión de 30 min caminando/corriendo con el Watch6 sin huecos de datos; calorías dentro de ±15 % de las de Samsung Health para la misma actividad.

### Fase 3 — Health Connect (≈1–2 semanas)
- Leer peso, altura y FC en reposo (desde Samsung Health).
- Escribir los entrenamientos (`ExerciseSessionRecord`, `HeartRateRecord`, `TotalCaloriesBurnedRecord`).
- Estadísticas semanales/mensuales.
- **Criterio de salida:** un entrenamiento de la app aparece en Health Connect y en Samsung Health.

### Fase 4 — Nube (opcional, ≈2 semanas)
- Cuenta de usuario y sincronización con Supabase.

### Fase 5 — Extras
- GPS (reloj o teléfono) y mapa para correr/bici, metas y rachas, Tile y complicación en el reloj, alertas de zona por vibración, exportación GPX/CSV, HRV con Samsung Health Sensor SDK, soporte para bandas pectorales BLE.

---

## 9. Riesgos y mitigación

| Riesgo | Mitigación |
|---|---|
| Ahorro de batería de Nothing OS / One UI Watch cierra las apps en segundo plano | Servicio en primer plano con notificación en ambos lados; pedir excluir la app de la optimización de batería; probar con pantalla bloqueada. |
| Pérdida de conexión reloj ⇄ teléfono | El reloj guarda la sesión completa y sincroniza al reconectar. |
| Galaxy Watch con teléfono no Samsung | Las funciones necesarias (Wear OS, Health Services, Data Layer) no dependen de un teléfono Samsung. Solo ECG/presión arterial lo requieren, y no se usan. |
| Precisión de calorías | Fórmulas documentadas, mostrar el método usado, comparar con las calorías del reloj, tests con valores de referencia. |
| Datos de salud sensibles | Datos locales por defecto, consentimiento explícito y política de privacidad (Google Play la exige para Health Connect y sensores corporales). |

---

## 10. Próximos pasos inmediatos
1. Instalar Android Studio (con SDK de Wear OS) si aún no está.
2. Activar depuración en el Nothing Phone 3 y el Galaxy Watch6 Classic.
3. Iniciar la Fase 0.
