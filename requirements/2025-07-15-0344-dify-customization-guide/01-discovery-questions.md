# Discovery Questions

## Q1: Apakah programmer junior akan membuat custom tools untuk AI agent mereka?
**Default if unknown:** Yes (custom tools adalah cara paling umum untuk extend Dify functionality)

**Reasoning:** Berdasarkan analisis codebase, sistem tool Dify sangat extensible dan merupakan extension point utama. Custom tools memungkinkan AI agent mengakses external APIs, databases, atau service lain.

---

## Q2: Apakah panduan ini perlu mencakup cara deploy customization ke production environment?
**Default if unknown:** Yes (programmer junior perlu tahu cara deploy hasil development mereka)

**Reasoning:** Dify adalah platform production-ready, sehingga panduan harus mencakup best practices untuk deployment agar customization tidak rusak saat update.

---

## Q3: Apakah programmer junior akan memodifikasi workflow nodes yang sudah ada atau membuat workflow nodes baru?
**Default if unknown:** No (lebih aman menggunakan existing nodes dan fokus pada custom tools)

**Reasoning:** Workflow nodes adalah bagian core engine. Untuk programmer junior, lebih baik fokus pada custom tools yang bisa digunakan dalam workflow existing.

---

## Q4: Apakah panduan perlu mencakup integrasi dengan model providers custom (seperti local models)?
**Default if unknown:** Yes (banyak use case memerlukan model custom atau local deployment)

**Reasoning:** Dari struktur `/api/core/model_runtime/model_providers/`, terlihat Dify mendukung multiple model providers dan ini adalah extension point yang aman.

---

## Q5: Apakah programmer junior akan menggunakan plugin system atau fokus pada code-based extensions?
**Default if unknown:** No (plugin system lebih complex, code-based extensions lebih suitable untuk junior developers)

**Reasoning:** Plugin system memerlukan pemahaman lebih dalam tentang Dify architecture. Code-based extensions di `/api/core/extension/` lebih straightforward untuk programmer junior.