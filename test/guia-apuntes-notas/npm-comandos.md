# 📦 Comandos npm — Referencia Rápida

---

## Instalación de paquetes

```bash
npm install                          # instalar todo el package.json
npm install express                  # dependencia de producción
npm install -D typescript            # dependencia de desarrollo (--save-dev)
npm install -g nodemon               # instalar globalmente
npm install express@4.18.2           # versión específica
npm install express mongoose bcrypt  # varios a la vez
npm install --legacy-peer-deps       # ignorar conflictos de versiones
```

---

## Desinstalar y actualizar

```bash
npm uninstall express                # desinstalar
npm uninstall -D nodemon             # desinstalar de dev
npm update                           # actualizar todos (respeta semver)
npm install express@latest           # actualizar uno a la última versión
npm outdated                         # ver paquetes desactualizados
```

---

## Ejecutar scripts

```bash
npm start                            # ejecuta "start" del package.json
npm run dev                          # ejecuta "dev"
npm test                             # ejecuta "test"
npm run build                        # ejecuta "build"
npm run                              # listar todos los scripts disponibles
```

---

## Información

```bash
npm -v                               # versión de npm
node -v                              # versión de Node.js
npm list --depth=0                   # paquetes instalados (nivel raíz)
npm info express                     # info de un paquete
npm info express versions            # todas las versiones disponibles
npm audit                            # revisar vulnerabilidades
npm audit fix                        # corregir vulnerabilidades
```

---

## Caché y limpieza

```bash
npm cache clean --force              # limpiar caché

# Reinstalar todo desde cero (Linux/macOS)
rm -rf node_modules && rm package-lock.json && npm install

# Reinstalar todo desde cero (Windows PowerShell)
Remove-Item -Recurse -Force node_modules; Remove-Item package-lock.json; npm install
```

---

## npx — ejecutar sin instalar globalmente

```bash
npx ts-node src/index.ts             # ejecutar TypeScript directamente
npx tsc                              # compilar TypeScript
npx tsc --init                       # generar tsconfig.json
npx tsc --noEmit                     # verificar errores sin compilar
npx jest                             # correr tests
npx jest --watch                     # tests en modo watch
npx jest --coverage                  # tests con cobertura
npx nodemon src/index.ts             # servidor con hot reload
npx eslint src/**/*.ts --fix         # linter con autocorrección
npx prettier --write "src/**/*.ts"   # formatear código
```

---

## Recetas listas para copiar

### API REST básica

```bash
npm install express dotenv
npm install -D typescript ts-node nodemon @types/express @types/node
```

### + MongoDB + JWT + bcrypt

```bash
npm install express dotenv mongoose bcrypt jsonwebtoken
npm install -D typescript ts-node nodemon @types/express @types/node @types/bcrypt @types/jsonwebtoken
```

### + Tests (Jest + Supertest)

```bash
npm install express dotenv mongoose bcrypt jsonwebtoken
npm install -D typescript ts-node nodemon @types/express @types/node @types/bcrypt @types/jsonwebtoken jest ts-jest @types/jest supertest @types/supertest mongodb-memory-server
```

---

## Scripts recomendados para package.json

```json
"scripts": {
  "start":          "node dist/index.js",
  "dev":            "nodemon --watch src --exec ts-node src/index.ts",
  "build":          "tsc",
  "test":           "jest --runInBand",
  "test:watch":     "jest --watch",
  "test:coverage":  "jest --coverage"
}
```

---

## Tabla de referencia: ¿qué instalar para qué?

| Necesito...                  | Comando                                                                 |
|------------------------------|-------------------------------------------------------------------------|
| Servidor HTTP                | `npm i express` + `npm i -D @types/express`                             |
| MongoDB                      | `npm i mongoose`                                                        |
| Variables de entorno         | `npm i dotenv`                                                          |
| Hashear contraseñas          | `npm i bcrypt` + `npm i -D @types/bcrypt`                               |
| Tokens JWT                   | `npm i jsonwebtoken` + `npm i -D @types/jsonwebtoken`                   |
| CORS                         | `npm i cors` + `npm i -D @types/cors`                                   |
| Validación (moderna)         | `npm i zod`                                                             |
| Validación (con Express)     | `npm i express-validator`                                               |
| Subida de archivos           | `npm i multer` + `npm i -D @types/multer`                               |
| Peticiones HTTP externas     | `npm i axios`                                                           |
| TypeScript                   | `npm i -D typescript ts-node @types/node`                               |
| Reinicio automático          | `npm i -D nodemon`                                                      |
| Tests                        | `npm i -D jest ts-jest @types/jest`                                     |
| Tests de endpoints           | `npm i -D supertest @types/supertest`                                   |
| MongoDB en memoria (tests)   | `npm i -D mongodb-memory-server`                                        |
