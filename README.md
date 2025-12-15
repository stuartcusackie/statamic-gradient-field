# Statamic Gradient Field

> Gradient Field is a Statamic addon that enables users to create linear gradients.

## Features

- Multiple gradient colour stop selection.
- Colour transparency selection.
- Gradient angle selection.
- Gradient presets.

## How to Install

You can install this addon via Composer:

``` bash
composer require stuartcusackie/statamic-gradient-field
```

## How to Use

1. Install the field via composer.
2. Add the field to your blueprints.
3. Update the field configuration options for presets and disabling custom gradients.
4. Use the generated CSS in your templates. Example below.

```
<div class="w-8 h-8" style="background: {{ gradientField }};"></div>
```
