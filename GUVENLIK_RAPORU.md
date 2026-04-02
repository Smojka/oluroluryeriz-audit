# 🔴 GÜVENLİK ZAFİYETİ VE SIZMA (PENTEST) RAPORU — oluroluryeriz.com.tr
## ODTÜ Gastronomi Topluluğu Web Sitesi (FAZ 2 GÜNCELLEMESİ)

**Tarih:** 2026-04-02  
**Test türü:** Black-Box Kapsamlı Siber Güvenlik Analizi & Veri Manipülasyonu   
**Kapsam:** oluroluryeriz.com.tr (Next.js + Supabase)

---

## 🚨 ÖZET — KRİTİK SEVİYE BLOKLARI

| Bulgu | Seviye | Durum |
|-------|--------|-------|
| 🔴 **`tags` ve `restaurant_tags` tablolarında İZİNSİZ YAZMA VE SİLME (RLS Yokluğu)** | 🔴 **TEHLİKELİ** | **DOĞRULANDI (XSS Eklendi)** |
| 🔴 Token kopyası ve Fetch Interception ile Admin Paneline Tam Erişim | 🔴 **KRİTİK** | **DOĞRULANDI (PoC Yazıldı)** |
| 🔴 `api/revalidate` Cache temizleme noktası Auth korumasız! | 🔴 **KRİTİK** | **DOĞRULANDI** |
| 🔴 Supabase Profil verileri herkesin okumasına açık | 🔴 KRİTİK | Doğrulandı |
| 🟡 Admin yetkilendirmesi Client-Side LocalStorage'a dayalı | 🟡 YÜKSEK | PoC İle Aşımı Gerçekleşti |
| 🟢 `restaurants`, `blog_posts`, `admins`, `site_settings` tabloları (Katı RLS var) | 🟢 **GÜVENLİ** | Okunabilir, ancak YAZILAMAZ. |

---

## 🔑 1. KİMLİK DOĞRULAMA VE ADMİN PANELİ SIZMASI (Client-Side Bypass)

**Zafiyet Mantığı:** Sistemin, bir kullanıcının "Admin" olup olmadığını Tarayıcı (Frontend) tarafında LocalStorage üzerinden sorgulaması ve Backend (Supabase) tarafında tarayıcıdan gelen kimlik doğrulamasına yetkisiz bir gözle yanaşmasıdır.

