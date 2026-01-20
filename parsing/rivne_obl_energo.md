# Парсим графіки відключень з Рівнеобленерго

1. В HACS встановлюємо [ha-multiscrape](https://github.com/danieldotnl/ha-multiscrape/tree/v8.0.5)
2. В `configuration.yaml` створюю сенсори:
   1. `roe_update_time` - час оновлення даних
   2. `roe_raw_6_2_today` - часи відключень для черги 6.2 сьогодні
   3. `roe_raw_6_2_tomorrow` - часи відключень для черги 6.2 завтра

Парсер кожні 900 с (15 хв) йде на сайт і забирає інформацію для сенсорів.

```
multiscrape:
  - name: Rivneoblenergo_Schedules
    resource: https://www.roe.vsei.ua/disconnections/
    scan_interval: 900
    sensor:
      - unique_id: roe_update_time
        name: "ROE Оновлено на сайті"
        select: ".entry-content"
        value_template: >
          {{ value | regex_findall('Оновлено: (\\d{2}\\.\\d{2}\\.\\d{4} \\d{2}:\\d{2})') | first }}

      - unique_id: roe_raw_6_2_today
        name: "ROE Raw 6.2 Today"
        select: "#fetched-data-container table tbody tr:nth-child(5) td:nth-child(13)"
        value_template: "{{ value | trim }}"

      - unique_id: roe_raw_6_2_tomorrow
        name: "ROE Raw 6.2 Tomorrow"
        select: "#fetched-data-container table tbody tr:nth-child(6) td:nth-child(13)"
        value_template: "{{ value | trim }}"
```

## Створюю автоматизацію для розумного сповіщення про графік відключення.

Для того, щоб порівнювати нові дані з попередніми на наявність змін створюємо текстовий елемент: `Налаштування` > `Пристрої та сервіси` > `Помічники` > `Створити` > `Текстовий елемент (input_text)`:

- **Назва:** ROE Last Sent Schedule
- **ID:** input_text.roe_last_sent_schedule
- **Максимальна довжина:** 255.

Якщо графік відрізняється від попереднього - відправляю повідомлення з новим графіком в телеграм бот. Якщо ні - оновлюється лише час оновлення.

```
alias: "ROE: Розумне сповіщення про графік"
description: Відправляє графік тільки якщо змінилися години відключень
triggers:
  - trigger: state
    entity_id: sensor.roe_update_time
conditions:
  - condition: template
    value_template: |
      {{ trigger.from_state is not none and
         trigger.to_state is not none and
         trigger.to_state.state != trigger.from_state.state and
         trigger.to_state.state not in ['unknown', 'unavailable'] }}
actions:
  - delay: "00:00:02"
  - if:
      - condition: template
        value_template: >
          {% set current_content = states('sensor.roe_raw_6_2_today') ~
          states('sensor.roe_raw_6_2_tomorrow') %} {% set last_sent =
          states('input_text.roe_last_sent_schedule') %} {{ current_content !=
          last_sent }}
    then:
      - action: telegram_bot.send_message
        data:
          parse_mode: Markdown
          message: |
            🔔 *Знайдено новий графік відключень електроенергії для Черги 6.2*

            {{ state_attr('sensor.roe_telegram_message', 'formatted_text') }}
      - action: input_text.set_value
        target:
          entity_id: input_text.roe_last_sent_schedule
        data:
          value: >-
            {{ (states('sensor.roe_raw_6_2_today') ~
            states('sensor.roe_raw_6_2_tomorrow')) | truncate(250) }}
    else:
      - stop: Графік для нашої черги не змінився, ігноруємо сповіщення.
mode: queued
```

## Форматування повідомлення для телеграм бота

Для гарного відображення результату

```
📋 Актуальний графік відключень (Черга 6.2)

20.01.2026
——————
✖️ 00:00 - 02:00 (2 год)
⚡️ 4 год
✖️ 06:00 - 11:00 (5 год)
⚡️ 3 год
✖️ 14:00 - 20:00 (6 год)

21.01.2026
——————
ℹ️ Очікується

Оновлено: 20.01.2026 14:30
```

 в `configuration.yaml` додаю `template` з сенсором `roe_telegram_message`:

```
template:
  - sensor:
      - name: "ROE Telegram Message"
        unique_id: roe_telegram_message
        state: "{{ states('sensor.roe_update_time') }}"
        attributes:
          formatted_text: >
            {%- set update_info = states('sensor.roe_update_time') -%}
            {%- if update_info in ['unknown', 'unavailable', 'None'] -%}
              ⚠️ Очікування даних від сервера...
            {%- else -%}
              {%- set ns = namespace(result="") -%}
              {%- set raw_today = states('sensor.roe_raw_6_2_today') -%}
              {%- set raw_tomorrow = states('sensor.roe_raw_6_2_tomorrow') -%}

              {%- set date_today = update_info.split(' ')[0] -%}
              {%- set date_tomorrow = (now() + timedelta(days=1)).strftime('%d.%m.%Y') -%}

              {%- set days = [
                {'date': date_today, 'raw': raw_today},
                {'date': date_tomorrow, 'raw': raw_tomorrow}
              ] -%}

              {%- for day in days -%}
                {%- set ns.result = ns.result ~ "\n" ~ day.date ~ "\n——————" -%}

                {%- set times = day.raw | regex_findall('(\d{1,2}:\d{2})\s*[\-–—]\s*(\d{1,2}:\d{2})') -%}

                {%- if times | length == 0 -%}
                  {%- if day.raw | length > 2 -%}
                    {%- set ns.result = ns.result ~ "\nℹ️ " ~ day.raw ~ "\n" -%}
                  {%- else -%}
                    {%- set ns.result = ns.result ~ "\n✅ Відключень не заплановано\n" -%}
                  {%- endif -%}
                {%- else -%}
                  {%- for i in range(times | length) -%}
                    {%- set start = times[i][0] -%}
                    {%- set end = times[i][1] -%}

                    {%- set t1 = (start.split(':')[0]|int * 60) + start.split(':')[1]|int -%}
                    {%- set t2 = (end.split(':')[0]|int * 60) + end.split(':')[1]|int -%}
                    {%- set duration = t2 - t1 if t2 > t1 else (1440 - t1 + t2) -%}

                    {%- set ns.result = ns.result ~ "\n✖ " ~ start ~ " - " ~ end ~ " (" ~ (duration // 60) ~ " год)" -%}

                    {%- if i < (times | length - 1) -%}
                      {%- set n_start = times[i+1][0] -%}
                      {%- set nt1 = (n_start.split(':')[0]|int * 60) + n_start.split(':')[1]|int -%}
                      {%- set light = nt1 - t2 -%}
                      {%- if light > 0 -%}
                        {%- set ns.result = ns.result ~ "\n⚡ " ~ (light // 60) ~ " год" -%}
                      {%- endif -%}
                    {%- endif -%}
                  {%- endfor -%}
                {%- endif -%}
                {%- set ns.result = ns.result ~ "\n" -%}
              {%- endfor -%}

              {# Додаємо пустий рядок та дату оновлення в кінці #}
              {{ ns.result | trim }}

            _Оновлено: {{ update_info }}_
            {%- endif -%}
```
