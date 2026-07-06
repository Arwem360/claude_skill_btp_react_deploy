# btp-react-deploy

Skill para **[Claude Code](https://claude.com/claude-code)** que guía la construcción y el despliegue de apps **React (Vite + Tailwind v4)** en **SAP BTP Cloud Foundry** vía **MTA**, servidas desde el **HTML5 Application Repository** y embebibles en **SAP Work Zone**.

## ¿Qué resuelve?

Despliegues de React en BTP se traban por detalles de configuración que hay que dejar bien **desde el primer commit**, no parchear al final. Este skill documenta el patrón completo:

- Estructura del `mta.yaml`, `xs-app.json` y `manifest.json` minimal.
- Empaquetado correcto del zip para el HTML5 Apps Repo (evita el típico `CODE 1001` del upload).
- Approuter managed + embed en Work Zone (reemplaza el patrón "wz-stub" de UI5 + iframe).
- Comandos `cf`/`mbt`, destinations y XSUAA, y las validaciones previas al deploy.

## Genérico / reutilizable

El `SKILL.md` no tiene datos de ningún cliente. Al principio hay una tabla de **placeholders** (`<CF_API>`, `<CF_ORG>`, `<CF_SPACE>`, `<REGION_HOST>`, `<APP_DIR>`, `<MTA_ID>`, `<SAP_CLOUD_SERVICE>`, `<WORKZONE_HOST>`) para completar con los valores de tu subaccount/proyecto.

## Instalación

Copiá la carpeta del skill dentro de tu proyecto (o de tu config global de Claude Code):

```bash
# Por proyecto
mkdir -p .claude/skills/btp-react-deploy
curl -sL https://raw.githubusercontent.com/Arwem360/claude_skill_btp_react_deploy/main/SKILL.md \
  -o .claude/skills/btp-react-deploy/SKILL.md

# O global (aplica a todos tus proyectos)
mkdir -p ~/.claude/skills/btp-react-deploy
# ...misma descarga a esa ruta
```

Claude Code detecta el skill por su front-matter (`name` + `description`) y lo activa solo cuando la tarea lo amerita (deploy a BTP, `mta.yaml`, `cf deploy`, Work Zone, etc.).

## Uso

Una vez instalado, pedile a Claude Code cosas como *"desplegá esta app React a BTP"* o *"prepará esta app para Work Zone"* y el skill se aplica automáticamente.

## Relacionado

- [`deploy-cpanel`](https://github.com/Arwem360/claude_skill_deploy_cpanel) — deploy de React + Node a hosting cPanel por FTP.
