# Localized index labels

> Reproduced from upstream `src/okf/index-labels.ts` (v0.3.3). Step 4 reads this file only when the wiki's effective language is not English — for `en`, the labels are `Files` / `Directories` / `Reference` and this table is not needed. Resolution (upstream `resolveIndexLabels` / `resolveConceptTypeLabel`): full tag → primary subtag → English fallback, so `pt-BR` uses the `pt` row while `pt-PT` has its own row; an unlisted language keeps English. The `Files` / `Directories` headings are curated structural chrome, not translated prose — use the table verbatim, never your own translation. The Derived type column is the localized `type` that Step 2's normalization (re-applied in Step 4) stamps on repaired pages; a language upstream leaves out of the map falls back to English `Reference`.

| Language | Files | Directories | Derived type |
|---|---|---|---|
| en | Files | Directories | Reference |
| ar | ملفات | مجلدات | مرجع |
| bg | Файлове | Директории | Reference |
| ca | Fitxers | Directoris | Referència |
| cs | Soubory | Adresáře | Reference |
| da | Filer | Mapper | Reference |
| de | Dateien | Verzeichnisse | Referenz |
| el | Αρχεία | Κατάλογοι | Αναφορά |
| es | Archivos | Directorios | Referencia |
| fi | Tiedostot | Hakemistot | Reference |
| fr | Fichiers | Répertoires | Référence |
| he | קבצים | תיקיות | Reference |
| hi | फ़ाइलें | निर्देशिकाएँ | संदर्भ |
| hr | Datoteke | Direktoriji | Referenca |
| hu | Fájlok | Könyvtárak | Reference |
| id | Berkas | Direktori | Referensi |
| it | File | Cartelle | Riferimento |
| ja | ファイル | ディレクトリ | リファレンス |
| ko | 파일 | 디렉터리 | 참조 |
| ms | Fail | Direktori | Rujukan |
| nb | Filer | Mapper | Referanse |
| nl | Bestanden | Mappen | Referentie |
| no | Filer | Mapper | Referanse |
| pl | Pliki | Katalogi | Reference |
| pt | Arquivos | Diretórios | Referência |
| pt-PT | Ficheiros | Diretórios | Referência |
| ro | Fișiere | Directoare | Referință |
| ru | Файлы | Каталоги | Справочник |
| sk | Súbory | Adresáre | Referencia |
| sl | Datoteke | Mape | Reference |
| sr | Датотеке | Директоријуми | Референца |
| sv | Filer | Kataloger | Referens |
| th | ไฟล์ | ไดเรกทอรี | อ้างอิง |
| tr | Dosyalar | Dizinler | Referans |
| uk | Файли | Каталоги | Довідник |
| vi | Tập tin | Thư mục | Tham khảo |
| zh | 文件 | 目录 | 参考 |
| zh-TW | 檔案 | 目錄 | 參考 |
