# xconductor — Named Pipe Protokolü

Bir oyun uygulamasının xconductor ile nasıl konuşacağını açıklayan kılavuz.

---

## Genel Mimari

xconductor ile oyun arasında **iki ayrı pipe** vardır:

| Pipe | Yön | Açıklama |
|---|---|---|
| **Game Input Pipe** | xconductor → Oyun | Oyun konfigürasyonunu buradan alır |
| **Game Output Pipe** | Oyun → xconductor | Skor ve diğer verileri buraya gönderir |

Her iki pipe'ta da mesajlar **UTF-8 JSON**, satır başı `\n` ile sonlandırılır.

> [!IMPORTANT]
> **Pipe'ları yalnızca xconductor oluşturur ve yönetir.** Oyun uygulaması hiçbir zaman `mkfifo()` veya `CreateNamedPipe()` çağırmamalıdır. xconductor pipe'ı oluşturur, oyun sadece `open()` / `CreateFile()` ile **bağlanır**. Bağlantı koptuğunda xconductor otomatik olarak 2 saniyede bir yeniden denediğinden oyun da aynı şekilde yeniden bağlanma döngüsü kurmalıdır.

---

## Pipe İsimleri

Pipe isimleri her zaman `game_input` ve `game_output`'tur. Platform bazında tam yol:

| Platform | Game Input Pipe | Game Output Pipe |
|---|---|---|
| **Linux / macOS** | `/tmp/game_input` | `/tmp/game_output` |
| **Windows** | `\\.\pipe\game_input` | `\\.\pipe\game_output` |

---

## Platform Bazında Bağlanma Kuralları

### Linux / macOS — POSIX FIFO

xconductor pipe'ı `mkfifo()` ile oluşturur. Oyun uygulaması yalnızca `open()` ile bağlanır.

**Game Input Pipe** (oyun okur):
```c
// Oyun tarafı — pipe'ı oluşturma yok, sadece bağlan:
fd = open("/tmp/game_input", O_RDONLY);  // blocking: xconductor bağlanana kadar bekler
```

**Game Output Pipe** (oyun yazar):
```c
fd = open("/tmp/game_output", O_WRONLY); // blocking: xconductor bağlanana kadar bekler
```

> [!IMPORTANT]
> `open()` **blocking** modda çalışır; karşı taraf bağlanmadan dönmez.  
> Oyun uygulaması her iki pipe'ı **ayrı thread**'lerde açmalıdır, aksi hâlde kilitlenir.

Bağlantı koptuğunda (EOF/EPIPE) xconductor 2 saniye bekleyip otomatik yeniden bağlanır. Oyun da aynı şekilde yeniden bağlanma döngüsü kurmalıdır.

---

### Windows — Named Pipe

xconductor pipe'ı `CreateNamedPipe()` ile server olarak oluşturur ve `ConnectNamedPipe()` ile oyunu bekler.

Oyun uygulaması yalnızca `CreateFile()` ile bağlanır. Her pipe **yalnızca kendi yönünde** açılmalıdır:

```c
// Game Input Pipe — sadece okuma (oyun config alır)
HANDLE hIn  = CreateFileW(L"\\\\.\\pipe\\game_input",  GENERIC_READ,  0, NULL, OPEN_EXISTING, 0, NULL);

// Game Output Pipe — sadece yazma (oyun skor gönderir)
HANDLE hOut = CreateFileW(L"\\\\.\\pipe\\game_output", GENERIC_WRITE, 0, NULL, OPEN_EXISTING, 0, NULL);
```

> [!IMPORTANT]
> xconductor `game_input`'u `PIPE_ACCESS_OUTBOUND`, `game_output`'u `PIPE_ACCESS_INBOUND` olarak açar. Oyun ters yönde (`GENERIC_WRITE` ile `game_input`, `GENERIC_READ` ile `game_output`) bağlanmaya çalışırsa bağlantı başarısız olur.

**Güvenlik:** Pipe'lar `D:(A;;GA;;;WD)(A;;GA;;;AN)(A;;GA;;;AU)` SDDL ile açılır — farklı user context'lerindeki process'ler sorunsuz bağlanabilir.

