# Plan de desarrollo — App de seguimiento de actividad física

## 1. Visión

App móvil que permite al usuario:

1. Registrar su perfil físico (edad, sexo, peso, altura, nivel de actividad, FC en reposo).
2. Elegir un tipo de ejercicio e iniciar una sesión.
3. Vincular un smartwatch / banda pectoral y ver su **ritmo cardiaco en tiempo real** durante la actividad.
4. Obtener una estimación de **calorías quemadas** basada en su perfil y, si hay datos, en su frecuencia cardiaca.
5. Consultar el historial y su progreso.

---

## 2. Decisiones técnicas propuestas

| Área | Propuesta | Motivo |
|---|---|---|
| App móvil | **React Native + Expo (dev build)**, TypeScript | Un solo código para Android e iOS; acceso a módulos nativos (BLE, HealthKit, Health Connect). |
| Estado | Zustand + React Query | Simple, sin boilerplate. |
| Persistencia local | SQLite (`expo-sqlite`) + Drizzle ORM | Funciona sin conexión; las sesiones nunca se pierden. |
| Backend (fase 3) | Supabase (Postgres + Auth) | Sincronización entre dispositivos y respaldo, sin servidor propio al inicio. |
| BLE | `react-native-ble-plx` | Lectura del perfil estándar *Heart Rate Service*. |
| Salud del sistema | `react-native-health` (iOS HealthKit) y `react-native-health-connect` (Android) | Leer FC histórica y guardar los entrenamientos. |
| Gráficas | `victory-native` o `react-native-gifted-charts` | FC en vivo y progreso. |
| Pruebas | Jest + React Native Testing Library; Maestro para E2E | |

> Alternativa: si se prefiere nativo puro, Kotlin (Android/Wear OS) + Swift (iOS/watchOS). Más trabajo, mejor integración con el reloj.

---

## 3. Integración con smartwatch (el punto crítico)

Hay tres vías y **no son equivalentes** en latencia:

| Vía | Tiempo real | Dispositivos | Fase |
|---|---|---|---|
| **A. BLE Heart Rate Profile** (servicio `0x180D`, característica `0x2A37`) | Sí (~1 lectura/s) | Bandas pectorales (Polar, Garmin HRM, Wahoo), Polar/Coros/Suunto, Garmin con *"Transmitir FC"*, muchas pulseras genéricas | **MVP** |
| **B. HealthKit / Health Connect** | No (datos con retraso de minutos) | Apple Watch, Wear OS, Samsung, Fitbit, Garmin (vía sus apps) | Fase 2 — historial y sincronización |
| **C. App compañera en el reloj** (watchOS / Wear OS) | Sí | Apple Watch, Wear OS (Pixel, Galaxy Watch) | Fase 4 |

**Importante:** el Apple Watch y la mayoría de relojes Wear OS **no emiten FC por BLE estándar**. Para FC en vivo con ellos hace falta la vía C (app en el reloj con `HKWorkoutSession` en watchOS o *Health Services* en Wear OS que envía datos al teléfono). Por eso el MVP empieza con BLE, que cubre el mayor número de dispositivos con menos esfuerzo.

### Flujo de vinculación BLE
1. Pedir permisos (Bluetooth / ubicación en Android < 12, `BLUETOOTH_SCAN` + `BLUETOOTH_CONNECT` en Android 12+).
2. Escanear filtrando por servicio `0x180D`.
3. Conectar, suscribirse a notificaciones de `0x2A37` y decodificar (bit 0 del byte de flags: FC en 8 o 16 bits).
4. Guardar el dispositivo para reconexión automática.
5. Manejar desconexiones: reintentos, aviso al usuario, y la sesión continúa (cálculo por MET si no hay FC).

---

## 4. Cálculo de calorías

El motor de cálculo será un módulo puro en TypeScript (`src/domain/calories.ts`), sin dependencias de UI, con pruebas unitarias.

### 4.1 Con frecuencia cardiaca (método preferido) — Keytel et al., 2005
Por minuto, con FC = frecuencia media del intervalo, P = peso (kg), E = edad (años):

- **Hombres:** `kcal/min = (-55.0969 + 0.6309·FC + 0.1988·P + 0.2017·E) / 4.184`
- **Mujeres:** `kcal/min = (-20.4022 + 0.4472·FC - 0.1263·P + 0.0740·E) / 4.184`

Se integra cada 5–10 s con la FC de ese tramo. Válido aprox. para FC entre 90 y 150 lpm; fuera de ese rango se combina con el método MET.

### 4.2 Sin frecuencia cardiaca — método MET corregido
- `kcal = MET × 3.5 × P / 200 × minutos`
- **Corrección por perfil:** se calcula el metabolismo basal con **Mifflin-St Jeor** (usa peso, altura, edad y sexo) y se ajusta el MET: `MET_corregido = MET × 3.5 / VO2_reposo_individual`, donde `VO2_reposo = (TMB / 1440 / 5) / P × 1000` (ml/kg/min). Así la altura y la edad influyen aunque no haya reloj.
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
- **FC máxima:** Tanaka `208 − 0.7 × edad` (editable por el usuario).
- **Zonas de FC:** método Karvonen con FC en reposo (Z1 50–60 % … Z5 90–100 %).
- **IMC** informativo en el perfil.

