# Expert Detail Questions

## Q6: Apakah panduan harus mencakup setup database PostgreSQL dengan pgvector extension secara manual?
**Default if unknown:** No (Docker setup sudah include semua dependencies, lebih mudah untuk junior developers)

**Technical Context:** Dify memerlukan PostgreSQL dengan pgvector extension untuk vector operations. Docker Compose setup di `/docker/docker-compose.yaml` sudah menghandle ini otomatis.

---

## Q7: Apakah programmer junior perlu memahami sistem authentication dan tenant isolation di Dify?
**Default if unknown:** Yes (custom tools dan extensions harus handle user_id dan tenant_id dengan benar)

**Technical Context:** Setiap tool dan extension menerima `user_id` parameter dan harus respect tenant isolation. Ini critical untuk security dan multi-tenancy.

---

## Q8: Apakah panduan perlu mencakup cara menambahkan dukungan bahasa Indonesia (id-ID) untuk custom tools?
**Default if unknown:** Yes (membuatnya lebih accessible untuk developer Indonesia dan sesuai dengan pattern i18n yang ada)

**Technical Context:** Pattern YAML di `/web/i18n/languages.json` sudah define `id-ID` tapi belum fully supported. Custom tools menggunakan multi-language labels.

---

## Q9: Apakah programmer junior perlu memahami sistem SSRF protection dan sandbox execution?
**Default if unknown:** No (cukup tahu bahwa external calls harus melalui tools system, detail security dihandle otomatis)

**Technical Context:** Dify menggunakan `ssrf_proxy` service dan sandbox execution. Junior developers cukup menggunakan existing patterns tanpa modify security layer.

---

## Q10: Apakah panduan harus mencakup cara testing custom tools dengan pytest framework yang ada?
**Default if unknown:** Yes (testing adalah best practice dan Dify sudah punya test infrastructure yang bisa digunakan)

**Technical Context:** Di `/api/pytest.ini` sudah ada test configuration dengan coverage. Custom tools harus di-test untuk memastikan reliability.