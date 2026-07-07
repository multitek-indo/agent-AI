# MULTITEK AI ASSISTANT

## Project Summary, Architecture, Deployment Baseline, dan Development Handover

**Project:** agent AI
**Produk:** Multitek AI Assistant
**Perusahaan:** Multitek Indonesia
**Status Saat Ini:** Beta Production / Internal Use
**Versi Development:** V1.4 Beta Production
**Tanggal Baseline:** 7 Juli 2026

---

# 1. TUJUAN PROJECT

Project Multitek AI Assistant dibangun untuk membuat sistem AI Assistant milik Multitek Indonesia yang berjalan pada server sendiri dan dapat dikembangkan secara bertahap.

Sistem dirancang bukan hanya sebagai chatbot umum, tetapi sebagai multi-agent AI Assistant yang dapat menangani beberapa fungsi berbeda.

Fungsi utama yang dikembangkan:

1. Customer Service Assistant.
2. Technical PCB Assistant.
3. Personal Assistant.
4. Document Agent.
5. Personal Memory.
6. Knowledge Base berbasis dokumen.
7. Integrasi Telegram.
8. Retrieval dokumen menggunakan embedding.
9. Jawaban teknis berdasarkan manual.
10. Deterministic fault extraction untuk mengurangi halusinasi LLM.

Tujuan jangka panjang adalah membuat AI Assistant internal Multitek Indonesia yang dapat membantu:

* customer service;
* teknisi elektronik industri;
* pencarian manual;
* troubleshooting inverter, servo drive, dan perangkat industri;
* penyimpanan pengetahuan perusahaan;
* personal assistant;
* pembuatan dokumen;
* otomatisasi pekerjaan perusahaan.

---

# 2. INFRASTRUKTUR SERVER

Backend berjalan pada VPS/VM Ubuntu.

Akses administrasi server dilakukan menggunakan SSH melalui Tailscale.

Direktori utama project:

/opt/multitek-ai/backend

Python virtual environment:

/opt/multitek-ai/backend/.venv

User yang pernah digunakan untuk SSH:

wordpress

Beberapa file backend dimiliki root sehingga perubahan source kadang memerlukan sudo atau akses root.

Service utama:

multitek-ai-backend.service

multitek-ai-telegram.service

ollama.service

Backend FastAPI berjalan pada:

127.0.0.1:8000

Ollama berjalan pada:

127.0.0.1:11434

Telegram Bot berjalan sebagai service systemd terpisah.

---

# 3. STACK TEKNOLOGI

Komponen utama:

* Python.
* FastAPI.
* Uvicorn.
* PostgreSQL.
* SQLAlchemy.
* Telegram Bot.
* Ollama.
* Qwen sebagai LLM lokal.
* nomic-embed-text sebagai embedding model.
* pypdf untuk ekstraksi dokumen PDF.
* systemd untuk menjalankan backend dan Telegram Bot.
* Tailscale untuk akses SSH private.

LLM yang pernah digunakan:

qwen3:0.6b

Embedding model:

nomic-embed-text

---

# 4. STRUKTUR BACKEND AWAL

Backend menggunakan struktur flat.

File utama yang pernah digunakan:

database.py

document_processor.py

knowledge_context.py

main.py

models.py

ollama_client.py

schemas.py

service_context.py

session_manager.py

telegram_bot.py

Kemudian ditambahkan beberapa modul baru:

agent_router.py

technical_pcb_agent.py

generic_fault_extractor.py

knowledge_indexer.py

knowledge_search.py

serta berbagai file regression test.

---

# 5. ARSITEKTUR MULTI-AGENT

Alur utama request:

USER / TELEGRAM

↓

FASTAPI /ai/chat

↓

AGENT ROUTER

↓

AGENT YANG SESUAI

↓

KNOWLEDGE RETRIEVAL / PERSONAL MEMORY / DOCUMENT PROCESSING

↓

DETERMINISTIC EXTRACTOR ATAU LLM

↓

ANSWER

↓

TELEGRAM / API RESPONSE

Agent utama:

## Customer Service Agent

Digunakan untuk pertanyaan mengenai:

* layanan Multitek Indonesia;
* service PCB;
* preventive maintenance;
* informasi perusahaan.

Knowledge domain:

customer_service

## Technical PCB Agent

Digunakan untuk:

* troubleshooting inverter;
* servo drive;
* PCB industri;
* fault code;
* over current;
* overload;
* over voltage;
* ground fault;
* pencarian manual;
* analisis masalah teknis.

