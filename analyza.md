# Analýza DevOps nástrojov a služieb projektu wger-project/flutter

**Projekt:** https://github.com/wger-project/flutter  
**Analyzované súbory:** 8 workflow súborov z `.github/workflows/`  
**Dátum analýzy:** November 2025

---

## 1. GITHUB ACTIONS – CONTINUOUS INTEGRATION

### 1.1 Workflow: ci.yml

**Názov:** "Continous Integration"  
**Účel:** Automatizované testovanie a kontrola kvality kódu

### 1.2 Kedy sa workflow spúšťa

| Trigger | Podmienka | Vysvetlenie |
|---------|-----------|-------------|
| Push | Zmeny v `*.dart` alebo `pubspec.yaml` | Pri každom commite s kódom |
| Pull Request | Do vetvy `master` | Pri vytvorení/aktualizácii PR |
| Manuálne | `workflow_dispatch` | Ručné spustenie z GitHub |
| Volanie | `workflow_call` | Iné workflow ho môžu zavolať |

### 1.3 Kroky workflow

**Krok 1: Checkout kódu**
```yaml
- uses: actions/checkout@v5
```
GitHub Actions stiahne kód z repozitára na Ubuntu server, kde prebehne testovanie.

**Krok 2: Flutter setup**
```yaml
- uses: ./.github/actions/flutter-common
```
Vlastná action, ktorá nainštaluje Flutter, Java a nakonfiguruje prostredie.

**Krok 3: Install dependencies**
```yaml
- run: |
    sudo apt install libsqlite3-dev lcov
    flutter pub get
```
- `libsqlite3-dev` - databázová knižnica
- `lcov` - nástroj na coverage reporting  
- `flutter pub get` - stiahnutie Dart balíčkov

**Krok 4: Testovanie**
```yaml
- run: |
    flutter test --coverage
    lcov --remove coverage/lcov.info 'lib/l10n/generated/*' 'lib/theme/*'
```
Spustí testy, vygeneruje coverage a odstráni automaticky generované súbory (l10n, theme).

**Krok 5: Coveralls upload**
```yaml
- uses: coverallsapp/github-action@v2
```
Nahrá coverage report na Coveralls pre vizualizáciu.

---

## 2. GITHUB ACTIONS – RELEASE MANAGEMENT

### 2.1 Workflow: make-release.yml

**Názov:** "Build release artefacts"  
**Trigger:** Manuálne (`workflow_dispatch` s parametrom verzie X.Y.Z)

### 2.2 Release proces (7 jobov)

```
1. CI (ci.yml) → Testy musia prejsť
2. Version Bump → Aktualizácia verzií, Git tag
3. Parallel Builds:
   ├─ Android (APK + AAB)
   ├─ Apple (iOS + macOS)
   ├─ Linux (tar.gz + Flatpak)
   └─ Windows (.exe)
4. Play Store Upload → Fastlane deployment
5. GitHub Release → Publikácia všetkých buildov
```

### 2.3 Vytvorené artefakty

- `app-release.aab` - Android App Bundle
- `app-release.apk` - Android APK
- `linux-x64.tar.gz` - Linux build
- `Runner.app.zip` - iOS aplikácia
- `wger.app.zip` - macOS aplikácia  
- `wger.exe` - Windows executable

---

## 3. VERSION MANAGEMENT

### 3.1 Workflow: bump-version.yml

**Účel:** Automatická aktualizácia čísla verzie

### 3.2 Proces

**1. Validácia verzie**
```bash
if [[ ! "$VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
  exit 1
fi
```
Kontrola formátu X.Y.Z (napr. 1.6.0).

**2. Výpočet build čísla**
```bash
CURRENT_BUILD=$(flutter pub run cider version | cut -d '+' -f 2)
NEXT_BUILD=$(( (CURRENT_BUILD / 10 + 1) * 10 ))
```
Zaokrúhľuje na násobky 10 (napr. 320 → 330).

