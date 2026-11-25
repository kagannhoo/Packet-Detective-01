# Packet Detective 01 — Teknik analiz raporu

**Vaka:** PACKET-001 | **Tarih:** 25.11.2026 | **Ortam:** Sentetik, çevrim dışı PCAP | **Karar:** Investigate | **Önerilen öncelik:** Düşük (eğitim senaryosu; kurumsal risk bağlamı bilinmiyor)

## 1. Amaç ve kapsam

Teslim edilen ağ kaydının bütünlüğünü doğrulamak, iletişimleri sınıflandırmak, farklı HTTP isteğini incelemek, kodlanmış içeriği çözmek ve elde edilen bulguları DNS kayıtları ile ilişkilendirmek. Bu çalışma gerçek bir olay müdahalesi değildir; canlı sistem incelenmedi ve herhangi bir müdahale yapılmadı.

## 2. Delil bütünlüğü

`sha256sum packet_detective_01.pcap` ve `sha256sum -c packet_detective_01.sha256` kullanıldı. Sonuç: `packet_detective_01.pcap: OK`.

SHA-256: `f8549097a8b082135483f16e935bb7a20c7bbd5e50d68dec8c141d06baa1dead`

Bu eşleşme verilen referans hash ile dosya içeriğinin tutarlı olduğunu gösterir; dosyanın kaynağını veya tarihsel değişmezliğini tek başına kanıtlamaz.

[Kanıt 01](screenshots/01_integrity.png)

## 3. Trafik profili

36 paket, 2.806 bayt çerçeve verisi; tüm paketler Ethernet/IPv4. Taşıma katmanında TCP 30 (%83,3), UDP 6 (%16,7). Bu kayıtta HTTP, TCP üzerinde 6 paket; DNS, UDP üzerinde 6 pakettir. Katmanlar üst üste bindiğinden protokol adetleri toplanmamalıdır.

| IPv4 | İlişkili paket | Gözlenen rol |
|---|---:|---|
| 192.168.56.23 | 24 | İstemci, GET ve POST |
| 192.0.2.80 | 20 | Intranet HTTP sunucusu |
| 192.168.56.42 | 12 | İstemci, GET |
| 198.51.100.42 | 10 | Collector HTTP sunucusu |
| 192.168.56.1 | 6 | DNS sunucusu |

Her paket iki uç noktaya sayılır. 192.0.2.0/24 ve 198.51.100.0/24 dokümantasyon IP aralıklarıdır; `.test` eğitim alan adlarıdır.

[Kanıt 02](screenshots/02_protocol_hierarchy.png) · [Kanıt 03](screenshots/03_endpoints.png)

## 4. HTTP ve DNS olay zaman çizelgesi

Zamanlar yakalamanın başlangıcına göre bağıl saniyelerdir.

| Paket | Zaman (s) | Bulgular |
|---:|---:|---|
| 1–2 | 0,00–0,25 | .42 istemcisinin intranet DNS sorgusu; yanıt 192.0.2.80 |
| 6 / 8 | 1,25 / 1,75 | .42 → intranet GET /status; 200 OK |
| 13–14 | 3,00–3,25 | .23 istemcisinin intranet DNS sorgusu; yanıt 192.0.2.80 |
| 18 / 20 | 4,25 / 4,75 | .23 → intranet GET /status; 200 OK |
| 25–26 | 6,00–6,25 | .23 → DNS; collector.packetlab.test → 198.51.100.42 |
| 30 | 7,25 | .23 → collector; POST /submit HTTP/1.1 |
| 32 | 7,75 | Collector → .23; HTTP 202 Accepted, Received |

İki GET isteğinin ardından farklı bir hedefe POST isteği görülmesi, bu kısa kayıt içinde incelenecek bir farklılıktır; tek başına saldırı göstergesi değildir.

[Kanıt 04](screenshots/04_baseline_http.png) · [Kanıt 05](screenshots/05_http_requests.png) · [Kanıt 06](screenshots/06_post_filter.png)

## 5. Veri içeriği

TCP Stream 2 incelendi. Paket 30, 238 bayt çerçeve; HTTP Content-Length 62 bayt, Content-Type `text/plain`. Kaynak 192.168.56.23:41210, hedef 198.51.100.42:80. Gövde içindeki `note=` alanı:

```text
UEFDS0VULUxBQi0wMDM6IHRyYWluaW5nIGRvY3VtZW50IHRyYW5zZmVy
```

CyberChef ile From Base64 işlemi uygulandı. Çıktı:

```text
PACKET-LAB-003: training document transfer
```

Base64 şifreleme değil kodlamadır. HTTP 202 yanıtı isteğin işleme kabul edildiğini gösterir; işlemin tamamlandığını veya bir belgenin gerçekten aktarıldığını kanıtlamaz. İncelenen veri bir metin notudur; belge dosyası içeriği tespit edilmedi.

[Kanıt 07](screenshots/07_tcp_stream.png) · [Kanıt 08](screenshots/08_base64_decode.png)

## 6. DNS korelasyonu

6 DNS paketi (3 sorgu, 3 yanıt). DNS sunucusu 192.168.56.1. Paket 26'da A kaydı `collector.packetlab.test` → `198.51.100.42`, TTL 60 saniye. Bu adres paket 30'un hedefiyle eşleşmektedir. Paket 14 ise farklı bir sorguya (intranet) aittir; ilk incelemedeki karışıklık `frame.number == 26` filtresiyle giderildi.

[Kanıt 09](screenshots/09_dns_overview.png) · [Kanıt 10 — intranet karşılaştırması](screenshots/10_baseline_dns.png) · [Kanıt 11](screenshots/11_collector_dns.png)

## 7. SOC değerlendirmesi

**Karar: Investigate.** Kaynağın farklı bir hedefe HTTP üzerinden Base64 kodlu bir not göndermesi gözlenmiştir. Yetkilendirme durumu ve aktarımın iş amacı bu kısa kayıttan belirlenememektedir. POST, Base64 ve 202 yanıtı tek başına veri sızıntısı veya zararlı etkinlik kanıtı değildir.

Gerçek bir kurumda sonraki adımlar: ilgili PCAP ve kayıtları korumak, hedefin izinli olup olmadığını varlık envanteri ve iş birimiyle doğrulamak, geçmiş ağ kayıtlarında benzer trafiği araştırmak ve uç nokta/uygulama kayıtlarından aktarımı başlatan süreç ile iş amacını değerlendirmek. Yeni kanıtlar ve kurum politikası doğrultusunda öncelik yeniden belirlenmelidir. Bu adımlar öneridir, gerçekleştirilmiş müdahaleler değildir.

## 8. Öğrenme çıktıları ve sınırlamalar

- Dosya yönetimi, SHA-256 kontrolü, protokol istatistikleri, IP uç noktaları, HTTP filtreleri, TCP akış takibi, Base64 çözümü ve DNS korelasyonu uygulandı.
- Paket uzunluğu ile HTTP metodu ve farklı DNS yanıtları arasındaki ayrım uygulamayla pekiştirildi.
- Kayıt yalnızca 36 sentetik paketten oluşur; uç nokta kayıtları, kurumsal politika ve uzun dönemli trafik bulunmaz.
- Senaryo/PCAP üretimi, bazı analiz ipuçları, bulguların doğrulanması ve rapor düzenlemesinde ChatGPT desteği kullanıldı. Ekran görüntüleri öğrencinin kendi Kali ortamındaki uygulamasına aittir. Bu çalışma bağımsız uzmanlık iddiası değildir.
