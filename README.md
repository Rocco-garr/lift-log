# Lift Log

Rocco's workout logger, served by GitHub Pages and installable to the iPhone home screen.

- `plan.json` holds the 5 workout templates, the bench progression and the coach note. Edit it to change the plan; the app picks it up next time it's online.
- Workouts are stored on the phone and backed up to the private repo `lift-log-data` (`sessions/<id>.json`).
- When changing `index.html` or `sw.js`, bump `VERSION` in `sw.js` so phones get the update banner.