---

## Mesaj Formatı

Tüm mesajlar tek satır **compact JSON** + `\n` karakteri:

```
{"type":"...", ...}\n
```

Buffer'da `\n` görülene kadar biriktirilir, sonra parse edilir. Eksik gelen parçalar birleştirilir.

---

## xconductor'dan Oyuna Gelen Mesajlar (Input Pipe)

### `b_game_config` — Oyun Başlangıç Konfigürasyonu

Oyun süreci başlatıldıktan sonra xconductor'ın **ilk MQTT heartbeat'i aldığında** bir kez gönderilir.

```json
{
  "gameId": "laser-maze",
  "difficulty": "normal",
  "type": "classic",
  "countdownDuration": 10,
  "gameDuration": 120,
  "exitDuration": 15,
  "playerNames": ["Ahmet", "Zeynep"],
  "highScores": [
    { "username": "Player1", "score": 1000 },
    { "username": "Player2", "score": 800 },
    { "username": "Player3", "score": 600 }
  ],
  "language": "tr"
}
```

| Alan | Tip | Açıklama |
|---|---|---|
| `gameId` | string | Oyunun benzersiz kimliği |
| `difficulty` | string | `"normal"`, `"easy"`, `"hard"` vb. |
| `type` | string | Oyun varyantı, genellikle `"classic"` |
| `countdownDuration` | number | Oyun öncesi geri sayım süresi (saniye) |
| `gameDuration` | number | Oyunun süresi (saniye) |
| `exitDuration` | number | Oyun bittikten sonra çıkış bekleme süresi (saniye) |
| `playerNames` | string[] | Bu session'daki oyuncuların isimleri |
| `highScores` | object[] | En yüksek 20 skor (yoksa mock data) |
| `language` | string | Kiosk'tan gelen dil kodu (varsa eklenir) |

> [!NOTE]
> `language` alanı xconductor tarafından **enjekte** edilir. xlocale'den gelen ham config'de bu alan yoktur; xconductor kendi önbelleğinden ekler.

---

## Oyundan xconductor'a Gönderilecek Mesajlar (Output Pipe)

### 1. Skor / Oyun Verisi

Oyun bitiminde gönderilir. xconductor bu mesajı olduğu gibi MQTT'e iletir.

```json
[
  { "playerName": "Ahmet", "score": 4200 },
  { "playerName": "Zeynep", "score": 3800 }
]
```

| Alan | Tip | Açıklama |
|---|---|---|
| `playerName` | string | `b_game_config`'deki isimle eşleşmeli (`username` da kabul edilir) |
| `score` | number | Oyuncu skoru |

---

### 2. Oyun Sonu Sinyali (Opsiyonel)

Oyun kendi kendine erken biterse bunu bildirmek için kullanılabilir. xconductor `room/{zoneId}/g_end` MQTT topic'ini tetikler.

```json
{"gameEnd": true}
```

> [!NOTE]
> Normal akışta xconductor zamanlayıcıya göre `b_stop` gönderir. Oyun bu mesajı yalnızca erken/beklenmedik bitiş durumunda göndermeli.

---

## Tam Akış Özeti

```
xlocale                   xconductor                Oyun Uygulaması
   |                           |                           |
   |                           |-- Pipe oluştur & bekle    |
   |                           |<========================= | (oyun pipe'a bağlandı)
   |                           |                           |
   |<-- MQTT: xconductor_hb -- |                           | (her 1 saniyede)
   |                           |                           |
   |  (ilk heartbeat geldi)    |                           |
   |-- MQTT: b_game_config --> |                           |
   |                           |-- pipe: b_game_config --> |
   |                           |                           |
   |           (oyun süreci)   |                           |
   |                           |<-- [{"playerName":...}] - | (oyun sonu)
   |<-- MQTT: g_score -------- |                           |
   |                           |                           |
   |<-- MQTT: b_stop --------- |                           |
```

> xconductor **kendisi** her saniye `room/{zoneId}/xconductor_heartbeat` MQTT mesajı gönderir. xlocale bunu alınca game config'i iletir. Oyunun ayrıca heartbeat göndermesi gerekmez.