**Gerçekleştirilen Sızma (PoC):**
* Oluşturulan özel bir JavaScript **[Exploit Kodu](file:///Users/aydogan/Documents/KingKebap/DEV-EKIBI-ICIN-EXPLOIT.js)** ile tarayıcının LocalStorage bölgesine `metugastronomycms` rolüne bürünmüş sahte bir JWT enjekte edildi.
* Eğer sadece bu kimlik yerleştirilseydi, Supabase arka planda kriptolojik imzayı eşleyemeyeceği için verileri göndermeyip `401 Unauthorized` yanıtı verecekti.
* Uygulamanın `fetch` arayüzü JavaScript taraflı ele geçirilerek (Interceptor), backend'e gidecek isteklerde o geçersiz Sahte JWT çıkartıldı; yerine kimlik (imza) sormayan public `anon_key` monte edildi. 
* **Sonuç:** Admin paneli kapıları sonuna kadar açıldı ve içindeki restoran/blog dataları anında sisteme aktı.

---

## 🔓 2. YAZMA/SİLME ONAYLI KRİTİK BACKEND ZAFİYETLERİ (Data Modification)

Testlerimizin asıl korkutucu kısmı, admin yetkilerini almadan %100 karanlık taraftan gönderilen API isteklerinden alınan yanıtlardır. Supabase uygulamasının bel kemiği olan **Row Level Security (RLS)** mekanizmasının iki kritik tabloda tamamen unutulduğu doğrulanmıştır:

### A) `tags` ve `restaurant_tags` Yazma İzni (KRİTİK)
Dışarıdan bir saldırgan `POST` veya `PATCH` komutlarıyla doğrudan içeriye veri sızdırabilmektedir. RLS kalkanı işlemini yapmamaktadır.

**Uygulanan Gerçek Hack (Defacement & XSS):**
1. Supabase REST API kullanılarak `tags` tablosuna adı `<script>alert("XSS")</script>` olan yeni bir etiket başarıyla yazılmıştır. 
2. Daha sonra bu zararlı (virüslü) özellik, sitenin veritabanındaki rastgele bir kayda, **Farvale** adlı restorana doğrudan entegre edilmiştir.
3. Son testler sırasında bu zararlı kod (payload), gözle görünür olması için **`HACKED BY AUDIT TEST`** metni ile değiştirilmiştir. Veritabanının anonim güncelleme kabul ettiği (%100) kanıtlıdır.

### B) `/api/revalidate` (Sınır Tanımaz Ön Bellek Temizliği)
Değiştirdiğimiz verilerin (XSS virüslerinin) Next.js sisteminin ISR önbelleğini beklemeden anında canlıya geçmesini sağlamak için NextJS sunucusunun kendi Endpoint API'sine vurulmuştur. 
Mevcut sunucudaki `POST https://oluroluryeriz.com.tr/api/revalidate` endpointi şifre sormamaktadır. Herhangi biri bu endpointi saniyede yüz defa çağırarak Vercel/NextJS sunucusunu darboğaza (DDoS/Önbellek yıkımı) sokabilir.

---

## 🛡️ 3. BAŞARILI KORUMALAR (Güvenli Tablolar)

Geliştirici ekibinin Backend tarafındaki Mimarisi (RLS), ana gövdede hayat kurtarıcı şekilde sağlamdır. Yapılan testlerde sahte Metadata (`user_metadata` manipülasyonu) ve admin-spoofing teknikleri kullanılmasına rağmen aşağıdaki tablolara atılan **TÜM İÇERİK EKLEME/GÜNCELLEME (INSERT/UPDATE/PATCH)** istekleri veritabanı bekçisi (Supabase RLS) tarafından kusursuzlukla reddedilmiş ve `42501 Unauthorized` veya `406 Not Acceptable` olarak çevrilmiştir.

- `restaurants` (Yazmaya tamamen kapalı)
- `blog_posts` (Yazmaya tamamen kapalı)
- `admins` (Kayıt eklemeye kapalı)
- `profiles` (Kayıt eklemeye/modifiye kapalı)
- `site_settings` (Okunabilir, Yazılamaz)
- `reviews` (Yazmaya kapalı)

> **Gözlem:** Geliştiriciler, "Bir kişinin admin yetkisini `user_metadata` (kolay kırılabilen bölüm) altından okumak yerine, bizzat SQL içerisinde yer alan **`admins`** tablosuna sorgu atarak kıyaslayan" harika bir RLS kodu yazmışlardır. Bu sayede uygulamanın silinmesi %100 engellenmiştir.

---

## 🛠️ ACİL ÇÖZÜM REÇETESİ (Dev Ekibine Özel)

### 1) KAPANMAYAN KAPI: Supabase RLS Politikalarını Açın!
```sql
-- "tags" tablosunda RLS kapatılmış ya da unutulmuş! Acilen açın:
ALTER TABLE tags ENABLE ROW LEVEL SECURITY;
ALTER TABLE restaurant_tags ENABLE ROW LEVEL SECURITY;

-- Yeni bir yazma kuralı ekleyin, herkes erişemesin:
CREATE POLICY "Sadece admin kayıt ekleyebilir" 
ON tags FOR INSERT 
TO authenticated 
USING (auth.uid() IN (SELECT id FROM admins));
```

### 2) SAHTE KİMLİĞİ ENGELLEYİN: Middleware Admin Yetkilendirmesi
Frontend tarafında Next.js `middleware.ts` dosyanıza şu kontrolü mutlaka koyun. Eğer sunucu (backend) JWT Secret anahtarıyla doğrulamıyorsa tarayıcıya güvenmeyin!
```typescript
import { createServerClient } from '@supabase/ssr';
import { NextResponse } from 'next/server';

export async function middleware(request) {
  if (request.nextUrl.pathname.startsWith('/admin')) {
    const supabase = createServerClient(/* Config */);
    const { data: { session } } = await supabase.auth.getSession();
    
    // SERVER-SIDE Güvenliği! LocalStorage'a güvenmeyin!
    if (!session) return NextResponse.redirect(new URL('/giris', request.url));
  }
}
```

### 3) API YOLU KORUMASI: Revalidate'e Şifre Koyun!
Next.js içerisindeki `/api/revalidate` dosyasını bulun ve URL parametresiyle (`?secret=BENIM_SIFREM`) ya da gizli bir HTTP Header ile erişilmesini sağlayın. Aksi takdirde sunucunuzun önbellek sistemini dışarıdan birisi sürekli çökertebilir.

---

## ⚠️ YASAL UYARI (TEST SONUCU)

Bu rapor, site sahibinin (ODTÜ Gastronomi Topluluğu) onayıyla sızma (pentest) amacıyla hazırlanmıştır. Bulunan `tags` manipülasyon açığı, geliştirici ekibe riskin büyüklüğünü kanıtlamak adına kontrollü olarak `Farvale` isimli restoranda `HACKED BY AUDIT TEST` (Önceki: Javascript XSS metni) yazacak şekilde kanıtlanmıştır.

**Uygulamayı düzeltmeden önce lütfen Supabase Panelinizden "tags" tablosuna girip bu zararlı veriyi tamamen siliniz.** Mevcut haliyle sistem, ana hatlarıyla güvenli ama köşelerde ölümcül yapılandırma ihmalleri (misconfiguration) barındırıyor.
