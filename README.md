# Hass Alerting

## 1. NotiScript

- <https://www.home-assistant.io/integrations/alert/>
- <https://github.com/redstone99/hass-alert2/>
- <https://github.com/sejnub/ha-notiscript>

## 2. My API

script.change_alert(alert_id, operation): result

| Service    | Meaning                               |
| ---------- | ------------------------------------- |
| activate   | Turns on  a single or multiple alerts |
| deactivate | Turns off a single or multiple alerts |
|            |                                       |
| ack        | acks   a single or multiple alert     |
| unack      | unacks a single or multiple alert     |
|            |                                       |

### 2.1. My implementation with alert integration

- each alert `alert.klaus` gets one `input_boolean.alert_klaus`

|                                        |     |                               |               |                             |
| -------------------------------------- | --- | ----------------------------- | ------------- | --------------------------- |
| script.change_alert(klaus, activate)   | ->  | call `input_boolean.turn_on`  | on the target | `input_boolean.alert_klaus` |
| script.change_alert(klaus, deactivate) | ->  | call `input_boolean.turn_off` | on the target | `input_boolean.alert_klaus` |
| script.change_alert(klaus, ack)        | ->  | call `alert.turn_off`         | on the target | `alert.klaus`               |
| script.change_alert(klaus, unack)      | ->  | call `alert.turn_on`          | on the target | `alert.klaus`               |


## 3. Alert Integration

### 3.1. Zustandsdiagramm

```mermaid

stateDiagram-v2
    [*] --> startup

    startup --> idle: Entity in normal state
    startup --> on: Entity in problem state

    idle --> on: Entity enters problem state
    idle --> idle: alert.turn_off (no effect)
    idle --> on: alert.turn_on (manual activation)

    on --> idle: Entity exits problem state
    on --> off: alert.turn_off (acknowledge)
    on --> on: alert.turn_on (no effect)

    off --> idle: Entity exits problem state
    off --> off: alert.turn_off (no effect)
    off --> on: alert.turn_on (manual reactivation)

    note right of startup
        Home Assistant starts
        Alert state is determined based on the entity's current state
    end note

    note right of idle
        state = idle
        Alert is inactive
    end note

    note right of on
        state = on
        Alert is active; notifications are being sent
        - If skip_first: false → Immediate notification upon activation
        - If skip_first: true → First notification is delayed until the repeat interval
    end note

    note right of off
        state = off
        Alert condition persists, but notifications are suppressed due to acknowledgment
        Acknowledgment persists across Home Assistant restarts
    end note



```

### 3.2. Eigenschaften der Alert-Entität

State ()

| Eigenschaft              | Beschreibung                             | Beispiel                     |
| ------------------------ | ---------------------------------------- | ---------------------------- |
| `state`                  | Aktueller Zustand der Alert-Entität      | `idle`, `on`, `acknowledged` |
| Attribut `friendly_name` | Anzeigename der Entität                  | "Wassersensor ausgelöst"     |
| Attribut `message`       | Text der Benachrichtigung                | "Wasser unter der Spüle!"    |
| Attribut `title`         | Titel der Nachricht (optional)           | "Achtung"                    |
| Attribut `done_message`  | Nachricht, wenn entity wieder "off" wird | "Alles trocken"              |
| Attribut `repeat`        | Liste von Intervallen in Minuten         | `[10, 30, 60]`               |
| Attribut `acknowledged`  | Gibt an, ob quittiert wurde              | `false`                      | `true` |
|                          |                                          |                              |

### 3.3. Beispiel-Konfiguration (configuration.yaml)

```yaml
alert:
  wasser_leck:
    name: Wassersensor ausgelöst
    entity_id: binary_sensor.wasser_unter_spuele
    state: "on"
    repeat: [10, 30]
    skip_first: false
    notifiers:
      - mobile_app_mein_telefon
    title: "Achtung!"
    message: "Wasser unter der Spüle erkannt!"
    done_message: "Alles trocken."
    acknowledge: true
```

### 3.4. UI-Button zum Quittieren (Lovelace)

```yaml
type: button
name: Alert quittieren
icon: mdi:check
tap_action:
  action: call-service
  service: alert.dismiss
  target:
    entity_id: alert.wasser_leck
```

## 4. Alert2 Integration

| Service                     | Meaning                                                                                                                                                                       |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| alert2.ack_all              | acks all alerts.                                                                                                                                                              |
| alert2.ack                  | acks a single alert.                                                                                                                                                          |
| alert2.unack                | unacks a single alert.                                                                                                                                                        |
|                             |                                                                                                                                                                               |
| alert2.report               | activate the specified alert, causing it to fire and send notifications as configured. This action overrides any conditions or triggers defined in the alert's configuration. |
| alert2.manual_on            | turns on a condition alert that was configured with manual_on: true.                                                                                                          |
| alert2.manual_off           | turns off a condition                                                                                                                                                         |
|                             |                                                                                                                                                                               |
| alert2.notification_control | adjust the notification settings.                                                                                                                                             |
|                             |                                                                                                                                                                               |

### 4.1. https://community.home-assistant.io/t/alert2-a-new-alerting-component/654597/201

Notifiers:

- `notifier`
- `summary_notifier`

Messages

|                  | Content                                   | Note                        |
| ---------------- | ----------------------------------------- | --------------------------- |
| First message    | {{friendly_name}}: {{message}}            |                             |
| Reminder message | {{friendly_name}}: on for {{min}} minutes |                             |
| Last message     | {{friendly_name}}: {{done_message}}       | if {{annotate_message}}     |
| Last message     | {{done_message}}                          | if not {{annotate_message}} |
| Summary          |                                           |                             |
|                  |                                           |                             |


#### LLM

I am writing a Home Assistant script
I have an incoming field with name `message` this can have the following forms where the numbers can be any one, two or three digit numbers

 "<This is some text without colons>: on for 53 s"
 "<This is some text without colons>: on for 4 min"
 "<This is some text without colons>: on for 12 h"
 "<This is some text without colons>: on for 12 d"


If `message contains no colon do not make any replacements`
If `message contains one colon (like in the given texts) make the folloowing replacement`: 
- replace "on for <number> s with "seit <number> Sekunden
- replace "on for <number> min with "seit <number> Minuten
- replace "on for <number> h with "seit <number> Stunden
- replace "on for <number> d with "seit <number> Tagen


####

Hi

I came to your solution because in my installation the standard alert integration seems buggy and only sent the first notification but not the “reminders”

Now i invested in moving everything to alert2 to now find out that i can not customize the reminder messages. Did i get that right?

I am living in germany and use the notifications mainly for voice. The standard reminder message is not very useful for us especially those members here who dont speak english.

Is there a way to customize that messages and if not: Is there a chance you would consider adding that?

I am very sorry if there already was a discussion about this topic and i missed it.

Thank you very much for your work
ETX
