---
title: Bayesian
description: Instructions on how to integrate threshold Bayesian sensors into Home Assistant.
ha_category:
  - Binary sensor
  - Helper
  - Utility
ha_iot_class: Local Polling
ha_release: 0.53
ha_quality_scale: internal
ha_domain: bayesian
ha_platforms:
  - binary_sensor
ha_integration_type: helper
ha_codeowners:
  - '@HarvsG'
related:
  - docs: /docs/configuration/
    title: Configuration file
---

A `bayesian` binary sensor is a virtual sensor that determines its state based on the probabilistic combination of multiple "observed" sensor states. By applying [Bayes' rule](https://en.wikipedia.org/wiki/Bayes%27_theorem), it estimates the likelihood that a specific event is occurring based on the combination of the current states of the observed sensors and a baseline probability - known as a `prior`. When the calculated probability - known as a 'posterior' - exceeds the defined `probability_threshold`, the `bayesian` sensor will turn `on`; otherwise, it will be `off`.

This approach enables the detection of complex events that are not directly or easily measurable, such as cooking, showering, being in bed, or starting a morning routine. Additionally, it can improve confidence in events that are measurable but where sensors may be unreliable, such as detecting presence.

Both UI and YAML setup is supported, importantly YAML uses probabilities of `0` to `1` whereas UI uses percentages.

## Theory

A fundamental concept in Bayes' Rule is the distinction between the probability of an *event given an observation* and the probability of an *observation given an event*. These two probabilities are not interchangeable and must be considered separately. While they may be similar in some cases—for example, when motion sensors are accurate, the probability that someone is in the room *given* that motion is detected is often close to the probability that motion is detected *given* someone is in the room—they are usually quite different.

Consider a more nuanced example: the probability that someone has just arrived home (the event) *given* the front door contact sensor reports open (the observation) is relatively low (e.g., p = 0.2). However, the probability that the front door contact sensor reports open (the observation) *given* someone came home (the event) is very high (e.g., p = 0.999). This difference exists because the door may be opened multiple times throughout the day for reasons unrelated to someone arriving home, but someone always opens the door when they arrive home.

When configuring a Bayesian binary sensor, you should define the probability of the observation being true given the event (the assumed state of the Bayesian sensor).

## Configuration

{% include integrations/config_flow.md %}

To configure a YAML Bayesian sensor, add an entry using the following structure to your {% term "`configuration.yaml`" %} file.
 {% include integrations/restart_ha_after_config_inclusion.md %}

```yaml
# Example configuration.yaml entry
binary_sensor:
  - platform: bayesian
    name: "Kitchen Occupied"
    prior: 0.3
    probability_threshold: 0.5
    observations:
      - entity_id: "switch.kitchen_lights"
        prob_given_true: 0.6
        prob_given_false: 0.2
        platform: "state"
        to_state: "on"
```

{% configuration %}
prior:
  description: >
     The baseline probability of the event (0 to 1). At any point in time
     (ignoring all external influences) how likely is this event to be occurring?
  required: true
  type: float
probability_threshold:
  description: >
    The posterior probability at which the sensor should trigger to `on`.
    use higher values to reduce false positives (and increase false negatives)
    Note: If the threshold is higher than the `prior` then the default state will be `off`
  required: false
  type: float
  default: 0.5
name:
  description: Name of the sensor to use in the frontend.
  required: false
  type: string
  default: Bayesian Binary Sensor
unique_id:
  description: An ID that uniquely identifies this bayesian entity. If two entities have the same unique ID, Home Assistant will raise an exception.
  required: false
  type: string
device_class:
  description: Sets the [class of the device](/integrations/binary_sensor/), changing the device state and icon that is displayed on the frontend.
  required: false
  type: string
observations:
  description: The observations which should influence the probability that the given event is occurring.
  required: true
  type: list
  keys:
    platform:
      description: >
        The supported platforms are `state`, `numeric_state`, and `template`.
        They are modeled after their corresponding triggers for automations,
        requiring `to_state` (for `state`), `below` and/or `above` (for `numeric_state`) and `value_template` (for `template`).
      required: true
      type: string
    entity_id:
      description: Name of the entity to monitor. Required for `state` and `numeric_state`.
      required: false
      type: string
    to_state:
      description: The entity state that defines the observation. Required (for `state`).
      required: false
      type: string
    value_template:
      description: Defines the template to be used, should evaluate to `True` or `False`. Required for `template`.
      required: false
      type: template
    prob_given_true:
      description: >
        Assuming the bayesian binary_sensor is `True`, the probability the entity state is occurring.
      required: true
      type: float
    prob_given_false:
      description: Assuming the bayesian binary_sensor is `False` the probability the entity state is occurring.
      required: true
      type: float
{% endconfiguration %}

## Estimating probabilities

1. Avoid `0` and `1`, these will mess with the odds and are rarely true - sensors fail.
2. When using `0.99` and `0.001`. The number of `9`s and `0`s matters.
3. Most probabilities will be time-based - the fraction of time something is true is also the probability it will be true.
4. Use your Home Assistant history to help estimate the probabilities.
   - `prob_given_true:` - Select the sensor in question over a time range when you think the `bayesian` sensor should have been `True`. `prob_given_true:` is the fraction of the time the sensor was in `to_state:`.
   - `prob_given_false:` - Select the sensor in question over a time range when you think the `bayesian` sensor should have been `False`. `prob_given_false:` is the fraction of the time the sensor was in `to_state:`.
5. Don't work backwards by tweaking `prob_given_true:` and `prob_given_false:` to give the results and behaviors you want, use #4 to try and get probabilities as close to the 'truth' as you can, if your behavior is not as expected consider adding more sensors or see #6.
6. If your Bayesian sensor ends up triggering `on` too easily, re-check that the probabilities set and estimated make sense, then consider increasing `probability_threshold:` and vice-versa.

## Full examples

These are a number of worked examples which you may find helpful for each of the state types.

### State

The following is an example for the `state` observation platform.

```yaml
# Example configuration.yaml entry
binary_sensor:
  platform: "bayesian"
  name: "in_bed"
  unique_id: "172b6ef1-e37e-4f04-8d64-891e84c02b43" # generated on https://www.uuidgenerator.net/
  prior: 0.25 # I spend 6 hours a day in bed 6hr/24hr is 0.25 
  probability_threshold: 0.8 # I am going to be using this sensor to turn out the lights so I only want to to activate when I am sure
  observations:
    - platform: "state"
      entity_id: "sensor.living_room_motion"
      prob_given_true: 0.05 # If I am in bed then I shouldn't be in the living room, very occasionally I have guests, however
      prob_given_false: 0.2 # My sensor history shows If I am not in bed I spend about a fifth of my time in the living room
      to_state: "on"
    - platform: "state"
      entity_id: "sensor.basement_motion"
      prob_given_true: 0.5 # My sensor history shows, when I am in bed, my basement motion sensor is active about half the time because of my cat
      prob_given_false: 0.3 # As above but my cat tends to spend more time upstairs or outside when I am awake and I rarely use the basement
      to_state: "on"
    - platform: "state"
      entity_id: "sensor.bedroom_motion"
      prob_given_true: 0.5 # My sensor history shows when I am in bed the sensor picks me up about half the time
      prob_given_false: 0.1 # My sensor history shows I spend about 10% of my waking hours in my bedroom
      to_state: "on"
    - platform: "state"
      entity_id: "sun.sun"
      prob_given_true: 0.7 # If I am in bed then there is a good chance the sun will be down, but in the summer mornings I may still be in bed
      prob_given_false: 0.45 # If I am am awake then there is a reasonable chance the sun will be below the horizon - especially in winter
      to_state: "below_horizon"
    - platform: "state"
      entity_id: "sensor.android_charger_type"
      prob_given_true: 0.95 # When I am in bed, I nearly always plug my phone in to charge
      prob_given_false: 0.1 # When I am awake, I occasionally AC charge my phone
      to_state: "ac"
```

### Numeric State

Next up an example which targets the `numeric_state` observation platform,
as seen in the configuration it requires `below` and/or `above` instead of `to_state`.

```yaml
# Example configuration.yaml entry
binary_sensor:
  name: "Heat On"
  platform: "bayesian"
  prior: 0.2
  probability_threshold: 0.9
  observations:
    - platform: "numeric_state"
      entity_id: "sensor.outside_air_temperature_fahrenheit"
      prob_given_true: 0.95
      prob_given_false: 0.05
      below: 50
```

### Template

Here's an example for `template` observation platform, as seen in the configuration it requires `value_template`. This template will evaluate to true if the device tracker `device_tracker.paulus` shows `not_home` and it last changed its status more than 5 minutes ago.

{% raw %}

```yaml
# Example configuration.yaml entry
binary_sensor:
  name: "Paulus Home"
  platform: "bayesian"
  device_class: "presence"
  prior: 0.5
  probability_threshold: 0.9
  observations:
    - platform: template
      value_template: >
        {{is_state('device_tracker.paulus','not_home') and ((as_timestamp(now()) - as_timestamp(states.device_tracker.paulus.last_changed)) > 300)}}
      prob_given_true: 0.05
      prob_given_false: 0.99
```

{% endraw %}

### Multiple state and numeric entries per entity

Lastly, an example illustrates how to configure Bayesian when there are more than two states of interest and several possible numeric ranges. When an entity can hold more than 2 values of interest (numeric ranges or states), then you may wish to specify probabilities for each possible value. Once you have specified more than one, Bayesian cannot infer anything about states or numeric values that are unspecified, like it usually does, so it is recommended that all possible values are included. As above, the `prob_given_true`s of all the possible states should sum to 1, as should the `prob_given_false`s. If a value that has not been specified is observed, then the observation will be ignored as it would be if the entity were `UNKNOWN` or `UNAVAILABLE`.

When more than one range is specified for the same entity, if a value falls on `below`, it will be included with the range that lists it in `below`. `below` then means "below or equal to". This is not true when only a single range is specified, where both `above` and `below` do not include "equal to".

This is an example sensor that can detect if the bins have been left on the side of the road and need to be brought closer to the house. It combines a theoretical presence sensor that gives a numeric signal strength and an API sensor from local government that can have 3 possible states: `due` when collection is due in the next 24 hours, `collected` when collection has happened in the last 24 hours, and `not_due` at other times.

```yaml
# Example configuration.yaml entry
binary_sensor:
  name: "Bins need bringing in"
  platform: "bayesian"
  prior: 0.14 # bins are left out for usually about one day a week
  probability_threshold: 0.5
  observations:
    - platform: "numeric_state"
      entity_id: "sensor.signal_strength"
      prob_given_true: 0.01 # if the bins are out and need bringing in there is only a 1% chance we will get a strong signal of above 10
      prob_given_false: 0.3 # if the bins are not out, we still tend not to get a signal this strong
      above: 10
    - platform: "numeric_state"
      entity_id: "sensor.signal_strength"
      prob_given_true: 0.02
      prob_given_false: 0.5 #if the bins are not out, we often get a signal this strong
      above: 5
      below: 10
    - platform: "numeric_state"
      entity_id: "sensor.signal_strength"
      prob_given_true: 0.07
      prob_given_false: 0.1
      above: 0
      below: 5
    - platform: "numeric_state"
      entity_id: "sensor.signal_strength"
      prob_given_true: 0.3
      prob_given_false: 0.07
      above: -10
      below: 0
    - platform: "numeric_state"
      entity_id: "sensor.signal_strength"
      prob_given_true: 0.6 #if the bins are out, we often get a signal this weak or even weaker
      prob_given_false: 0.03
      below: -10
    # then lets say we want to combine this with an imaginary sensor.bin_collection which reads a local government API that can have one of three values (collected, due, not due)
    - platform: "state"
      entity_id: "sensor.bin_collection"
      prob_given_true: 0.8 # If the bins need bringing in, usually it's because they've just been collected
      prob_given_false: 0.05 # 
      to_state: "collected"
    - platform: "state"
      entity_id: "sensor.bin_collection"
      prob_given_true: 0.05 # If the bins need bringing in, then the sensor.bin_collection shouldn't be 'due'
      prob_given_false: 0.11 # The sensor will be 'due' for about 1 day a week (the 24 hours before collection)
      to_state: "due"
    - platform: "state"
      entity_id: "sensor.bin_collection"
      prob_given_true: 0.15 #All the prob_given_true should add to 1
      prob_given_false: 0.84 # All the prob_given_false should add to 1
      to_state: "not due"
```