Knowledge domain:

technical_pcb

## Personal Assistant

Digunakan untuk:

* pertanyaan umum;
* catatan;
* reminder;
* perintah personal;
* memory command.

## Document Agent

Digunakan untuk:

* quotation;
* invoice;
* dokumen perusahaan.

---

# 6. DATABASE KNOWLEDGE

Tabel utama:

knowledge_documents

Field penting:

* id;
* filename;
* file_path;
* content;
* uploaded_by;
* knowledge_domain;
* document_type;
* brand;
* model_name;
* is_active;
* created_at.

Tabel chunk:

knowledge_chunks

Field penting:

* id;
* document_id;
* chunk_index;
* content;
* source_page;
* embedding;
* created_at.

Metadata brand dan model_name digunakan untuk membantu pemisahan manual teknis.

Knowledge domain digunakan untuk mencegah kebocoran dokumen antara customer service dan technical PCB.

---

# 7. DOCUMENT PIPELINE

Alur upload manual:

TELEGRAM ADMIN

↓

UPLOAD PDF

↓

DOCUMENT PROCESSOR

↓

EXTRACT TEXT PER PAGE

↓

CREATE KNOWLEDGE DOCUMENT

↓

CREATE CHUNKS

↓

GENERATE EMBEDDING

↓

SAVE KNOWLEDGE_CHUNKS

↓

TECHNICAL KNOWLEDGE AVAILABLE

PDF diproses menggunakan pypdf.

Embedding dibuat menggunakan Ollama model:

nomic-embed-text

Manual teknis disimpan dengan metadata:

knowledge_domain = technical_pcb

document_type = service_manual

brand

model_name

source_page

---

# 8. KNOWLEDGE SEARCH

Knowledge search dikembangkan agar mendukung pemisahan domain.

Default search digunakan untuk customer service.

Technical search menggunakan:

knowledge_domain = technical_pcb

Regression test sudah memastikan:

* technical document tidak bocor ke customer search;
* customer document tidak bocor ke technical search;
* source_page tersimpan;
* metadata technical result tersedia.

---

# 9. TECHNICAL RETRIEVAL V1.2

Technical PCB Agent menggunakan multi-query retrieval.

Fungsi utama:

get_technical_evidence_v12()

Tahapan:

QUERY

↓

QUERY EXPANSION

↓

MULTIPLE KNOWLEDGE SEARCH

↓

DEDUPLICATION BY CHUNK_ID

↓

RECIPROCAL RANK FUSION

↓

SECTION BOOST

↓

BEST SCORE FILTER

↓

SELECTED EVIDENCE

Parameter penting yang pernah digunakan:

TECHNICAL_RETRIEVAL_MIN_SCORE = 0.45

TECHNICAL_EVIDENCE_MIN_SCORE = 0.45

TECHNICAL_RELATIVE_THRESHOLD = 0.80

Retrieval kemudian dikembangkan menggunakan neighbor expansion.

Tujuannya adalah mengambil chunk sebelum dan sesudah semantic anchor agar section manual yang terpotong antar-chunk dapat direkonstruksi.

Implementasi terakhir menggunakan:

PER-ANCHOR NEIGHBOR WINDOW

Setiap semantic anchor mendapat window ±2 chunk.

Semua hasil kemudian:

* digabung;
* dideduplicate berdasarkan chunk ID;
* diurutkan berdasarkan document_id dan chunk_index.

Pendekatan ini menggantikan desain sebelumnya yang membuat satu window besar dari minimum anchor sampai maksimum anchor.

---

# 10. AGENT ROUTER

Agent Router menentukan agent berdasarkan isi query.

Masalah penting yang ditemukan:

Query Telegram nyata seperti:

Apa tindakan untuk overload pada Delta VFD-EL?

Bagaimana mengatasi ground fault pada Delta VFD-EL?

Apa tindakan untuk over current pada Delta VFD-EL?

Bagaimana mengatasi over current saat constant speed pada Delta VFD-EL?

awalnya tidak masuk Technical PCB Agent.

Query jatuh ke:

personal_assistant

Akibatnya LLM menghasilkan jawaban yang berpotensi berhalusinasi.

Router kemudian diperbaiki secara konservatif.

Regression test ditambahkan menggunakan exact live Telegram queries.

Hasil regression terakhir:

TOTAL: 18

PASSED: 18

FAILED: 0

