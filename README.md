# Rise Online Clientless

Rise Online World (mobil) hesabıyla oyun olmadan giriş yapıp sunucu listesini çeken komut satırı aracı ve protokol üzerine notlar.

Oyun Unreal Engine 5.4 ile yazılmış. Giriş için bir JSON api'si, oyun içi trafik için Unreal'ın kendi UDP replikasyon protokolü kullanılıyor.

```
$ rise-clientless -user kullanici -password sifre
ID  SUNUCU  YOĞUNLUK
51  Aragon  3
```

## Gereksinimler

Go 1.24 veya üstü ve bir Rise Online hesabı.

## Derleme

```
make build
./bin/rise-clientless -user kullanici -password sifre
```

ya da direkt:

```
go run ./cmd/rise-clientless -user kullanici -password sifre
```

## Kullanım

| Parametre | Varsayılan | Açıklama |
|-----------|------------|----------|
| `-user` | | Kullanıcı adı |
| `-password` | | Şifre |
| `-code` | | E-postaya gelen doğrulama kodu |
| `-token` | | Daha önce alınmış oturum anahtarı |
| `-version` | `1.18.14055` | Oyun sürümü |
| `-url` | `https://mobile-login.riseonlineworld.com` | Login api adresi |
| `-json` | `false` | Listeyi json olarak yazdırır |
| `-timeout` | `20s` | İstek zaman aşımı |

### E-posta doğrulaması

Hesapta e-posta doğrulaması açıksa her girişte e-postaya yeni bir kod geliyor ve sadece en son gelen kod geçerli oluyor. `-code` verilmezse program kodu terminalden soruyor.

Doğrulamadan sonra alınan oturum anahtarı ekrana yazdırılıyor. Anahtar yaklaşık bir hafta geçerli, bu süre boyunca `-token` ile kullanıcı adı ve şifre vermeden listeyi çekebilirsin:

```
rise-clientless -token eyJhbGciOi...
```

### Oyun güncellenince

Sunucu eski sürümle gelen girişleri "Lütfen oyunun en güncel sürümünü kullanınız" diyerek reddediyor. Oyun güncellendiğinde Google Play'deki yeni sürüm numarasını `-version` ile vermen yeterli:

```
rise-clientless -user kullanici -password sifre -version 1.19.xxxxx
```

## Genel yapı

```
  istemci                                   sunucu
  ───────                                   ──────
  HTTPS  /pre-login  ───────────────────▶  mobile-login.riseonlineworld.com
         /login
         /security   (e-posta doğrulaması açıksa)
         /servers    ◀──────────────────  sunucu listesi
                                             │
                                             ▼ oturum anahtarı
  UDP    StatelessConnect el sıkışması ──▶  oyun sunucusu
         NMT_Hello → NMT_Challenge
         NMT_Login → NMT_Welcome        ◀── harita + oyun modu
         replikasyon / RPC
```

Giriş kısmı tamamen düz HTTPS ve JSON. Oyun sunucusu tarafında paket şifrelemesi yok, sadece Oodle sıkıştırması var.

## Giriş api'si

Bütün istekler `POST`, gövde JSON. Başlıklarda oyunun kendi `User-Agent` değeri (`RiseOnlineMobile/UE5-CL-0 (http-eventloop) Android/16`) ve `X-Device: Android` gönderiliyor. Tarayıcı `User-Agent`'ı ile istek atınca sunucu 403 dönüyor.

### 1. Ön giriş

```
POST /pre-login
{ "username": "<kullanıcı adı>", "secret": md5(<kullanıcı adı> + <tuz>) }
```

`secret` alanı kullanıcı adının arkasına istemciye gömülü sabit bir tuz eklenip MD5'i alınarak (küçük harf hex) hesaplanıyor. Tuz oyunun blueprint'lerinden birinde düz metin olarak duruyor. Doğru hesaplanırsa cevapta yaklaşık bir gün geçerli, kullanıcı adına ve ip adresine bağlı bir `ticket` (JWT) geliyor. Yanlışsa `Invalid verification key` dönüyor.

### 2. Giriş

```
POST /login
{
  "username": "...",
  "password": "...",
  "ticket": "<ön girişten gelen ticket>",
  "mac": "...",
  "motherboard": "...",
  "deviceInfo": {
    "deviceModel": "...", "os": "Android", "sdkVersion": "...",
    ...
    "gameversion": "1.18.14055"
  }
}
```

