# Retry

[![HACS Badge](https://img.shields.io/badge/HACS-Default-31A9F4.svg?style=for-the-badge)](https://github.com/hacs/integration)

[![GitHub Release](https://img.shields.io/github/release/amitfin/retry.svg?style=for-the-badge&color=blue)](https://github.com/amitfin/retry/releases)

![Download](https://img.shields.io/github/downloads/amitfin/retry/total.svg?style=for-the-badge&color=blue) ![Analytics](https://img.shields.io/badge/dynamic/json?style=for-the-badge&color=blue&label=Analytics&suffix=%20Installs&cacheSeconds=15600&url=https://analytics.home-assistant.io/custom_integrations.json&query=$.retry.total)

![Project Maintenance](https://img.shields.io/badge/maintainer-Amit%20Finkelstein-blue.svg?style=for-the-badge)

Smart homes include a network of devices. A case of a failed command can happen due to temporary connectivity issues or invalid device states. The cost of such a failure can be high, especially for background automation. For example, failing to shutdown an irrigation system which should run for 20 minutes can have severe consequences.

The integration increases the automation reliability by implementing 2 custom services - `retry.actions` and `retry.call`.
`retry.actions` is the recommended and UI friendly service which should be used. `retry.call` is the engine behind the scene. It's being called by `retry.actions` but can also be called directly by advanced users.

## `retry.actions`

Here is a short demo of using `retry.actions` in the automation rule editor:

https://github.com/amitfin/retry/assets/19599059/318c2129-901f-4f6c-8e79-e155ae097ba4

`retry.actions` wraps any service call inside the sequence of actions with `retry.call`. `retry.call` calls the original service with a background retry logic on failures. A complex sequence of actions with a nested structure and conditions is supported. The service traverses through the actions and wraps any service call. There is no impact or changes to the rest of the actions. The detailed behavior and the list of optional parameters of `retry.call` is explained in the section below. All features and parameters of `retry.call` are also supported by `retry.actions`, so there is no reason to use the YAML configuration of `retry.call`. A straightforward UI usage as demonstrated above should be the way to go.

Note: `retry.actions` and `retry.call` are not suitable for the following scenarios:

1. When the order of the actions matters: the background retries are running independently to the rest of the actions.
2. For a relative state change: for example, `homeassistant.toggle` and `fan.increase_speed` are relatives operations while `light.turn_on` is an absolute one. The reason is that a relative service call might change the state and only then a failure occurs. Calling it again might have an unintentional result.
3. If any [service call response data](https://www.home-assistant.io/docs/scripts/service-calls/#use-templates-to-handle-response-data) is needed: the service calls are running in the background and therefore it's not possible to propagate responses.

## `retry.call`

This service warps an inner service call with background retries on failures. It can be useful to mitigate temporary issues of connectivity or invalid device states.

For example, instead of:

```
service: homeassistant.turn_on
target:
  entity_id: light.kitchen
```

The following should be used:

```
service: retry.call
data:
  service: homeassistant.turn_on
target:
  entity_id: light.kitchen
```

It's possible to add additional parameters to the `data` section. The extra parameters will be passed to the inner service call.

The inner service call will get called again if one of the following happens:

1. The inner service call raises an exception.
2. The target entity is unavailable. Note that this is important since HA silently skips unavailable entities ([here](https://github.com/home-assistant/core/blob/580b20b0a83c561986e7571b83df4a4bcb158392/homeassistant/helpers/service.py#L763)).

Here is the list of parameters to control the behavior of `retry.call` (directly or via `retry.actions`). These parameters are not passed to the inner service call and are consumed only by the `retry.call` service itself.

#### `service` parameter (mandatory)

The `service` parameter is the only mandatory parameter. It contains the name of the inner service. It supports templates.

#### `retries` parameter (optional)

Controls the amount of retries. The default value is 7. For example:

```
service: retry.call
data:
  service: homeassistant.turn_on
  retries: 10
target:
  entity_id: light.kitchen
```

#### `backoff` parameter (optional)

The amount of seconds to wait between attempts. It's expressed in a special template format with square brackets `"[[ ... ]]"` instead of curly brackets `"{{ ... }}"`. This is needed to prevent from rendering the expression in advance. `attempt` is provided as a variable, holding a zero-based counter - it's zero for 1st time the expression is evaluated, and increasing by one for subsequence evaluations. Note that there is no delay for the initial attempt, so the list of delays always starts with a zero.

The default value is `"[[ 2 ** attempt ]]"` which is an exponential backoff. These are the delay times of the first 7 attempts: [0, 1, 2, 4, 8, 16, 32] (each delay is twice than the previous one). The following are the offsets from the initial call [0, 1, 3, 7, 15, 31, 63].

Linear backoff is a different strategy which can be expressed as a simple non-template string (without brackets). For example, these are the delay times of the first 7 attempts when using `"10"`: [0, 10, 10, 10, 10, 10, 10]. The following are the offsets from the initial call [0, 10, 20, 30, 40, 50, 60].

Another example is `"[[ 10 * 2 ** attempt ]]"` which is a slower exponential backoff. These are the delay times of the first 7 attempts: [0, 10, 20, 40, 80, 160, 320]. The following are the offsets from the initial call [0, 10, 30, 70, 150, 310, 630].

```
service: retry.call
data:
  service: homeassistant.turn_on
  backoff: "[[ 10 * 2 ** attempt ]]"
target:
  entity_id: light.kitchen
```

#### `expected_state` parameter (optional)

Validation of the entity's state after the inner service call. For example:

```
service: retry.call
data:
  service: homeassistant.turn_on
  expected_state: "on"
target:
  entity_id: light.kitchen
```

If the new state is different than expected, the attempt is considered a failure and the loop of retries continues. The `expected_state` parameter can be a list and it supports templates.

#### `validation` parameter (optional)

A boolean expression of a special template format with square brackets `"[[ ... ]]"` instead of curly brackets `"{{ ... }}"`. This is needed to prevent from rendering the expression in advance. `entity_id` is provided as a variable. For example:

```
service: retry.call
data:
  service: light.turn_on
  brightness: 70
  validation: "[[ state_attr(entity_id, 'brightness') == 70 ]]"
target:
  entity_id: light.kitchen
```

The boolean expression is rendered after each call to the inner service. If the value is False, the attempt is considered a failure and the loop of retries continues.

Note: `validation: "[[ states(entity_id) == 'on' ]]"` has an identical logic and impact as setting `expected_state: "on"`. Therefore, the later is preferable from simplicity reasons.

#### `state_delay` parameter (optional)

Controls the amount of seconds to wait before the initial check of `expected_state` and `validation` (has no impact if both are absent). The default value is zero (no delay). This option can be used if the new state is being updated immediately but later on getting reverted to the previous state since the operation failed on the remote device. It's not the common behavior as integrations should update the state only when getting a new state from the device. Here is a configuration example:

```
service: retry.call
data:
  service: light.turn_off
  state_delay: 2
  expected_state: "off"
target:
  entity_id: light.kitchen
```

#### `state_grace` parameter (optional)

Controls the amount of seconds to wait before the final check of `expected_state` and `validation` (has no impact if both are absent). The default value is 0.2 seconds. The 2nd (final) check is done if the initial check fails. The service call attempt is considered a failure only if the 2nd check fails. Here is a configuration example:

```
service: retry.call
data:
  service: light.turn_off
  transition: 5
  state_grace: 5.1
  expected_state: "off"
target:
  entity_id: light.kitchen
```

#### `on_error` parameter (optional)

A sequence of actions to execute if all retries fail.

Here is an automation rule example with a self remediation logic:

```
alias: Kitchen Evening Lights
mode: parallel
trigger:
  - platform: sun
    event: sunset
action:
  - service: retry.call
    data:
      service: light.turn_on
      entity_id: light.kitchen_light
      retries: 2
      on_error:
        - service: homeassistant.reload_config_entry
          data:
            entry_id: "{{ config_entry_id(entity_id) }}"
        - delay:
            seconds: 20
        - service: automation.trigger
          target:
            entity_id: automation.kitchen_evening_lights
```

(This example can be configured in UI mode by using `retry.actions`. YAML is not needed.)

`entity_id` is provided as a variable and can be used by `on_error` templates.

Note that each entity is running individually when the inner service call has a list of entities. In such a case `on_error` can get executed multiple times, once per each failed entity. Similarly, `retry.actions` has a sequence of actions which might include multiple service calls. This can also trigger multiple execution of `on_error`, once per each failed inner service call.

#### `retry_id` parameter (optional)

A service call cancels a previous running call with the same retry ID. This parameter can be used to set the retry ID explicitly but it should be rarely used, if at all. The default value of `retry_id` is the `entity_id` of the inner service call. For inner service calls with no `entity_id`, the default value of `retry_id` is the service name.

An example of the cancellation scenario might be when turning off a light while the turn on retry loop of the same light is still running due to failures or light's transition time. The turn on retry loop will be getting canceled by the turn off call since both share the same `retry_id` by default (the entity ID).

Note that each entity is running individually when the inner service call has a list of entities. Therefore, they have a different default `retry_id`. However, an explicit `retry_id` is shared for all entities of the same retry call. Nevertheless, retry loops created by the same service call (`retry.call` or `retry.actions`) are not canceling each other even when they share the same `retry_id`.

It's possible to disable the cancellation logic by setting `retry_id` to an empty string (`retry_id: ""`) or null (`retry_id: null`). In such a case, the service call doesn't cancel any other running call and will not be canceled by any other future call. Note that it's not possible to set `retry_id` to an empty string or null via the "UI Mode" but instead the "YAML Mode" in the UI should be used.

### Notes

1. The service does not propagate inner service failures (exceptions) since the attempts are done in the background. However, the service logs a warning when the inner function fails (on every attempt). It also logs an error and issue a repair ticket when the maximum amount of attempts is reached. Repair tickets can be disabled via the [integration's configuration dialog](https://my.home-assistant.io/redirect/integration/?domain=retry).
2. Service calls support a list of entities either by providing an explicit list or by [targeting areas and devices](https://www.home-assistant.io/docs/scripts/service-calls/#targeting-areas-and-devices). It's also possible to specify a [group](https://www.home-assistant.io/integrations/group) entity. The call to the inner service is done individually per entity to isolate failures. Group entities are expanded (recursively.)

## Install

HACS is the preferred and easier way to install the component, and can be done by using this My button:

[![Open your Home Assistant instance and open a repository inside the Home Assistant Community Store.](https://my.home-assistant.io/badges/hacs_repository.svg)](https://my.home-assistant.io/redirect/hacs_repository/?owner=amitfin&repository=retry&category=integration)

Otherwise, download `retry.zip` from the [latest release](https://github.com/amitfin/retry/releases), extract and copy the content under `custom_components` directory.

Home Assistant restart is required once the integration files are copied (either by HACS or manually).

The Retry integration should also be added to the configuration in order to use the new custom service. This can be done via the user interface, by using this My button:

[![Open your Home Assistant instance and start setting up a new integration.](https://my.home-assistant.io/badges/config_flow_start.svg)](https://my.home-assistant.io/redirect/config_flow_start/?domain=retry)

It's also possible to add the integration via `configuration.yaml` by adding the single line `retry:`.

## Contributions are welcome!

If you want to contribute to this please read the [Contribution guidelines](CONTRIBUTING.md)

[Link to post in Home Assistant's community forum](https://community.home-assistant.io/t/improving-automation-reliability/558627)
