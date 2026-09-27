# NFC Android

Librería / app de referencia para interactuar con **NFC** en Android: lectura de tags, escritura NDEF, host card emulation (HCE) y emulación de tarjetas. Construida en **Kotlin** + **Jetpack Compose** con una API simple, moderna y testeable.

> **Read · Write · Emulate · Scan.** Una sola librería, sin dependencias nativas raras, sin wrappers legacy.

---

## Stack

| Capa | Tecnología |
|---|---|
| Lenguaje | Kotlin 1.9 · JDK 17 |
| UI | Jetpack Compose · Material 3 |
| Async | Kotlin Coroutines · Flow |
| NFC | `android.nfc.NfcAdapter` · `Ndef` · `Tag` · `HostApduService` |
| Persistencia | DataStore Preferences |
| DI | Hilt |
| Testing | JUnit 5 · Espresso · MockK |

**Min SDK:** 24 (Android 7) · **Target SDK:** 34 · **Permissions:** `NFC`

---

## Features

- 🔍 **Read** NDEF tags (Text, URI, MIME, Smart Poster, custom records)
- ✏️ **Write** NDEF messages a tags formateables
- 👤 **Emulate** tarjetas contactless via Host Card Emulation (ISO 14443-4)
- 📡 **Scan & inventory** — múltiples tags en sesión, deduplicados por UID
- 🧩 **Extension parsers** — NTAG, MIFARE Classic, Ultralight, DESFire (vía APDU directo)
- 🎯 **Type filter** — filtra por tipo de tag al disparar Activity
- 🔔 **Foreground dispatch** — captura tags mientras la app está al frente
- 🧪 **Tests** — instrumented + unitarios + fakes de `Tag`

---

## Quick start

### 1. Añadir dependencia (cuando esté publicada)

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://jitpack.io") }
    }
}

// app/build.gradle.kts
dependencies {
    implementation("com.aguitech:nfc-android:1.0.0")
}
```

### 2. Declarar en Manifest

```xml
<uses-permission android:name="android.permission.NFC" />
<uses-feature android:name="android.hardware.nfc" android:required="true" />

<application>
    <activity android:name=".MainActivity">
        <intent-filter>
            <action android:name="android.nfc.action.NFC_TECH_DISCOVERED" />
            <category android:name="android.intent.category.DEFAULT" />
        </intent-filter>
    </activity>
    
    <!-- HCE service -->
    <service
        android:name=".emulation.NfcCardService"
        android:exported="true"
        android:permission="android.permission.BIND_NFC_SERVICE">
        <intent-filter>
            <action android:name="android.nfc.cardemulation.action.HOST_APDU_SERVICE" />
            <category android:name="android.intent.category.DEFAULT" />
        </intent-filter>
    </service>
</application>
```

### 3. Leer un tag

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val nfc = NfcManager(this)
        
        // Pedir permission (Android 13+: opcional, ya viene granted)
        if (!nfc.isNfcEnabled()) {
            startActivity(Intent(Settings.ACTION_NFC_SETTINGS))
        }
        
        // Procesar intent inicial (si app fue abierta por tap)
        intent?.let { nfc.handleIntent(it) }
        
        // Enable foreground dispatch
        nfc.enableForegroundDispatch(this) { tag ->
            when (val rec = tag.readNdef()) {
                is NdefRecord.Text -> println("text: ${rec.text}")
                is NdefRecord.Uri -> openUrl(rec.uri)
                else -> println("raw: $rec")
            }
        }
    }
    
    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        nfcManager.handleIntent(intent)
    }
    
    override fun onPause() {
        super.onPause()
        nfcManager.disableForegroundDispatch(this)
    }
}
```

### 4. Escribir un tag

```kotlin
val writer = NdefWriter(connection)
writer.write(
    NdefMessage(
        NdefRecord.uri("https://aguitech.com"),
        NdefRecord.text("Tap to claim your billboard"),
    )
)
```

### 5. Host Card Emulation

```kotlin
class NfcCardService : HostApduService() {
    override fun processCommandApdu(data: ByteArray, extras: Bundle?): ByteArray {
        // SELECT AID
        if (data.size >= 2 && data[0] == 0x00.toByte() && data[1] == 0xA4.toByte()) {
            return byteArrayOf(
                0x90.toByte(), // success
                0x00.toByte()
            )
        }
        return byteArrayOf(0x6F.toByte(), 0x00.toByte()) // error
    }
    
    override fun onDeactivated(reason: Int) {
        // handle deactivate
    }
}
```

---

## Api surface

