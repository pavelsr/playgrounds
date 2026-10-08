# 🇬🇧 Playgrounds

Docker Compose stacks for experimental and study database servers (PostgreSQL, ClickHouse, and others). Each numbered folder is one stack. From that folder, start it with a single command:

```bash
docker compose up -d
```

Dataset files are not stored in the repository. An init container downloads them into a named volume before the database starts.

**The README in each subfolder is written in Russian.**

## 🇷🇺 Playgrounds

Набор Docker Compose-стеков тестовых серверов СУБД (PostgreSQL, ClickHouse и другие) для экспериментов и учёбы. Каждая нумерованная папка — один стек. Запуск из этой папки одной командой:

```bash
docker compose up -d
```

Файлы датасетов в репозиторий не входят. Init-контейнер скачивает их в named volume до старта базы.

**README в каждой подпапке написан на русском языке.**
