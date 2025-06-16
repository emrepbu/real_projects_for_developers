# iOS CI/CD Pipeline Tutorial: GitHub Secrets ile Güvenli Key Yönetimi

Bu dokümanda, iOS uygulamalarında hassas bilgileri (API keys, secrets vb.) güvenli bir şekilde yönetmek için GitHub Actions ve GitHub Secrets kullanarak CI/CD pipeline kurulumunu adım adım anlatmaktadır.

> Bu dokümandaki KEY ve VALUE değerlerini projenize uygun olarak seçin ve güvenli bir şekilde oluşturulduğundan emin olun.

## Adım Adım Kurulum

### 1. SwiftUI Projesi Oluşturma

```bash
# Xcode'da yeni proje oluşturun
- Product Name: SecretDemo
- Interface: SwiftUI
- Language: Swift
```

### 2. ContentView.swift Oluşturma

Burada test amaçlı olarak bir view oluşturuldu.

```swift
import SwiftUI

struct ContentView: View {
    @State private var apiKey = ""
    @State private var showingSecret = false
    
    var body: some View {
        VStack(spacing: 30) {
            Text("GitHub Secrets Demo")
                .font(.largeTitle)
                .fontWeight(.bold)
            if showingSecret {
                VStack {
                    Text("API Key:")
                        .font(.headline)
                    Text(apiKey.isEmpty ? "Yükleniyor..." : apiKey)
                        .font(.system(.body, design: .monospaced))
                        .padding()
                        .background(Color.yellow.opacity(0.2))
                        .cornerRadius(8)
                }
            }
            
            Spacer()
            
            Button(action: {
                showingSecret.toggle()
            }) {
                Text(showingSecret ? "API Key'i Gizle" : "API Key'i Göster")
                    .padding()
                    .background(Color.blue)
                    .foregroundColor(.white)
                    .cornerRadius(10)
            }
        }
        .padding()
        .onAppear {
            loadAPIKey()
        }
    }
    
    func loadAPIKey() {
        // Build time'da inject edilen değeri okuyoruz
        if let key = Bundle.main.infoDictionary?["API_KEY"] as? String {
            apiKey = key
        } else {
            apiKey = "API Key bulunamadı!"
        }
    }
}

#Preview {
    ContentView()
}
```

### 3. Info.plist Konfigürasyonu

Info.plist dosyanıza şu satırı ekleyin:

```xml
<key>API_KEY</key>
<string>$(API_KEY)</string>
```

<img width="1016" alt="image" src="https://github.com/user-attachments/assets/9bfb323e-ef6f-4769-93fb-6eb1d049ba72" />

### 4. Configuration File Oluşturma

Proje root dizininde `Config.xcconfig` dosyasını aşağıdaki komut ile oluşturun:

```bash
touch Config.xcconfig
```

Ardından dosya içeriğini aşağıdaki gibi güncelleyin:

```xcconfig
// Local test için
API_KEY = TopSecretKeyInDevelopmentEnvironment
```

> Production için gerekli **xcconfig** dosyası GitHub Actions ile oluşturulmaktadır.

### 5. Xcode'da Configuration Dosyasını Bağlama

1. Xcode'da projenizi açın
2. Sol panelde proje adına (mavi ikon) tıklayın
3. PROJECT → SecretDemo seçin
4. Info sekmesine gidin
5. Configurations bölümünde Debug ve Release için Config dosyasını seçin

<img width="1363" alt="image" src="https://github.com/user-attachments/assets/08819462-169d-43f4-ad0c-f944b7d0bf81" />

Bu aşamaya kadar geldiyseniz projeyi çalıştırdığınızda aşağıdaki gibi bir ekranla karşılaşacaksınız.

<img width="250" alt="image" src="https://github.com/user-attachments/assets/90802e23-7766-4593-b2da-8d4fcb3a5f9a" />

### 6. .gitignore Dosyası

```gitignore
# Xcode
*.xcodeproj/xcuserdata/
*.xcworkspace/xcuserdata/
*.xcuserstate

# Config files with secrets
# !!! PRODUCTION ORTAMINDA .gitignore DOSYANIZA '*.xcconfig' DEĞERİNİ EKLEMEYİ UNUTMAYIN !!!
# *.xcconfig

# Build
build/
DerivedData/
```

### 7. GitHub Repository Oluşturma

```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/YOUR_USERNAME/SecretDemo.git
git push -u origin main
```

### 8. GitHub Secrets Ekleme

1. GitHub repository sayfasında: **Settings** → **Secrets and variables** → **Actions**
2. **"New repository secret"** butonuna tıklayın
3. Name: `API_KEY_VALUE`
4. Value: `TopSecretKeyInProductionEnvironment`
5. **"Add secret"** butonuna tıklayın

<img width="1000" alt="image" src="https://github.com/user-attachments/assets/d2aadae8-02f7-428b-a571-be48aa93a855" />
<img width="600" alt="image" src="https://github.com/user-attachments/assets/55e9a57b-4c99-4a55-85fd-87c93320a11a" />
<img width="600" alt="image" src="https://github.com/user-attachments/assets/e1e448f1-ee96-454f-b40a-1edc1a5df559" />

### 9. GitHub Actions Workflow Oluşturma

`.github/workflows/build.yml` dosyası oluşturun:

