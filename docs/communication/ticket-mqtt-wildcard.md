# Ticket à ouvrir sur GladysAssistant/Gladys

> À coller dans un nouveau ticket « Bug report » : https://github.com/GladysAssistant/Gladys/issues/new?template=bug_report.yml. Titre et corps ci-dessous, en anglais comme le reste du dépôt Gladys. Les sections correspondent aux champs du formulaire.

**Title**: Scene trigger "MQTT message received" never fires when the topic contains a wildcard (+ or #)

---

### Describe the bug

A scene trigger of type `mqtt.received` whose topic contains an MQTT wildcard (`+` or `#`) never fires.

When the scene is loaded, `scene.addScene` subscribes to the trigger topic **as a filter** through the MQTT service (`server/lib/scene/scene.addScene.js`, around lines 240-253). The MQTT service correctly matches incoming messages against that filter with `mqttTopicMatches` (`server/services/mqtt/lib/handleNewMessage.js`), and calls the scene callback with the **actual** topic received (e.g. `home/kitchen/button`).

The scene trigger check then compares that actual topic with the filter using strict equality (`server/lib/scene/scene.triggers.js`, `EVENTS.MQTT.RECEIVED` handler, around lines 250-251):

```js
event.topic === trigger.topic && (!trigger.message || trigger.message === event.message)
```

`'home/kitchen/button' === 'home/+/button'` is always false, so the scene is never executed, although the subscription and the message delivery both work.

Found while reading the code (commit `71edb17` on `master`), not yet reproduced on a running instance.

### To reproduce

1. Configure the MQTT integration with a broker.
2. Create a scene with an "MQTT message received" trigger on topic `test/+/button` (no message filter) and any action (e.g. send a message).
3. Publish any payload on `test/kitchen/button`.
4. The scene is not executed. With the trigger topic set to `test/kitchen/button` instead, it is.

### Expected behavior

The trigger should fire for every topic that matches the filter, with MQTT wildcard semantics — the same matching the MQTT service already applies to its own bindings.

Suggested fix: in the `EVENTS.MQTT.RECEIVED` trigger handler, replace `event.topic === trigger.topic` with `mqttTopicMatches(trigger.topic, event.topic)` (`server/services/mqtt/lib/mqttTopicMatches.js`), and add a test with `+` and `#` filters.

### Gladys Assistant version

`master` at `71edb17` (5.3.3)

### Installation type

Other (found by code reading)

### Additional context

Spotted while studying the scene engine for a Home Assistant → Gladys migration tool: Home Assistant automations with MQTT wildcard triggers cannot be converted to Gladys scenes until this works.
