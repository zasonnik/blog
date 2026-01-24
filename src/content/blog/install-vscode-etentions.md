---
title: Устанавливаем вручную расширения VSCode
description: Если при установке расширений VSCode из маркета что то идет не так 
pubDate: 2026-1-24
---

Сегодня мне припекло поставить расширение для VSCode. Проблема в том, что при нажатии на кнопку `Install` в маркете ничего не происходило. Делюсь решением:

## Найти настоящее имя расширения

Для VSCode расширения именуются `<Имя автора>.<Название расширения>`. Быстрее всего его найти, нажав по заголовку расширения, где откроется страница с ним.

В моём случае это `tamasfe.even-better-toml` - имя есть и в URL страницы и в автогнерируемой CLI команде для установки.

## Найти vsix пакет

Я делал это вызывая установку в дебаг режиме в терминале

```sh
code --verbose --install-extension tamasfe.even-better-toml
```

Из всего вывода мне интересен только URL вида `https://tamasfe.gallery.vsassets.io/_apis/public/gallery/publisher/tamasfe/extension/even-better-toml/0.21.2/assetbyname/Microsoft.VisualStudio.Services.VSIXPackage?install=true`

Т.е. содержащий `VSIXPackage`.

Далее я его просто wget'нул

```sh
wget <URL> -o <package_name>.vsix
```

## Установка

Я использовал UI VSCode (`Extentions` - `View and more actions` (троеточние в заголовке панели) - `Install from VSXI`).

Всё, он появился в VSCode и работает.
