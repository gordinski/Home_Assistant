# Інтеграція графіків відключень Рівнеобленерго в Home Assistant

Ця документація описує процес налаштування парсингу даних з сайту Рівнеобленерго, створення сенсора з форматованим текстом та налаштування "розумних" сповіщень у Telegram, які спрацьовують лише при реальних змінах у графіку.

## 1. Попередні вимоги

1. Встановлений [HACS](https://hacs.xyz/).
2. Встановлена інтеграція [ha-multiscrape](https://github.com/danieldotnl/ha-multiscrape).
3. Налаштований [Telegram Bot](https://www.home-assistant.io/integrations/telegram_bot/) в Home Assistant.

## 2. Налаштування сенсорів (Multiscrape)

Додайте наступний код у ваш `configuration.yaml`. Парсер кожні 15 хвилин перевіряє сайт на наявність змін.

```yaml
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

      # Рядок 5 - дата
      - unique_id: roe_date_row5
        name: "ROE Date Row 5"
        select: "#fetched-data-container table tbody tr:nth-child(5) td:nth-child(1)"
        value_template: >
          {{ value | regex_findall('(\\d{2}\\.\\d{2}\\.\\d{4})') | first | default('unknown') }}

      # Рядок 5 - дані відключень (черга 6.2)
      - unique_id: roe_raw_row5
        name: "ROE Raw Row 5"
        select: "#fetched-data-container table tbody tr:nth-child(5) td:nth-child(13)"
        value_template: "{{ value | trim }}"

      # Рядок 6 - дата
      - unique_id: roe_date_row6
        name: "ROE Date Row 6"
        select: "#fetched-data-container table tbody tr:nth-child(6) td:nth-child(1)"
        value_template: >
          {{ value | regex_findall('(\\d{2}\\.\\d{2}\\.\\d{4})') | first | default('unknown') }}

      # Рядок 6 - дані відключень (черга 6.2)
      - unique_id: roe_raw_row6
        name: "ROE Raw Row 6"
        select: "#fetched-data-container table tbody tr:nth-child(6) td:nth-child(13)"
        value_template: "{{ value | trim }}"
```

## 3. Форматування повідомлення (Template Sensor)

Цей сенсор перетворює "сирі" дані з сайту у зручний для читання вигляд з розрахунком годин наявності світла:

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

Додайте в розділ `template:`:

```yaml
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
              
              {# Поточні дати #}
              {%- set today = as_local(now()).strftime('%d.%m.%Y') -%}
              {%- set tomorrow = (as_local(now()) + timedelta(days=1)).strftime('%d.%m.%Y') -%}

              {# Дані з сайту #}
              {%- set date_row5 = states('sensor.roe_date_row5') -%}
              {%- set raw_row5 = states('sensor.roe_raw_row5') -%}
              {%- set date_row6 = states('sensor.roe_date_row6') -%}
              {%- set raw_row6 = states('sensor.roe_raw_row6') -%}

              {# Визначаємо дані для сьогодні та завтра #}
              {%- set today_data = '' -%}
              {%- set tomorrow_data = '' -%}
              
              {%- if date_row5 == today -%}
                {%- set today_data = raw_row5 -%}
              {%- elif date_row6 == today -%}
                {%- set today_data = raw_row6 -%}
              {%- endif -%}
              
              {%- if date_row5 == tomorrow -%}
                {%- set tomorrow_data = raw_row5 -%}
              {%- elif date_row6 == tomorrow -%}
                {%- set tomorrow_data = raw_row6 -%}
              {%- endif -%}

              {# Формуємо список для виводу #}
              {%- set days = [
                {'date': today, 'raw': today_data},
                {'date': tomorrow, 'raw': tomorrow_data}
              ] -%}

              {%- for day in days -%}
                {%- if day.raw or loop.first -%}
                  {%- set ns.result = ns.result ~ "\n" ~ day.date ~ "\n——————" -%}

                  {%- set times = day.raw | regex_findall('(\\d{1,2}:\\d{2})\\s*[\\-–—]\\s*(\\d{1,2}:\\d{2})') -%}

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
                {%- endif -%}
              {%- endfor -%}

              {{ ns.result | trim }}

            _Оновлено: {{ update_info }}_
            {%- endif -%}
```

## 4. Налаштування допоміжного елемента (Helper)

Для уникнення повторних сповіщень при зміні лише часу оновлення (а не самого графіка), потрібно створити "пам'ять" для автоматизації:

1. Перейдіть у **Налаштування** > **Пристрої та сервіси** > **Допоміжні елементи**.
2. Натисніть **Створити допоміжний елемент** > **Текстовий елемент**.
3. Налаштуйте його:
   - **Назва:** `ROE Last Sent Schedule`
   - **ID:** `input_text.roe_last_sent_schedule`
   - **Максимальна довжина:** 255

## 5. Автоматизація сповіщень

Ця автоматизація спрацьовує при зміні часу на сайті, але надсилає повідомлення лише якщо текст графіка змінився.

```yaml
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
          {% set current_content = states('sensor.roe_raw_row5') ~
          states('sensor.roe_raw_row6') ~  states('sensor.roe_date_row5') ~
          states('sensor.roe_date_row6') %} {% set last_sent =
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
            {{ (states('sensor.roe_raw_row5') ~  states('sensor.roe_raw_row6') ~
            states('sensor.roe_date_row5') ~ states('sensor.roe_date_row6')) |
            truncate(250) }}
    else:
      - stop: Графік для нашої черги не змінився, ігноруємо сповіщення.
mode: queued
```
