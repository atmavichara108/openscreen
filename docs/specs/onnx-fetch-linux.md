# Спека: ONNX Runtime для linux-x64 — пере-пин на PyPI

- Дата: 2026-11-19. Ветка: `task/adapt-korabl-20261118`.
- Реализация: правка `scripts/fetch-onnxruntime.mjs` + его теста.
- Статус: реализация и сухой прогон выполнены; вердиктор-приёмка — отдельным ходом
  (линия `mistral/codestral-2508`, как у закрытого BF-20261118-02).

## Цель

`npm run fetch:onnxruntime` на Linux должен работать из сети Корабля. Сегодня он
даёт 404: `microsoft/onnxruntime` заблокирован на уровне репозитория (404 и на
странице, и на релизах — даже win-x64 ассете, чей SHA уже прописан в скрипте).
Без библиотеки пакет Linux не собирается (`before-pack.cjs`, `LINUX_REQUIRED`), и
фоновый эффект камеры (сегментация) молча выключен.

## Scope

- `scripts/fetch-onnxruntime.mjs`: запись `linux-x64` → pinned-URL на PyPI-wheel
  `onnxruntime` 1.27.0 (manylinux, содержит `libonnxruntime.so.1.27.0`);
  загрузка через `curl` (уважает `HTTPS_PROXY`, которого undici не читает);
  распаковка `.whl` через `python3 -m zipfile`; проверка версии per-entry.
- `scripts/fetch-onnxruntime.test.mjs`: инвариант «sha/tag у всех записей,
  slug только у шаблонных».
- Вне scope: win/mac записи (пины 1.27.1 не трогаются), глобальный `VERSION`,
  зеркало в релизах форка (долгосрочная форма — когда байты смогут уйти из сети).

## Поведение (критерии)

1. `node scripts/fetch-onnxruntime.mjs --force` на linux-x64: скачивает wheel,
   SHA-256 совпадает с пином (`445fb702…`), верификация даёт
   «MIT ONNX Runtime 1.27.0, ELF, 22.6 MB».
2. В `electron/native/bin/linux-x64/` появляется `libonnxruntime.so` + штамп
   `.onnxruntime-linux-x64.json` (archive sha + library sha).
3. Пины win32-x64/win32-arm64/darwin-arm64/linux-arm64 не изменены; глобальный
   `VERSION` = 1.27.1 (Cargo `api-27` floor: minor 27 у 1.27.0 соблюдён).
4. `npx vitest run scripts/fetch-onnxruntime.test.mjs` — 12/12 PASS.
5. `npx biome check scripts/fetch-onnxruntime.mjs` — чисто.

## Приёмка

- [x] Состав правки: два файла, лишнего нет.
- [x] Сухой прогон повторяем (`--force`, оба шага совпадают).
- [x] Тесты 12/12, линт чист.
- [ ] Вердиктор на модели, отличной от автора (Р-109).

## Риски

- PyPI-файл иммутабелен по версии; SHA записи — тот же trust, что у upstream-пина.
- wheel собран под manylinux_2_28 — glibc-пол Manjaro закрывает; на более старом
  хосте потребовался бы другой источник.