```kotlin
// Core
class NfcManager(activity: Activity)
class NdefReader(connection: TagConnection)
class NdefWriter(connection: TagConnection)
class NdefFormatter(message: NdefMessage)

// HCE
abstract class HostApduService : Service()

// Models
sealed class NdefRecord {
    data class Text(val text: String, val lang: String) : NdefRecord()
    data class Uri(val uri: Uri) : NdefRecord()
    data class Mime(val type: String, val payload: ByteArray) : NdefRecord()
    data class External(val domain: String, val type: String, val payload: ByteArray) : NdefRecord()
    data class Custom(val tnf: Short, val type: ByteArray, val payload: ByteArray) : NdefRecord()
}

// Result wrapper (no exceptions thrown across the API)
sealed class NfcResult<out T> {
    data class Success<T>(val value: T) : NfcResult<T>()
    data class Failure(val error: NfcError, val cause: Throwable? = null) : NfcResult<Nothing>()
}

sealed class NfcError(val message: String) {
    object NotSupported : NfcError("Device doesn't support NFC")
    object Disabled : NfcError("NFC is turned off in system settings")
    object TagLost : NfcError("Connection to the tag was lost")
    object FormatError : NfcError("Tag is not NDEF-formatted")
    object ReadOnly : NfcError("Tag is read-only")
    object NdefTooBig : NfcError("Message too large for tag capacity")
    object IOException : NfcError("I/O error during tag communication")
}
```

---

## Tech detection

Al tocar un tag, la librería detecta automáticamente el tipo:

| Tag Type | Tech List |
|---|---|
| NDEF | `Ndef` · `NdefFormatable` |
| NTAG213/215/216 | `NfcA` · `Ndef` · `MifareUltralight` |
| MIFARE Classic | `NfcA` · `MifareClassic` |
| MIFARE DESFire | `NfcA` · `IsoDep` |
| FeliCa | `NfcF` |
| ISO 14443-4 | `IsoDep` |

```kotlin
val tech = tag.detectTech() // TagTech.NTAG215, TagTech.Desfire, etc.
```

---

## Demo app

El módulo `app/` incluye una app demo con 3 pantallas:

1. **Read** —Acerca un tag, muestra payload parseado
2. **Write** —Formulario → escribe NDEF (URI + Text) al tag
3. **Emulate** —Activa HCE service, lee tarjetas contactless

```bash
./gradlew :app:installDebug
adb shell am start -n com.aguitech.nfc/.MainActivity
```

---

## Permissions & Settings

```kotlin
// Android 13+: NFC no requiere runtime permission, pero sí capability check
fun checkNfc(context: Context): NfcResult<Unit> = when {
    !context.packageManager.hasSystemFeature(PackageManager.FEATURE_NFC) ->
        NfcResult.Failure(NfcError.NotSupported)
    !NfcAdapter.getDefaultAdapter(context).isEnabled ->
        NfcResult.Failure(NfcError.Disabled)
    else -> NfcResult.Success(Unit)
}
```

---

## Testing

```bash
# unit tests
./gradlew :nfc:testDebugUnitTest

# instrumented (requiere device con NFC)
./gradlew :nfc:connectedDebugAndroidTest

# con fakes (sin device)
./gradlew :nfc:testDebugUnitTest --tests="*Fake*"
```

Los tests usan **FakeNfcTag** y **FakeHostApduService** — no requieren device físico.

---

## Roadmap

- [x] NDEF read/write
- [x] Foreground dispatch
- [x] HCE service base
- [ ] APDU builder DSL (typed protocol)
- [ ] NDEF formatter con pretty-print
- [ ] MIFARE Classic read/write (key map)
- [ ] DESFire EV1/EV2 (con EV2 secure messaging)
- [ ] NTAG password protection
- [ ] Multi-tag inventory (peer-to-peer session)
- [ ] Compose preview para Tag payloads
- [ ] Sample app: NFC Billboard Inspector (caso AGUITECH)

---

## Casos de uso

- 🏙️ **Inventario de espectaculares** — audit de medios físicos outdoor (caso Medidor de Espectaculares)
- 🎟️ **Tickets NFC** — entradas a eventos con counter anti-duplicado
- 💳 **Wallets internos** —badge corporativo, monedero con HCE
- 🚚 **Logística** — check-in/out de paquetería, lectores handheld
- 🔐 **Auth factors** — segundo factor físico en login

---

## Licencia

MIT © 2026 AGUITECH

---

## Créditos

**AGUITECH** — ingeniería de producto end-to-end.  
Construido por Héctor Aguilar · `hector@aguitech.com`