Şifre düz gidiyor, koruma TLS'e bırakılmış. Sunucunun sürüm kontrolü `gameversion` alanına bakıyor ve bu alan üst seviyede değil, `deviceInfo` nesnesinin içinde olmak zorunda. Üst seviyeye ya da başlıklara konan sürüm bilgisini sunucu hiç görmüyor ve hata mesajındaki parantezin içi boş kalıyor: `Lütfen oyunun en güncel sürümünü kullanınız. ()`. Doğru yere konunca parantezin içinde gönderilen sürüm görünüyor.

Cevap iki şekilde gelebiliyor:

- E-posta doğrulaması kapalıysa `redirect: "authorized"` ile birlikte `token` doğrudan oturum anahtarı oluyor.
- Açıksa `redirect: "security"` ile geçici bir `token` geliyor ve hesabın e-posta adresine bir kod gönderiliyor. Kod yaklaşık bir saat geçerli ve her girişte yenisi geliyor, sadece en son gelen kod kabul ediliyor.

### 3. E-posta doğrulaması

```
POST /security
Authorization: <geçici token>
{ "code": "<e-postadaki kod>" }
```

Cevapta oturum anahtarı (yaklaşık bir hafta geçerli bir JWT), 20 haneli hex `sessionkey`, kullanıcı numarası ve ırk bilgisi geliyor. Oturum anahtarı süresi boyunca tekrar tekrar kullanılabildiği için doğrulama haftada bir yapılsa yetiyor.

### 4. Sunucu listesi

```
POST /servers
Authorization: <oturum anahtarı>
{}
```

```
{ "type": "success", "results": [ { "ID": 51, "Name": "Aragon", "High": 3 } ] }
```

`Authorization` başlığına JWT oturum anahtarı konmalı. Ham 20 haneli `sessionkey` konursa `Invalid session key ... OA-3` dönüyor.

`High` alanı sunucu seviyesinde bir yoğunluk değeri, oyunda doğrudan gösterilmiyor. Oyundaki doluluk çubuğu kanal bazında hesaplanıyor: `TotalUser / 2000`. Yani bir kanalın kapasitesi 2000 kişi.

Aynı oturum anahtarıyla `/characters` uç noktasına `{ "serverId": 51 }` gönderilince o sunucudaki karakterlerin listesi geliyor.

## Sürüm kontrolü

Oyun her güncellendiğinde eski sürüm numarasıyla yapılan girişler reddediliyor. Protokolün geri kalanı güncellemelerde değişmedi, sadece numaranın güncellenmesi yetiyor. Güncel sürüm Google Play'deki uygulama sayfasının kaynağında `1.xx.xxxxx` biçiminde duruyor.

| Sürüm | Not |
|-------|-----|
| 1.17.12663 | Eylül 2026 |
| 1.18.14055 | 16 Ekim 2026 güncellemesi |

## Oyun sunucusu

Oyun içi trafik Unreal Engine'in `IpNetDriver` replikasyon protokolüyle UDP üzerinden gidiyor. Ağ katmanında sadece `StatelessConnectHandlerComponent` (el sıkışma) ve `OodleNetworkHandlerComponent` (sıkıştırma) var, herhangi bir AES ya da şifreleme bileşeni yok. Kontrol kanalı tamamen düz.

Unreal'ın bit yazıcısı her baytı en düşük bitten başlayarak dolduruyor. Aşağıdaki bit numaraları da bu sıraya göre.

### StatelessConnect el sıkışması

Her paketin ilk 5 biti sabit bir sihirli başlık (değeri 12), ardından gelen bit paketin el sıkışma paketi olup olmadığını belirtiyor. Bu yüzden el sıkışma paketleri `ac 01` ile başlıyor. Sihirli başlık olmadan gönderilen paketlere sunucu hiç cevap vermiyor.

El sıkışma dört paketten oluşuyor:

```
İ→S  ac 01 02 80 | istemci kimliği | sıfır çerez alanı | rastgele kuyruk        InitialPacket
S→İ  ac 01 82 80 | istemci kimliği | zaman damgası | çerez (20 bayt) | ...     Challenge
İ→S  ac 01 02 01 | istemci kimliği | zaman damgası | çerez (aynen) | ...      Response
S→İ  ac 01 82 01 | ...                                                         Ack
```

