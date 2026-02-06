---
title: "2 Справочник фильтров Jinja2"
description: "Полезные фильтры для преобразования данных внутри шаблонов и плейбуков."
---

## Значения по умолчанию
*   `{{ my_var | default('default_value') }}`: Если переменная не определена.
*   `{{ my_var | default('val', true) }}`: Если переменная определена, но пустая (null/false).

## Форматирование данных
*   `{{ my_dict | to_yaml }}`: Превратить словарь в YAML.
*   `{{ my_dict | to_json }}`: Превратить в JSON.
*   `{{ my_dict | to_nice_json }}`: Красивый JSON с отступами.

## Строковые операции
*   `{{ "hello" | upper }}` → "HELLO"
*   `{{ "/var/www/html" | basename }}` → "html"
*   `{{ "password" | b64encode }}` → Base64 строка.
*   `{{ "MTIz" | b64decode }}` → "123"

## Списки
*   `{{ [1, 2, 2] | unique }}` → `[1, 2]`
*   `{{ ['a', 'b'] | join(',') }}` → "a,b"
*   `{{ [1, 2, 3] | random }}` → Случайный элемент.
*   `{{ list1 | difference(list2) }}` → Вычитание списков.