Router sekarang berhasil mengarahkan live technical queries ke:

technical_pcb

---

# 11. TECHNICAL PCB AGENT

Fungsi utama:

answer_as_technical_pcb()

Alur production saat ini:

QUERY

↓

GET TECHNICAL EVIDENCE V1.2

↓

EVIDENCE GUARD

↓

NEIGHBOR EXPANSION

↓

STRUCTURED FAULT EXTRACTION

↓

GENERIC FAULT EXTRACTION

↓

DETERMINISTIC ANSWER

Jika structured evidence tidak ditemukan:

↓

LLM FALLBACK

↓

CONTEXT RANK #1

↓

OLLAMA

↓

SOURCE SUMMARY DETERMINISTIK

Tujuan desain ini adalah memprioritaskan jawaban deterministik dari manual sebelum menggunakan LLM.

---

# 12. STRUCTURED FAULT EXTRACTION

Awalnya Technical PCB Agent hanya mengenali:

* over-current during acceleration;
* over-current during deceleration;
* over-current during constant speed operation.

Fungsi:

detect_requested_fault()

Kemudian regression test ditambahkan untuk:

* compact overcurrent deceleration;
* generic over current;
* overload;
* over voltage;
* ground fault.

Technical PCB Agent regression terakhir:

TOTAL: 31

PASSED: 31

FAILED: 0

---

# 13. GENERIC FAULT EXTRACTOR V1.3

File:

generic_fault_extractor.py

Generic Fault Extractor dikembangkan untuk membaca corrective action dari manual secara deterministik.

Public API utama:

CanonicalSpan

CanonicalDocument

normalize_space()

normalize_match_text()

clean_chunk_content()

longest_suffix_prefix_overlap()

build_canonical_document()

fault_query_score()

select_fault_name()

flexible_fault_pattern()

find_fault_boundaries()

extract_numbered_actions()

deduplicate_actions()

find_target_section()

spans_for_range()

extract_generic_fault_evidence()

Fault yang telah memiliki regression fixture:

## Over Current

Expected actions:

7

Expected page:

144

## Over Voltage

Expected actions:

4

Expected page:

144

## Overload

Expected actions:

3

Expected page:

144

## Over-current During Constant Speed Operation

Expected actions:

3

Expected page:

145

## Ground Fault

Expected actions:

2

Expected page:

146

Generic Fault Extractor regression:

FAILED: 0

---

# 14. BUG PENTING YANG SUDAH DITEMUKAN

## Router Gap

Live Telegram queries tidak masuk technical agent.

Status:

FIXED.

## Generic Fault Detection Gap

detect_requested_fault() tidak mengenali generic fault.

Status:

FIXED.

## Fault Bleeding

Extractor membaca action dari fault berikutnya.

Contoh:

Ground Fault

masuk ke:

Auto accel/decel failure

Communication Error

Status:

BELUM SEPENUHNYA SELESAI PADA LIVE DATABASE.

## Multiline Corrective Actions

Parameter:

Pr.07.02

hilang karena action manual terpotong antar-line.

Status:

FIXED.

## Generic Extractor Integration Type Error

StructuredFaultEvidence digunakan seperti dictionary.

Error:

TypeError:
'StructuredFaultEvidence' object is not subscriptable

Status:

FIXED menggunakan generic evidence builder yang kompatibel dengan build_structured_fault_answer().

## Neighbor Expansion db=None

Regression test memanggil:

answer_as_technical_pcb(db=None)

Neighbor expansion mencoba:

db.query()

Error:

AttributeError:
'NoneType' object has no attribute 'query'

Status:

FIXED.

## Neighbor Expansion Window Terlalu Besar

Semantic anchors yang tersebar menyebabkan satu canonical document besar dan bercampur.

Status:

FIXED menggunakan per-anchor neighbor window.

## Numbered Section Boundary

Helper ditambahkan:

is_numbered_section_heading()

Kemudian dihubungkan ke:

find_fault_boundaries()

Tujuannya mengenali heading seperti:

5.2 Ground Fault

5.3 Over Voltage (OV)

Status:

REGRESSION GREEN.

Live Ground Fault masih perlu penyempurnaan.

---

# 15. TEST STRATEGY

Prinsip development yang digunakan:

RED

↓

PATCH KECIL

↓

GREEN

↓

LIVE DATABASE SMOKE TEST

↓

RESTART SERVICE

↓

TELEGRAM PRODUCTION TEST

Setiap perubahan penting menggunakan:

1. baseline hash;
2. backup source;
3. syntax check;
4. regression test;
5. protected file verification;
6. live database smoke test;
7. service restart;
8. production Telegram test.

Jangan patch production code sebelum regression test merah tersedia untuk bug baru.

---

# 16. REGRESSION TEST UTAMA

File test utama:

test_agent_router.py

test_technical_pcb_agent.py

test_generic_fault_extractor.py

test_technical_knowledge.py

Baseline terakhir:

Agent Router:

18 PASS
0 FAIL

Technical PCB Agent:

31 PASS
0 FAIL

Generic Fault Extractor:

FAILED 0

Technical Knowledge:

8 PASS
0 FAIL

Catatan:

Regression hijau tidak otomatis berarti production ready.

Live database smoke test tetap wajib karena fixture test dapat berbeda dengan struktur chunk database nyata.

---

# 17. BETA PRODUCTION BASELINE

Service aktif:

multitek-ai-backend.service

multitek-ai-telegram.service

ollama.service

Status terakhir:

active

active

active

Production source hashes:

agent_router.py

5a9817d1f04dda4afc2cac027d3969e2bdba2510db2c14bf9707bfcf8acefaae

technical_pcb_agent.py

e47682e68f961593b6189279f401cbcc86bdb208af8060c53a59e4f6135bac93

generic_fault_extractor.py

46b7572d2045b24a12a52811ca41d122e2632bd70188e2d11d482c6b6eecd5a6

Status deployment:

BETA PRODUCTION ACTIVE

Target penggunaan:

INTERNAL MULTITEK / LIMITED PRODUCTION

Belum disarankan menjadi diagnosis otomatis langsung kepada customer tanpa verifikasi teknisi.

---

# 18. HASIL TELEGRAM PRODUCTION TERAKHIR

## Overload

Masih kemungkinan menggunakan LLM fallback.

Sumber yang muncul:

Delta VFD-EL halaman PDF 137.

Target deterministic corrective action sebelumnya berada pada halaman PDF 144.

Status:

PERLU DIPERBAIKI.

## Ground Fault

Menghasilkan 8 tindakan.

Expected deterministic actions:

2.

Terjadi fault bleeding ke:

Auto accel/decel failure

Communication Error

Status:

PERLU DIPERBAIKI.

## Over Current

Menghasilkan:

7 tindakan.

Status:

BERHASIL.

## Over Current During Constant Speed

Menghasilkan:

3 tindakan.

Status:

BERHASIL.

## Over Voltage

Menghasilkan:

4 tindakan.

Status:

BERHASIL SECARA ACTION COUNT.

Masih perlu memastikan tidak ada truncation text dan sumber benar.

---

# 19. PROSEDUR INSTALASI SERVER BARU

Urutan deployment yang disarankan:

## STEP 1 — SIAPKAN SERVER

Install:

* Ubuntu;
* Python;
* PostgreSQL;
* Ollama;
* Tailscale;
* Git jika source menggunakan repository.

## STEP 2 — BUAT DIRECTORY

/opt/multitek-ai/backend

## STEP 3 — COPY SOURCE CODE

Copy seluruh backend source ke:

/opt/multitek-ai/backend

Disarankan menggunakan Git repository private agar deployment berikutnya lebih mudah.

## STEP 4 — CREATE PYTHON VIRTUAL ENVIRONMENT

python -m venv .venv

Aktifkan:

source .venv/bin/activate

## STEP 5 — INSTALL DEPENDENCIES

Gunakan requirements.txt yang dikunci versinya.

Contoh:

pip install -r requirements.txt

## STEP 6 — INSTALL OLLAMA MODELS

Pastikan tersedia:

qwen3:0.6b

nomic-embed-text

## STEP 7 — CONFIGURE POSTGRESQL

Buat:

database

database user

password

connection configuration

## STEP 8 — CREATE DATABASE TABLES

Jalankan database initialization/migration.

Catatan:

Project saat ini perlu memiliki mekanisme migration formal sebelum deployment otomatis.

Disarankan menggunakan Alembic.

## STEP 9 — COPY KNOWLEDGE DOCUMENTS

Ada dua pendekatan.

Pendekatan A:

Backup dan restore PostgreSQL.

Pendekatan B:

Upload ulang semua manual kemudian re-index.

Untuk deployment cepat, backup/restore database lebih disarankan.

## STEP 10 — CONFIGURE TELEGRAM BOT

Set:

Telegram Bot Token

Admin ID

API endpoint

environment variables lainnya.

Secret jangan disimpan langsung di repository.

Gunakan:

.env

atau:

systemd EnvironmentFile.

## STEP 11 — INSTALL SYSTEMD SERVICES

Service:

multitek-ai-backend.service

multitek-ai-telegram.service

Enable:

systemctl enable multitek-ai-backend.service

systemctl enable multitek-ai-telegram.service

## STEP 12 — RUN REGRESSION TEST

Wajib jalankan:

python test_agent_router.py

python test_technical_pcb_agent.py

python test_generic_fault_extractor.py

python test_technical_knowledge.py

Expected:

ALL GREEN.

## STEP 13 — START BACKEND

systemctl restart multitek-ai-backend.service

Verifikasi:

systemctl status multitek-ai-backend.service

ss -ltnp

Pastikan:

127.0.0.1:8000

listening.

## STEP 14 — START TELEGRAM BOT

systemctl restart multitek-ai-telegram.service

Verifikasi:

systemctl status multitek-ai-telegram.service

journalctl -u multitek-ai-telegram.service

## STEP 15 — LIVE DATABASE SMOKE TEST

Uji fault yang sudah memiliki expected result:

Overload = 3 actions

Ground Fault = 2 actions

Over Current = 7 actions

Constant Speed Over Current = 3 actions

Over Voltage = 4 actions

Jangan hanya menghitung action.

Verifikasi juga:

* fault name;
* source page;
* brand;
* model;
* deterministic path;
* tidak ada fault bleeding;
* tidak ada text truncation.

## STEP 16 — TELEGRAM PRODUCTION TEST

Kirim query nyata melalui Telegram.

Bandingkan jawaban dengan manual.

Jika benar:

SERVER READY.

Jika salah:

CAPTURE FAILURE

↓

ADD REGRESSION TEST

↓

PATCH

↓

RUN REGRESSION

↓

LIVE DATABASE TEST

↓

REDEPLOY

---

# 20. CHECKLIST DEPLOYMENT SERVER BARU

Sebelum deployment:

[ ] Source repository tersedia.

[ ] requirements.txt tersedia.

[ ] .env.example tersedia.

[ ] PostgreSQL backup tersedia.

[ ] systemd unit files tersedia.

[ ] Ollama model list terdokumentasi.

[ ] regression tests tersedia.

[ ] production source hashes/tag tersedia.

[ ] knowledge documents tersedia.

[ ] Telegram token tersedia secara aman.

Sesudah deployment:

[ ] PostgreSQL aktif.

[ ] Ollama aktif.

[ ] qwen model tersedia.

[ ] embedding model tersedia.

[ ] backend aktif.

[ ] port 8000 listening.

[ ] Telegram service aktif.

[ ] router regression green.

[ ] technical PCB regression green.

[ ] generic extractor regression green.

[ ] technical knowledge regression green.

[ ] live database smoke test dijalankan.

[ ] Telegram real query test dijalankan.

---

# 21. PRIORITAS DEVELOPMENT BERIKUTNYA

## PRIORITAS 1

Perbaiki live Ground Fault extraction.

Expected:

2 corrective actions.

Tidak boleh bleeding ke section berikutnya.

## PRIORITAS 2

Pastikan Overload masuk deterministic path.

Expected:

3 corrective actions dari corrective-action section yang benar.

## PRIORITAS 3

Tambahkan production failure capture.

Simpan:

* query;
* selected agent;
* selected document;
* selected chunks;
* source pages;
* extraction mode;
* deterministic/LLM path;
* answer;
* timestamp.

## PRIORITAS 4

Tambahkan manual merek/model kedua.

Disarankan tetap satu kategori perangkat.

Contoh:

inverter merek lain.

Tujuannya menguji:

* brand selection;
* model selection;
* cross-document isolation;
* retrieval accuracy.

## PRIORITAS 5

Buat deployment automation.

Target:

install.sh

deploy.sh

backup.sh

restore.sh

smoke_test.sh

## PRIORITAS 6

Gunakan Git private repository.

Workflow:

development branch

↓

regression tests

↓

release tag

↓

deployment

Contoh release tag:

v1.4-beta.1

## PRIORITAS 7

Tambahkan database migration menggunakan Alembic.

## PRIORITAS 8

Buat requirements.txt dengan dependency versions yang dikunci.

## PRIORITAS 9

Tambahkan structured logging dan observability.