Üçüncü baytın en yüksek biti yönü (istemci `0x02`, sunucu `0x82`), dördüncü bayt paket tipini (`0x80` başlangıç ve meydan okuma, `0x01` cevap ve onay) gösteriyor. Sunucu meydan okumada 20 baytlık bir çerez üretiyor, istemci bu çerezi hiç değiştirmeden geri gönderiyor.

### Sıra numaraları

Kontrol kanalındaki başlangıç sıra numaraları çerezden türetiliyor:

```
sunucu_sırası = LE16(çerez[0:2]) & 0x3FFF
istemci_sırası = LE16(çerez[2:4]) & 0x3FFF
```

İstemcinin ilk paketi `Seq = istemci_sırası`, `AckedSeq = sunucu_sırası - 1` ile gidiyor ve her pakette bir artıyor.

### Kontrol kanalı paket başlığı

```
bit  0-4    sihirli başlık (12)
bit  5      el sıkışma paketi mi (kontrol paketlerinde 0)
bit  6-9    geçmiş kelime sayısı
bit 10-23   AckedSeq (14 bit)
bit 24-37   Seq      (14 bit)
```

Başlığın ardından bunch verisi geliyor. Mesaj tipi bunch verisinin içinde bayta hizalı duruyor:

| Değer | Mesaj |
|-------|-------|
| 0 | NMT_Hello |
| 1 | NMT_Welcome |
| 2 | NMT_Upgrade |
| 3 | NMT_Challenge |
| 4 | NMT_Netspeed |
| 5 | NMT_Login |
| 6 | NMT_Failure |

### Güvenilir bunch sırası

Bağlantının en çok vakit aldıran kısmı burasıydı. Güvenilir (reliable) her bunch bir `ChSequence` değeri taşıyor ve bu değer bağlantıya özel:

```
taban = istemci_sırası & 1023
ilk güvenilir bunch (NMT_Hello)   ChSequence = taban + 1
ikinci güvenilir bunch (NMT_Login) ChSequence = taban + 2
```

Kayıtlı bir oturumdan alınan `NMT_Hello` paketi başka bir bağlantıda olduğu gibi tekrar gönderilince sunucu bunu sıra dışı kabul edip sessizce bekletiyor ve hiçbir cevap vermiyor. `ChSequence` her bağlantıda çerezden yeniden hesaplanınca sunucu `NMT_Challenge` ile cevap veriyor. Kanalı açan `NMT_Hello` paketinde bu alan 98. bitte, kanalı açmayan sonraki bunch'larda 96. bitte duruyor.

### Giriş mesajı

`NMT_Login` mesajının içinde Unreal'ın URL seçenekleri biçiminde düz metin bir dizi var:

```
?MyUserId=<kullanıcı no>?MyCharacterId=<karakter no>?MyClanId=<klan no>
?MyAccountName=<hesap adı>?SessionKey=<20 haneli hex>?GuidHash=<cihaz guid'i>
?MyHash=?MySecondaryHash=?Name=<cihaz adı>
```

Kullanıcı numarası, karakter numarası ve `SessionKey` değerleri HTTPS girişinden geliyor. Sunucu bilgileri kabul ederse `NMT_Welcome` ile haritayı (`/Game/Environment/Dorion/L_Dorion`) ve oyun modunu (`MMOGame.MMOGame_C`) gönderiyor. Bundan sonrası standart Unreal aktör kanalları ve RPC'ler.

## Oyun içi RPC'ler

İkili dosyada 223 istemciden sunucuya (`Server*`) ve 124 sunucudan istemciye (`Client*`) RPC bulunuyor. Hareket için Unreal'ın standart `CharacterMovementComponent` RPC'leri (`ServerMove`, `ServerMovePacked`, `ServerMoveDual`) kullanılıyor. Oyuna özel olanların bir kısmı Türkçe isimli: `ServerBankaIslem`, `ServerBuyPazarKurNative` gibi. Lobi ve sunucu seçimi Unreal'ın Online Beacon sistemi üzerinden yürüyor (`BeaconInitLoginServer`, `BeaconPortalConnect`, `BeaconTeleportUser`).

## Test

```
make test
```