> Mostrar siempre que las calorías son una **estimación**, no un dato médico.

---

## 5. Modelo de datos (simplificado)

```
UserProfile   { id, birthDate, sex, weightKg, heightCm, restingHr?, maxHrOverride?, units }
ExerciseType  { id, name, icon, met, category, usesGps }
Device        { id, name, bleId, type, lastConnectedAt }
Workout       { id, exerciseTypeId, startedAt, endedAt, durationSec, avgHr, maxHr,
                calories, calorieMethod ('hr'|'met'|'mixed'), distanceM?, notes? }
HrSample      { workoutId, timestamp, bpm }
WeightLog     { date, weightKg }   // las calorías usan el peso vigente en la fecha
```

---

## 6. Pantallas

1. **Onboarding / Perfil** — datos físicos, unidades (kg/lb, cm/ft).
2. **Inicio** — resumen de la semana, botón "Empezar actividad".
3. **Selección de ejercicio** — lista con búsqueda y favoritos.
4. **Dispositivos** — escanear, vincular, estado de batería y conexión.
5. **Sesión en curso** — cronómetro, FC grande con color de zona, gráfica en vivo, calorías acumuladas, pausar/reanudar/terminar. Pantalla siempre encendida.
6. **Resumen de sesión** — tiempo, FC media/máx, tiempo por zona, calorías, método usado.
7. **Historial y estadísticas** — lista, filtros por tipo, gráficas semanales/mensuales.
8. **Ajustes** — permisos, exportación, Health Connect / HealthKit.

---

## 7. Estructura del proyecto

```
src/
  app/            # pantallas (expo-router)
  components/
  domain/         # calories.ts, heartRateZones.ts, metTable.ts  ← lógica pura con tests
  services/
    ble/          # escaneo, conexión, parser 0x2A37
    health/       # HealthKit / Health Connect
    workout/      # máquina de estados de la sesión (idle → running ⇄ paused → finished)
  db/             # esquema y migraciones SQLite
  store/
docs/
```

---

## 8. Fases y entregables

### Fase 0 — Preparación (≈1 semana)
- Inicializar Expo + TypeScript, ESLint/Prettier, CI con GitHub Actions (lint + tests).
- Development build en Android e iOS (BLE no funciona en Expo Go).
- **Criterio de salida:** la app arranca en un dispositivo físico desde CI/local.

### Fase 1 — MVP sin reloj (≈2 semanas)
- Perfil de usuario y persistencia local.
- Catálogo de ejercicios con MET.
- Sesión con cronómetro y cálculo MET corregido.
- Resumen e historial básico.
- Tests unitarios del motor de calorías con casos de referencia.
- **Criterio de salida:** registrar una sesión completa y verla en el historial.

### Fase 2 — Smartwatch por BLE (≈2–3 semanas)
- Pantalla de dispositivos: escaneo, vinculación, reconexión.
- FC en vivo en la sesión, gráfica y zonas.
- Cálculo de calorías por FC (Keytel) con respaldo MET si se pierde la señal.
- Guardado de muestras de FC.
- **Criterio de salida:** sesión de 30 min con banda pectoral sin cortes; calorías dentro de ±15 % de lo que reporta el propio dispositivo.

### Fase 3 — Plataformas de salud + nube (≈2–3 semanas)
- Health Connect (Android) y HealthKit (iOS): leer peso, FC en reposo e historial; escribir entrenamientos.
- Cuenta de usuario y sincronización con Supabase.
- Estadísticas semanales/mensuales.
- **Criterio de salida:** un entrenamiento hecho en la app aparece en Google Fit/Salud de Apple.

### Fase 4 — Apps en el reloj (≈4+ semanas)
- App watchOS (Swift, `HKWorkoutSession` + `WatchConnectivity`).
- App Wear OS (Kotlin, *Health Services* + *Data Layer API*).
- Iniciar/terminar la sesión desde el reloj.

### Fase 5 — Extras
- GPS y mapa para correr/bici, metas y rachas, notificaciones, alertas de zona por vibración, exportación GPX/CSV, modo oscuro, localización.

---

## 9. Riesgos y mitigación

| Riesgo | Mitigación |
|---|---|
| Apple Watch / Wear OS no transmiten FC por BLE | Comunicarlo en la UI; Fase 4 con apps nativas en el reloj. |
| Sistema mata la app en segundo plano durante la sesión | Servicio en primer plano (Android) y modo *background BLE* (iOS); probar en pantalla bloqueada. |
| Precisión de calorías | Documentar las fórmulas, mostrar el método usado, tests con valores de referencia. |
| Datos de salud sensibles | Datos locales por defecto, cifrado en reposo, consentimiento explícito, cumplimiento de políticas de las tiendas para HealthKit/Health Connect. |
| Fragmentación de permisos BLE en Android | Capa de permisos centralizada y probada en Android 10, 12 y 14. |

---

## 10. Próximos pasos inmediatos
1. Confirmar el stack (React Native/Expo vs. nativo).
2. Definir qué relojes/bandas tiene el equipo para pruebas.
3. Iniciar la Fase 0.
