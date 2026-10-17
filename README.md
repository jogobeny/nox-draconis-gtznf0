NeoForge 21.1.252 · Minecraft 1.21.1 · [packwiz](https://packwiz.infra.link)

## MultiMC / UltimMC setup

1. **Add Instance** → Minecraft `1.21.1`.
2. **Edit Instance** → **Version** → **Install NeoForge** → `21.1.252`.
3. Download [packwiz-installer-bootstrap.jar](https://github.com/packwiz/packwiz-installer-bootstrap/releases/latest/download/packwiz-installer-bootstrap.jar) into the instance's `.minecraft` folder, next to `mods` (**Edit Instance** → bottom right **Open .minecraft**).
4. **Edit Instance** → **Settings** → **Custom commands** → tick it and set **Pre-launch command**:
   ```
   "$INST_JAVA" -jar packwiz-installer-bootstrap.jar https://raw.githubusercontent.com/jogobeny/nox-draconis-gtznf0/main/pack.toml
   ```
5. **Launch.** Mods sync on every start.