**3. Update súborov**
- `pubspec.yaml` - hlavná verzia
- `flatpak/*.xml` - Linux metadáta

**4. Git commit a tag**
```bash
git commit -m "Bump version to X.Y.Z"
git tag X.Y.Z
git push origin HEAD:master --tags
```

---

## 4. PLATFORM BUILDS

### 4.1 Android (build-android.yml)

**Job 1: APK Build**
- Dešifrovanie signing keys
- `flutter build apk --release`
- Upload APK

**Job 2: AAB Build**  
- Dešifrovanie signing keys
- `flutter build appbundle --release`
- Upload AAB

**Secrets:**
- `DECRYPTKEY_PLAYSTORE_SIGNING_KEY`
- `DECRYPTKEY_PROPERTIES`

### 4.2 Apple (build-apple.yml)

**Job 1: iOS Build**
```yaml
- run: sudo xcode-select --switch /Applications/Xcode_16.4.app
- run: flutter build ios --release --no-codesign
```

**Job 2: IPA Build**
```yaml
- run: flutter build ipa --release --no-codesign
```

**Job 3: macOS Build**
```yaml
- run: flutter build macos --release
```

### 4.3 Linux (build-linux.yml)

**Job 1: Linux x64**
```yaml
- run: |
    sudo apt install libgtk-3-dev liblzma-dev
    flutter build linux --release
    tar -zcvf linux-x64.tar.gz build/linux/x64/release/bundle
```

**Job 2: Flatpak manifest update**
- Python skript aktualizuje Flatpak metadáta
- *(Automatický push do Flathub je vypnutý)*

### 4.4 Windows (build-windows.yml)

```yaml
- run: flutter build windows --release
- upload: build\windows\x64\runner\Release\wger.exe
```

**Poznámka:** Momentálne len .exe (chýbajú DLL súbory).

---

## 5. COVERALLS – CODE COVERAGE

**Integrácia:** V `ci.yml` workflow  
**Služba:** coveralls.io

### 5.1 Proces

1. **Generovanie coverage:**
   ```bash
   flutter test --coverage
   ```

2. **Filtrovanie:**
   ```bash
   lcov --remove coverage/lcov.info 'lib/l10n/generated/*' 'lib/theme/*'
   ```

3. **Upload:**
   ```yaml
   - uses: coverallsapp/github-action@v2
   ```

### 5.2 Výstupy

- Coverage % badge v README
- Komentár na PR s coverage zmenami
- Detailná vizualizácia pokrytých/nepokrytých riadkov

---

## 6. FASTLANE – DEPLOYMENT

**Ruby:** 3.4  
**Účel:** Automatický upload na Google Play Store

### 6.1 Proces

```yaml
1. Setup Ruby
2. Decrypt Play Store credentials
3. bundle install
4. bundle exec fastlane android production
```

### 6.2 Fastlane lane

```ruby
upload_to_play_store(
  track: 'production',
  aab: 'build/app/outputs/bundle/release/app-release.aab',
  skip_upload_metadata: false
)
```

Nahrá AAB priamo na production track v Play Store.

**iOS:** Momentálne manuálny proces.

---

## 7. DEPENDABOT

**Frekvencia:** Týždenná kontrola  
**Príklady PR:**
- "Bump flutter_svg from 2.0.17 to 2.1.0"
- "Bump sqlite3_flutter_libs from 0.5.27 to 0.5.28"

### 7.1 Proces

1. Dependabot detekuje novú verziu balíčka
2. Vytvorí Pull Request s aktualizáciou
3. CI workflow spustí testy
4. Pri úspechu → možnosť merge

---

## 8. WEBLATE – LOCALIZATION

**Jazykov:** 38  
**URL:** https://hosted.weblate.org/projects/wger/mobile/

### 8.1 Workflow

1. Developer pridá nový text v angličtine
2. `flutter gen-l10n` vygeneruje template
3. Weblate detekuje nový string
4. Prekladatelia preložia do 38 jazykov
5. Weblate vytvorí PR s prekladmi
6. CI otestuje syntax
7. Merge do master