```yaml
name: iOS Build and Release

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: macos-latest
    permissions:
      contents: write
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set up Xcode
      uses: maxim-lobanov/setup-xcode@v1
      with:
        xcode-version: latest-stable
    
    - name: Create Config file with Secret
      run: |
        echo "API_KEY = ${{ secrets.API_KEY_VALUE }}" > Config.xcconfig
        
    - name: Build iOS App
      run: |
        xcodebuild -project SecretDemo.xcodeproj \
                   -scheme SecretDemo \
                   -sdk iphonesimulator \
                   -configuration Debug \
                   -derivedDataPath ./build \
                   -xcconfig Config.xcconfig \
                   build
    
    - name: Create IPA
      run: |
        cd build/Build/Products/Debug-iphonesimulator
        mkdir Payload
        cp -R SecretDemo.app Payload/
        zip -r ../../../SecretDemo-Simulator.ipa Payload
        cd -
    
    - name: Create Release
      uses: softprops/action-gh-release@v1
      with:
        tag_name: build-${{ github.run_number }}
        name: "Build #${{ github.run_number }}"
        files: |
          build/SecretDemo-Simulator.ipa
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Nasıl Çalışır?

### Variable Substitution Süreci

1. **Local Development**: 
   - `Config.xcconfig` dosyasında test değerleri tanımlanır
   - Xcode build sırasında `$(API_KEY_VALUE)` değişkeni çözümlenir
   - Info.plist'teki `$(API_KEY)` değeri doldurulur

2. **CI/CD Pipeline**:
   - GitHub Actions workflow tetiklenir
   - GitHub Secrets'tan `API_KEY_VALUE` alınır
   - Config.xcconfig dosyası dinamik olarak oluşturulur
   - Xcode build sırasında gerçek değer Info.plist'e yazılır

### Güvenlik Akışı

```
GitHub Secrets (Encrypted)
    ↓
GitHub Actions (Runtime)
    ↓
Config.xcconfig (Temporary)
    ↓
Xcode Build Process
    ↓
Info.plist (Compiled App)
```

## Build'i İndirme ve Test Etme

### GitHub Releases'den İndirme

1. Repository sayfasında sağ tarafta **"Releases"** bölümüne tıklayın
2. En son release'i bulun
3. Assets bölümünden `SecretDemo-Simulator.ipa` dosyasını indirin

<img width="363" alt="image" src="https://github.com/user-attachments/assets/d4718c8b-1449-47cd-87e8-c78da2897cdb" />
<img width="1245" alt="image" src="https://github.com/user-attachments/assets/619e402c-a15e-4fd0-a087-c42ee45bcc1c" />

### Simulator'a Yükleme

```bash
# ZIP'i açın
unzip SecretDemo-Simulator.ipa

# .app dosyasını Simulator'a yükleyin
xcrun simctl install booted Payload/SecretDemo.app

# Veya Finder'dan sürükle-bırak yapın
```

.app dosyasını simulatöre kurduğunuzda aşağıdaki gibi bir ekranla karşılaşacaksınız. Burada 8. adımda eklediğimiz production ortamına ait gizli veri artık güvenli bir şekilde projemize geliyor.

<img width="250" alt="image" src="https://github.com/user-attachments/assets/c4cbcd90-cb13-48ae-8265-7d32c2611630" />

## Güvenlik Uyarıları ve Önemli Notlar

### Kritik Güvenlik Uyarıları

#### 1. Config Dosyalarını ASLA Commit Etmeyin

```bash
# YANLIŞ - Config dosyası Git'e eklenmemeli
git add Config.xcconfig  

# DOĞRU - .gitignore'a ekleyin
echo "*.xcconfig" >> .gitignore
```

**Neden?** Config dosyaları gerçek API key'leri içerebilir. Bir kez commit edilirse Git geçmişinde kalır!

#### 2. GitHub Secrets'a Erişim Kontrolü

- Repository Settings → Manage access kısmından kimlerin erişimi olduğunu kontrol edin
- Sadece güvendiğiniz kişilere admin/write yetkisi verin
- Secrets'ları düzenli olarak rotate edin (değiştirin)

#### 3. Public Repository Riski

```yaml
# Public repo'da bu tehlikeli olabilir:
- name: Debug Config
  run: |
    cat Config.xcconfig  # Secret'ı loglara yazdırır
```

**Not:** GitHub Actions loglarında secrets otomatik maskelenir ama yine de dikkatli olun!

### Production Ortamı İçin Uyarılar

#### 1. Bu Yöntem Demo/Development İçindir

Production uygulamalar için:
- iOS Keychain kullanın
- Certificate pinning uygulayın  
- Encrypted configuration dosyaları kullanın
- Runtime'da API endpoint'ten key alın

#### 2. Secret Rotation Politikası

- API key'leri düzenli değiştirin (3-6 ayda bir)
- Eski key'leri kullanımdan kaldırın
- Access log'larını kontrol edin

### Güvenlik Kontrol Listesi

Deployment öncesi kontrol edin:

- [ ] Config dosyaları .gitignore'da mı?
- [ ] Secrets düzgün adlandırılmış mı?
- [ ] Repository private mı? (hassas projeler için)
- [ ] Gereksiz log/debug kodu temizlendi mi?
- [ ] Team üyeleri güvenlik politikasını biliyor mu?
- [ ] Backup secret recovery planı var mı?
- [ ] API rate limit kontrolü yapıldı mı?

## Referanslar

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Xcode Build Configuration Files](https://developer.apple.com/documentation/xcode/build-settings-reference)
- [iOS Code Signing in CI/CD](https://docs.github.com/en/actions/deployment/deploying-xcode-applications/installing-an-apple-certificate-on-macos-runners-for-xcode-development)

---

**Not**: Bu tutorial eğitim amaçlıdır. Production ortamında daha gelişmiş güvenlik önlemleri (KeyChain, encrypted storage, certificate pinning vb.) kullanmanız önerilir.

**Hatırlatma**: Güvenlik bir kerelik iş değil, sürekli bir süreçtir. Düzenli olarak güvenlik pratiklerinizi gözden geçirin ve güncelleyin.
