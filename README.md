# Aurora Commerce — Mock API

Statik, **GitHub'dan servis edilebilen** sahte (mock) bir API. Backend hazır olmadan Jetpack Compose
storefront'unu baştan sona geliştirmek için tasarlandı. Tüm veriler ve **görseller gerçek** — ürün
görselleri gerçek bir CDN'den (`cdn.dummyjson.com`) yüklenir, yani uygulamada gerçek fotoğraflar görünür.

- **320 ürün** (190 benzersiz gerçek ürün + 130 özellik varyantı), **17 kategori**, sayfalama (16 sayfa × 20 ürün)
- **Kadın giyim/çanta/takı/ayakkabı ve makyaj/parfüm/cilt bakımı YOK** — teknoloji, ev aletleri, mobilya, elektronik, giyim (nötr), spor, market odaklı
- Görsel kaynakları: **DummyJSON CDN** (283 ürün, çok kararlı) + **Platzi/imgur** (37 ürün, gerçek ama ara sıra hız-sınırlı olabilir)
- Aurora sözleşmesine uygun uç noktalar: `products`, `cart/preview`, `checkout`, `orders/{id}`
- Zengin alanlar: fiyat (integer kuruş), indirim, stok, puan, yorumlar, kargo, garanti, varyantlar, rozetler
- **Tüm veriler Türkçe** (isim, açıklama, yorum, durum, kategori, rozet), **para birimi TL (₺)**, KDV %20
- Para birimi her yerde **integer kuruş** (`unitPrice: 46990` = ₺469,90) — Aurora sözleşmesindeki "integer cents" kuralı, TL için kuruş
- Fiyatlar gerçekçi: USD verisi ~47 kurdan TL'ye çevrilip "charm" fiyatına (₺…,90 / ₺…9) yuvarlandı
- Görsel yolları İngilizce ürün adına dayanır (CDN gereği); ürünlerde ayrıca `englishTitle` alanı vardır

---

## İki kullanım şekli

### 1) Statik dosya olarak (en kolay — sunucu gerekmez)
Bu klasörü bir GitHub reposuna at ve **raw** URL'lerini API gibi kullan:

```
https://raw.githubusercontent.com/<kullanıcı>/aurora-commerce-mock-api/main/v1/products.json
```

Retrofit/Ktor `baseUrl` olarak şunu ver:
```
https://raw.githubusercontent.com/<kullanıcı>/aurora-commerce-mock-api/main/v1/
```
ve `products.json`, `products/123.json`, `categories.json` gibi yolları çağır.

### 2) Gerçek REST sunucusu gibi (json-server)
Dinamik olarak `GET /products/1`, `POST /orders` çalışsın istersen:
```
npm install -g json-server
json-server --watch db.json --port 8080
# GET http://localhost:8080/products/1
# GET http://localhost:8080/products?category=beauty&_page=1&_limit=20
```
Emülatörden host'a erişim: `http://10.0.2.2:8080`.

---

## Uç noktalar (statik dosya karşılıkları)

| Amaç | Aurora çağrısı | Statik dosya |
|---|---|---|
| Ürün listesi (sayfa 1) | `GET /products` | `v1/products.json` |
| Ürün listesi (sayfa N) | `GET /products?page=N` | `v1/products/pages/page-NN.json` |
| Tüm ürünler (tek dosya) | — | `v1/products/all.json` |
| Ürün detayı | `GET /products/{id}` | `v1/products/{id}.json` (1–320) |
| Kategoriler | — | `v1/categories.json` |
| Kategoriye göre ürünler | `GET /products?category=` | `v1/categories/{slug}.json` |
| Arama (örnek) | `GET /search?q=saat` | `v1/search/saat.json` |
| Sepet önizleme | `POST /cart/preview` | `v1/cart/preview.example.json` (+ `.request.example.json`) |
| Ödeme (başarılı) | `POST /checkout` → 200 | `v1/checkout/success.example.json` |
| Ödeme (stok yok) | → 409 | `v1/checkout/out-of-stock.example.json` |
| Ödeme (geçersiz) | → 422 | `v1/checkout/invalid-request.example.json` |
| Sipariş detayı | `GET /orders/{id}` | `v1/orders/1001.json`, `1002`, `1003` |
| Hata gövdeleri | — | `v1/errors/{400,401,404,409,422,429,500}.json` |
| API özeti | — | `v1/index.json` |

---

## Ürün nesnesi (özet)

```jsonc
{
  "id": 1,
  "sku": "AUR-BEA-ESS-0001",
  "title": "Essence Lash Princess Maskara",
  "englishTitle": "Essence Mascara Lash Princess",   // sadece görsel yolu için
  "brand": "Essence",
  "category": "beauty",
  "categoryName": "Güzellik",
  "description": "…",
  "unitPrice": 46990,           // integer kuruş — Aurora sözleşmesi
  "currency": "TRY",
  "price": { "amount": 46990, "formatted": "₺469,90",
             "compareAt": 52491, "compareAtFormatted": "₺524,91",
             "discountPercentage": 10.48, "savings": 5501 },
  "stock": 99, "inStock": true, "availabilityStatus": "Stokta",
  "minimumOrderQuantity": 48,
  "rating": 2.56, "reviewCount": 3, "ratingBreakdown": { "5": 1, "4": 1, ... },
  "tags": ["makyaj","kozmetik","hayvan-dostu"], "badges": ["indirim"],
  "variantOf": null, "variantLabel": null,   // varyantlarda parent id + Türkçe etiket dolu gelir
  "images": ["https://cdn.dummyjson.com/product-images/beauty/essence-mascara-lash-princess/1.webp"],
  "thumbnail": "https://cdn.dummyjson.com/.../thumbnail.webp",
  "reviews": [ { "author": "Fatma Aydın", "rating": 4, "comment": "…", "verifiedPurchase": true } ],
  "shipping": { "free": false, "info": "3-5 iş günü içinde kargo", "estimatedDays": 4 },
  "warrantyInformation": "1 hafta garanti", "returnPolicy": "7 gün içinde iade"
}
```

Liste uç noktaları hafif "kart" nesneleri döner (`unitPrice`, `thumbnail`, `rating`, `badges`, …);
tam detay için `products/{id}.json` çağrılır.

---

## Compose için notlar
- **Görselleri Coil ile yükle**: `AsyncImage(model = product.thumbnail, ...)`. URL'ler `.webp` — Coil destekler.
- **Parayı kuruş (Int/Long) olarak taşı**, sadece gösterirken formatla:
  `NumberFormat.getCurrencyInstance(Locale("tr","TR")).format(kurus / 100.0)` → `₺469,90`.
- **Sayfalama**: `pagination.hasNext` / `page-NN.json` ile sonsuz kaydırma yapılabilir.
- **Durum modeli**: liste uçları `data` + `pagination` döner; hata örnekleri `errors/` altında.
- Görseller gerçek bir CDN'de; internet gerektirir (emülatör/telefon çevrimiçi olmalı).

Görsel kaynağı: gerçek ürün fotoğrafları `cdn.dummyjson.com` ve `i.imgur.com` üzerinden servis edilir.
Ürün nesnesindeki `meta.imageSource` hangi kaynaktan geldiğini belirtir; her üründe görsel yolu için
`englishTitle` alanı da vardır. `v1/index.json` tüm uç noktaların makine-okur özetidir.
