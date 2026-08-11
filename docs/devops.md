## 11. DevOps

### Docker

Image — иммутабельный снимок файловой системы + метаданные.
Container — запущенный экземпляр image; изолирован через namespaces + cgroups.
Ключевые инструкции Dockerfile: FROM, RUN, COPY, ENV, EXPOSE, CMD, ENTRYPOINT.
Multi-stage builds: маленький финальный image без build-зависимостей.

---