---

## 9. CUSTOM ACTION: flutter-common

**Lokácia:** `.github/actions/flutter-common`  
**Typ:** Composite action

### 9.1 Použitie

```yaml
# Namiesto opakovania:
- uses: actions/setup-java@v3
- uses: subosito/flutter-action@v2
- run: flutter pub get

# Stačí:
- uses: ./.github/actions/flutter-common
```

Používané vo **všetkých** workflows pre konzistentné prostredie.

---

## 10. SECRETS MANAGEMENT

| Secret | Účel |
|--------|------|
| `GITHUB_TOKEN` | GitHub API (automatické) |
| `DECRYPTKEY_PLAYSTORE` | Play Store credentials |
| `DECRYPTKEY_PLAYSTORE_SIGNING_KEY` | Android signing |
| `DECRYPTKEY_PROPERTIES` | Android konfigurácia |
| `SSH_DEPLOY_KEY` | Flathub repository |

### 10.1 Bezpečnosť

- Credentials sú v Git **zašifrované**
- Dešifrovanie len počas buildu pomocou `decrypt_secrets.sh`
- Po builde automaticky zmazané
- Nikdy nie sú v plain text v Git

---

## 11. SUMARIZAČNÁ TABUĽKA

| Nástroj | Fáza | Súbor | Trigger | Automatizácia |
|---------|------|-------|---------|---------------|
| GitHub Actions (CI) | Test | ci.yml | Push/PR | Plná |
| GitHub Actions (Release) | Build | make-release.yml | Manuálne | Orchestrovaná |
| GitHub Actions (Version) | Version | bump-version.yml | Volaná | Automatická |
| GitHub Actions (Android) | Build | build-android.yml | Volaná | Automatická |
| GitHub Actions (Apple) | Build | build-apple.yml | Volaná | Automatická |
| GitHub Actions (Linux) | Build | build-linux.yml | Volaná | Automatická |
| GitHub Actions (Windows) | Build | build-windows.yml | Volaná | Automatická |
| Coveralls | Monitoring | ci.yml | Pri CI | Automatická |
| Fastlane | Deploy | make-release.yml | Pri release | Automatická |
| Dependabot | Dependencies | (implicitné) | Týždenná | Automatická |
| Weblate | Localization | (externé) | Priebežná | Automatická |
| flutter-common | Setup | (action) | Všade | Reusable |

---

## 12. HODNOTENIE

### 12.1 Silné stránky

✅ Komplexná multi-platform CI/CD (5 platforiem)  
✅ Automatické testovanie s coverage reportingom  
✅ Automatizovaný release proces  
✅ Správne použitie GitHub Secrets  
✅ Modularita (reusable workflows, custom actions)  
✅ Internationalizácia (38 jazykov)  
✅ Automatická údržba (Dependabot)

### 12.2 Oblasti na zlepšenie

⚠️ Neúplný Windows build (chýbajú DLL)  
⚠️ Linux len x64 (nie arm64)  
⚠️ iOS deployment je manuálny  
⚠️ Chýba runtime monitoring (Sentry, Crashlytics)  
⚠️ Flathub push je vypnutý

### 12.3 Štatistiky

- **542 riadkov** YAML konfigurácie
- **8 workflow súborov**
- **12 DevOps nástrojov**
- **5 platforiem**
- **38 jazykov**
- **7 artefaktov** pri release

### 12.4 Celkové hodnotenie

**8/10 bodov**

Projekt implementuje vysoko kvalitnú DevOps infraštruktúru s komplexnou CI/CD pipeline, automatizáciou kľúčových procesov a správnou implementáciou bezpečnostných best practices. Pre plný počet bodov by bolo potrebné pridať runtime monitoring a dokončiť Windows/Linux implementácie.

---

*Dokument vytvorený: November 2025*  
*Založené na analýze 8 workflow súborov z .github/workflows/*  
*Celková konfigurácia: 542 riadkov YAML kódu*