## PRIORITAS 10

Buat admin interface untuk:

* document management;
* re-index;
* disable document;
* inspect retrieval;
* inspect failed queries;
* inspect AI answer path.

---

# 22. REKOMENDASI AGAR INSTALASI SERVER BARU LEBIH CEPAT

Project jangan lagi hanya bergantung pada kumpulan source file dan perintah manual.

Buat deployment package dengan struktur:

multitek-ai/

backend/

systemd/

scripts/

docs/

tests/

requirements.txt

.env.example

README.md

VERSION

MANIFEST.sha256

Script minimum:

scripts/install.sh

scripts/deploy.sh

scripts/backup.sh

scripts/restore.sh

scripts/regression.sh

scripts/smoke_test.sh

Dengan struktur ini, deployment server baru dapat berubah dari proses manual panjang menjadi:

1. clone repository;
2. jalankan install.sh;
3. restore database;
4. pull Ollama models;
5. jalankan regression;
6. jalankan smoke test;
7. start services;
8. Telegram production test.

---

# 23. PRINSIP PENGEMBANGAN YANG HARUS DIPERTAHANKAN

1. Jangan patch production code tanpa memahami failure.

2. Buat regression test dari query nyata.

3. Patch sekecil mungkin.

4. Selalu buat backup sebelum perubahan.

5. Catat hash source penting.

6. Regression green bukan satu-satunya production gate.

7. Gunakan live database smoke test.

8. Uji melalui Telegram setelah service restart.

9. Technical answers harus berdasarkan manual.

10. Deterministic extraction diprioritaskan dibanding LLM jika manual memiliki structured corrective actions.

11. Jangan membuka diagnosis otomatis kepada customer sebelum kualitas technical path stabil.

12. Setiap bug production harus menjadi regression test baru.

---

# 24. STATUS AKHIR PROJECT

Multitek AI Assistant telah berkembang dari backend AI sederhana menjadi sistem multi-agent dengan:

* FastAPI backend;
* Telegram interface;
* PostgreSQL knowledge database;
* PDF ingestion;
* document metadata;
* embeddings;
* semantic retrieval;
* multi-query technical retrieval;
* Reciprocal Rank Fusion;
* technical evidence guard;
* per-anchor neighbor expansion;
* agent router;
* customer service knowledge isolation;
* technical knowledge isolation;
* deterministic structured fault extraction;
* generic fault extractor;
* LLM fallback;
* deterministic source summary;
* regression test suite;
* live database smoke testing;
* systemd production services.

Sistem saat ini aktif sebagai:

BETA PRODUCTION / INTERNAL LIMITED USE.

Fokus pengembangan berikutnya:

menstabilkan Ground Fault dan Overload pada live database, menambahkan manual merek/model lain, membangun failure capture, dan mengemas project menjadi deployment package yang dapat dipasang cepat pada server lain.

---

# 25. BASELINE UNTUK MELANJUTKAN PERCAKAPAN DEVELOPMENT

Jika development dilanjutkan pada percakapan baru, gunakan informasi berikut:

Project:

Multitek AI Assistant / agent AI.

Server backend:

/opt/multitek-ai/backend

Akses server:

SSH melalui Tailscale.

Current status:

V1.4 Beta Production aktif.

Services:

multitek-ai-backend.service

multitek-ai-telegram.service

ollama.service

Current production hashes:

agent_router.py:
5a9817d1f04dda4afc2cac027d3969e2bdba2510db2c14bf9707bfcf8acefaae

technical_pcb_agent.py:
e47682e68f961593b6189279f401cbcc86bdb208af8060c53a59e4f6135bac93

generic_fault_extractor.py:
46b7572d2045b24a12a52811ca41d122e2632bd70188e2d11d482c6b6eecd5a6

Regression status:

Agent Router:
18/18 PASS.

Technical PCB Agent:
31/31 PASS.

Generic Fault Extractor:
0 FAILED.

Technical Knowledge:
8/8 PASS.

Known live production issues:

Ground Fault:
fault bleeding, actual 8 actions, expected 2.

Overload:
kemungkinan masih menggunakan LLM fallback dan mengambil halaman 137, target deterministic corrective actions berada pada halaman 144.

Over Current:
7 actions, working.

Constant Speed Over Current:
3 actions, working.

Over Voltage:
4 actions, working by action count.

Next recommended stage:

Production Failure Capture / regression tests for exact live Ground Fault and Overload failures before further production patching.
