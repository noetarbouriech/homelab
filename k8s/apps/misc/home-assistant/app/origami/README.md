# Origami Full assets

Vendored from [hazymorning/origami](https://github.com/hazymorning/origami) at
release **V2.0** (commit `76f4fcd`, MIT license):

- `theme.yaml` — upstream `themes/origami.yaml`
- `modules/origami_*.yaml` — upstream `bubble_card/modules/` (the modules used by
  the dashboard)
- `dashboard.yaml` — Full layout adapted to this home (header/menu, greeting,
  weather, favorites, rooms + climate table, House, navbar, pop-ups)
- `packages.yaml` — the helpers the Full layout reads (`sensor.climate_*`,
  `sensor.greetings`, `sensor.lights_on_count`, `light.all_lights`,
  `input_boolean.room_climate_card`, `script.thermostat_step`)

The layout also uses the custom cards installed in `/config/www/community/`:
Paper Buttons Row, Navbar Card, Origami Weather, Origami Notifications and
Origami Header (via `frontend.extra_module_url`). Bubble Card is the existing
frontend module; the Montserrat font is under `/config/www/fonts/`.
