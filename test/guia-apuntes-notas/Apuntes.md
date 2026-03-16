# 📚 README — Guía de Referencia: Node.js + TypeScript + Express + MongoDB



---

## Tabla de contenido

1. [Arquitectura típica del proyecto](#1-arquitectura-típica-del-proyecto)
2. [Modelos con Mongoose](#2-modelos-con-mongoose)
3. [Services — lógica de negocio](#3-services--lógica-de-negocio)
4. [Controllers — manejo de requests](#4-controllers--manejo-de-requests)
5. [Rutas con Express](#5-rutas-con-express)
6. [Middleware de autenticación JWT](#6-middleware-de-autenticación-jwt)
7. [Validación de entradas](#7-validación-de-entradas)
8. [Conexión a MongoDB](#8-conexión-a-mongodb)
9. [Punto de entrada — index.ts](#9-punto-de-entrada--indexts)
10. [Errores comunes a detectar](#10-errores-comunes-a-detectar)
11. [Tests con Jest — Cheatsheet](#11-tests-con-jest--cheatsheet)
12. [Rutas documentadas](#12-rutas-documentadas)

---

## 1. Arquitectura típica del proyecto

```
src/
├── config/
│   └── db.ts              → Conexión a MongoDB (mongoose.connect)
├── controllers/
│   └── entidad.controller.ts  → Recibe req/res, llama al service
├── middlewares/
│   └── auth.ts            → Verifica JWT antes de rutas protegidas
├── models/
│   └── entidad.ts         → Schema de Mongoose + interfaces TypeScript
├── routes/
│   └── index.ts           → Define todas las rutas y sus middlewares
├── schemas/
│   └── entidad.schema.ts  → Validación de campos de entrada
├── services/
│   └── entidad.service.ts → Lógica de negocio, accede al modelo
└── index.ts               → App Express, un solo app.listen()

test/
├── unit/
│   └── entidad.service.test.ts    → Mockean el modelo, prueban el service
└── integration/
    └── entidad.controller.test.ts → Mockean el service, prueban el controller
```

**Flujo de una petición:**
```
Cliente → Ruta → [Middleware auth] → [Middleware validación] → Controller → Service → Modelo → MongoDB
```

---

## 2. Modelos con Mongoose

```typescript
import mongoose, { Types } from 'mongoose';

// 1. Interface de entrada (campos que envía el cliente)
export interface EntidadInput {
  nombre: string;
  email: string;
  activo: boolean;
  referenciaId?: Types.ObjectId; // opcional, referencia a otro modelo
}

// 2. Interface del documento (lo que devuelve MongoDB, extiende la entrada)
export interface EntidadDocument extends EntidadInput, mongoose.Document {
  createdAt: Date;
  updatedAt: Date;
}

// 3. Schema de Mongoose
const entidadSchema = new mongoose.Schema({
  nombre:       { type: String,  required: true },
  email:        { type: String,  required: true, unique: true, index: true },
  activo:       { type: Boolean, required: true, default: true },
  referenciaId: { type: mongoose.Schema.Types.ObjectId, ref: 'OtroModelo', required: false }
}, {
  timestamps: true,       // agrega createdAt y updatedAt automáticamente
  collection: 'entidades' // nombre exacto de la colección en MongoDB
});

// 4. Exportar el modelo
const Entidad = mongoose.model<EntidadDocument>('Entidad', entidadSchema);
export default Entidad;
```

**Tipos de campo más usados:**

| Tipo          | Mongoose                          | Notas                          |
|---------------|-----------------------------------|--------------------------------|
| Texto         | `type: String`                    |                                |
| Número        | `type: Number`                    |                                |
| Booleano      | `type: Boolean, default: false`   |                                |
| Fecha         | `type: Date, default: Date.now`   |                                |
| Referencia    | `type: Schema.Types.ObjectId, ref: 'Modelo'` | Para populate       |
| Arreglo       | `type: [String]`                  | O `[{ type: ObjectId }]`      |

---

## 3. Services — lógica de negocio

El service es la clase que contiene **toda** la lógica. Nunca accede directamente a `req`/`res`.

```typescript
import EntidadModel, { EntidadInput, EntidadDocument } from '../models/entidad';

class EntidadService {

  // Obtener todos
  async getAll(): Promise<EntidadDocument[]> {
    try {
      return await EntidadModel.find();
    } catch (error) {
      throw new Error('Error al obtener registros');
    }
  }

  // Obtener por ID
  async findById(id: string): Promise<EntidadDocument | null> {
    try {
      return await EntidadModel.findById(id);
    } catch (error) {
      throw new Error('Error al buscar por ID');
    }
  }

  // Buscar por campo (ej. email)
  async findByEmail(email: string): Promise<EntidadDocument | null> {
    try {
      return await EntidadModel.findOne({ email });
    } catch (error) {
      throw new Error('Error al buscar por email');
    }
  }

  // Buscar por campo de texto (insensible a mayúsculas)
  async findByName(nombre: string): Promise<EntidadDocument[]> {
    try {
      return await EntidadModel.find({
        nombre: { $regex: new RegExp(nombre, 'i') }
      });
    } catch (error) {
      throw new Error('Error al buscar por nombre');
    }
  }

  // Crear
  async create(input: EntidadInput): Promise<EntidadDocument> {
    try {
      return await EntidadModel.create(input);
    } catch (error) {
      throw new Error('Error al crear registro');
    }
  }

  // Actualizar (devuelve el documento ACTUALIZADO con { new: true })
  async update(id: string, input: Partial<EntidadInput>): Promise<EntidadDocument | null> {
    try {
      return await EntidadModel.findOneAndUpdate(
        { _id: id },
        input,
        { new: true }          // ← devuelve el doc después del update
        // { returnOriginal: false } es equivalente en versiones antiguas
      );
    } catch (error) {
      throw new Error('Error al actualizar');
    }
  }

  // Eliminar (devuelve true/false según si encontró el doc)
  async remove(id: string): Promise<boolean> {
    try {
      const result = await EntidadModel.findOneAndDelete({ _id: id });
      return result !== null;
    } catch (error) {
      throw new Error('Error al eliminar');
    }
  }

  // Filtrar por campo booleano
  async getActivos(): Promise<EntidadDocument[]> {
    try {
      return await EntidadModel.find({ activo: true });
    } catch (error) {
      throw new Error('Error al obtener activos');
    }
  }
}

export default new EntidadService();
```

---

## 4. Controllers — manejo de requests

El controller **recibe** `req`/`res`, **delega** al service y **responde** al cliente.

```typescript
import { Request, Response } from 'express';
import entidadService from '../services/entidad.service';
import { EntidadInput } from '../models/entidad';

class EntidadController {

  // GET /entidades
  public async getAll(req: Request, res: Response): Promise<void> {
    try {
      const items = await entidadService.getAll();
      res.status(200).json(items);
    } catch (error) {
      res.status(500).json({ message: 'Error al obtener registros' });
    }
  }

  // GET /entidades/:id
  public async getOne(req: Request, res: Response): Promise<void> {
    try {
      const item = await entidadService.findById(req.params.id ?? '');
      if (!item) {
        res.status(404).json({ message: 'Registro no encontrado' });
        return;
      }
      res.status(200).json(item);
    } catch (error) {
      res.status(500).json({ message: 'Error al obtener registro' });
    }
  }

  // POST /entidades
  public async create(req: Request, res: Response): Promise<void> {
    try {
      const item = await entidadService.create(req.body as EntidadInput);
      res.status(201).json(item);
    } catch (error) {
      res.status(500).json({ message: 'Error al crear registro' });
    }
  }

  // PUT /entidades/:id
  public async update(req: Request, res: Response): Promise<void> {
    try {
      const existe = await entidadService.findById(req.params.id ?? '');
      if (!existe) {
        res.status(404).json({ message: 'Registro no encontrado' });
        return;
      }
      const actualizado = await entidadService.update(req.params.id ?? '', req.body);
      res.status(200).json(actualizado);
    } catch (error) {
      res.status(500).json({ message: 'Error al actualizar' });
    }
  }

  // DELETE /entidades/:id
  public async remove(req: Request, res: Response): Promise<void> {
    try {
      const success = await entidadService.remove(req.params.id ?? '');
      if (!success) {
        res.status(404).json({ message: 'Registro no encontrado' });
        return;
      }
      res.status(204).send();
    } catch (error) {
      res.status(500).json({ message: 'Error al eliminar' });
    }
  }
}

export default new EntidadController();
```

**Códigos de respuesta HTTP más usados:**

| Código | Significado              | Cuándo usarlo                           |
|--------|--------------------------|-----------------------------------------|
| 200    | OK                       | GET, PUT exitosos                       |
| 201    | Created                  | POST exitoso                            |
| 204    | No Content               | DELETE exitoso (sin body)               |
| 400    | Bad Request              | Validación fallida, datos inválidos     |
| 401    | Unauthorized             | Sin token o token inválido              |
| 403    | Forbidden                | Sin permisos                            |
| 404    | Not Found                | Recurso no existe                       |
| 500    | Internal Server Error    | Error inesperado del servidor           |

---

## 5. Rutas con Express

```typescript
import { Express } from 'express';
import entidadController from '../controllers/entidad.controller';
import auth from '../middlewares/auth';
import { validateEntidadSchema } from '../schemas/entidad.schema';

const routes = (app: Express) => {

  // Rutas públicas (sin auth)
  app.post('/login', authController.login);
  app.post('/entidades', validateEntidadSchema, entidadController.create);

  // Rutas protegidas (con auth)
  app.get('/entidades',          auth, entidadController.getAll);
  app.get('/entidades/:id',      auth, entidadController.getOne);
  app.put('/entidades/:id',      auth, entidadController.update);
  app.delete('/entidades/:id',   auth, entidadController.remove);

  // Rutas con parámetro especial
  // ⚠️ IMPORTANTE: rutas específicas ANTES de rutas genéricas con :id
  app.get('/entidades/activos',      auth, entidadController.getActivos);  // ← primero
  app.get('/entidades/:id',          auth, entidadController.getOne);       // ← después
};

export default routes;
```

> **Regla crítica de orden:** Si tienes `/entidades/activos` y `/entidades/:id`, la ruta estática (`/activos`) DEBE ir antes. Si `:id` va primero, Express interpretará `activos` como un ID.

---

## 6. Middleware de autenticación JWT

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

const auth = async (req: Request, res: Response, next: NextFunction) => {
  try {
    let token = req.headers.authorization;

    if (!token) {
      return res.status(401).json({ message: 'Not authorized' });
    }

    // El header viene como "Bearer <token>" → quitamos el prefijo
    token = token.replace('Bearer ', '');

    const decoded = jwt.verify(token, process.env.JWT_SECRET || 'secret');

    // Adjuntamos el usuario decodificado al body o a un campo custom
    req.body.loggedUser = decoded;

    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized' });
  }
};

export default auth;
```

**Generar un token (en el service de login):**

```typescript
import jwt from 'jsonwebtoken';

login(email: string): string {
  const secret = process.env.JWT_SECRET || 'secret';
  const token = jwt.sign(
    { email },                  // payload (datos que queremos guardar)
    secret,
    { expiresIn: '1d' }         // expira en 1 día
  );
  return token;
}
```

---

## 7. Validación de entradas

Middleware de validación manual (sin librerías externas):

```typescript
import { Request, Response, NextFunction } from 'express';

export const validateEntidadSchema = (
  req: Request,
  res: Response,
  next: NextFunction
): void => {
  const { nombre, email, password } = req.body;
  const errors: string[] = [];

  // Validar nombre
  if (!nombre || typeof nombre !== 'string' || nombre.trim() === '') {
    errors.push("El campo 'nombre' es requerido y debe ser texto.");
  }

  // Validar email
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!email || !emailRegex.test(email)) {
    errors.push("El campo 'email' es requerido y debe ser un email válido.");
  }

  // Validar password (mínimo 8 caracteres)
  if (!password || typeof password !== 'string' || password.length < 8) {
    errors.push("El campo 'password' debe tener al menos 8 caracteres.");
  }

  if (errors.length > 0) {
    res.status(400).json({ errors });
    return; // ← importante: no llamar a next() si hay errores
  }

  next();
};
```

---

## 8. Conexión a MongoDB

```typescript
import mongoose from 'mongoose';
import dotenv from 'dotenv';

dotenv.config();

const connectionString =
  process.env.MONGO_URL || 'mongodb://localhost:27017/miapp';

export const db = mongoose
  .connect(connectionString)
  .then(() => console.log('Conectado a MongoDB'))
  .catch((err) => console.error('Error al conectar:', err));
```

---

## 9. Punto de entrada — index.ts

```typescript
import express from 'express';
import dotenv from 'dotenv';
import routes from './routes';
import { db } from './config/db';

dotenv.config();

const app = express();

// Middlewares globales
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Registrar rutas
routes(app);

const port = process.env.PORT || 3000;

// ✅ UN SOLO app.listen, dentro del .then() de la BD
db.then(() => {
  app.listen(port, () => {
    console.log(`Servidor corriendo en http://localhost:${port}`);
  });
});

export default app; // necesario para los tests de integración con supertest
```

> **Error más común:** Llamar `app.listen()` dos veces (una directamente y otra en el `.then()` de la BD). Causa error `EADDRINUSE` — el puerto ya está en uso.

---

## 10. Errores comunes a detectar

Esta sección cubre los bugs típicos que se inyectan en los parciales.

### Error 1 — Doble `app.listen()`
```typescript
// ❌ MAL: dos llamadas a listen
app.listen(3000, () => console.log('corriendo'));  // primera vez
db.then(() => app.listen(3000, () => console.log('ok'))); // segunda vez → CRASH

// ✅ BIEN: solo una llamada, dentro del then
db.then(() => app.listen(3000, () => console.log('ok')));
```

### Error 2 — Orden incorrecto: hashear antes de verificar existencia
```typescript
// ❌ MAL: hashea aunque el usuario ya exista (ineficiente y contamina req.body)
const existe = await userService.findByEmail(req.body.email);
req.body.password = await bcrypt.hash(req.body.password, 10); // ← hash innecesario
if (existe) return res.status(400).json({ message: 'Ya existe' });

// ✅ BIEN: verificar primero, hashear solo si es necesario
const existe = await userService.findByEmail(req.body.email);
if (existe) return res.status(400).json({ message: 'Ya existe' });
req.body.password = await bcrypt.hash(req.body.password, 10); // ← solo si no existe
```

### Error 3 — Service con clase vacía
```typescript
// ❌ MAL: clase sin métodos → todos los controllers fallan en runtime
class EntidadService {}
export default new EntidadService();

// ✅ BIEN: implementar todos los métodos que usa el controller
class EntidadService {
  async getAll() { return await EntidadModel.find(); }
  // ...resto de métodos
}
```

### Error 4 — Controller sin método que la ruta referencia
```typescript
// routes/index.ts
app.post('/entidades', auth, entidadController.create); // ← llama a .create

// controllers/entidad.controller.ts
class EntidadController {
  // ❌ MAL: no existe el método create → TypeError en runtime
  async getAll() { ... }
}

// ✅ BIEN: agregar el método faltante
class EntidadController {
  async create(req: Request, res: Response) { ... } // ← agregar
  async getAll(req: Request, res: Response) { ... }
}
```

### Error 5 — Imports innecesarios o incorrectos
```typescript
// ❌ MAL: imports que no se usan → warnings de TypeScript
import { error } from 'console';  // nunca se usa
import { Timestamp } from 'mongodb'; // no se usa

// ✅ BIEN: solo importar lo que se necesita
import { Request, Response, NextFunction } from 'express';
```

### Error 6 — findOneAndUpdate sin `{ new: true }`
```typescript
// ❌ MAL: devuelve el documento ANTES del update
const updated = await Model.findOneAndUpdate({ _id: id }, data);

// ✅ BIEN: devuelve el documento DESPUÉS del update
const updated = await Model.findOneAndUpdate({ _id: id }, data, { new: true });
```

### Error 7 — Ruta genérica antes de ruta específica
```typescript
// ❌ MAL: Express interpreta "vendidos" como un :id
app.get('/libros/:id',      controller.getOne);    // atrapa todo
app.get('/libros/vendidos', controller.getSold);   // nunca se alcanza

// ✅ BIEN: específica primero
app.get('/libros/vendidos', controller.getSold);   // primero
app.get('/libros/:id',      controller.getOne);    // después
```

---

## 11. Tests con Jest — Cheatsheet

### Estructura de un test unitario (mockea el modelo)

```typescript
import entidadService from '../../src/services/entidad.service';
import EntidadModel from '../../src/models/entidad';

jest.mock('../../src/models/entidad'); // ← mockea todo el modelo

describe('EntidadService', () => {
  afterEach(() => jest.clearAllMocks()); // limpia mocks entre tests

  it('debe obtener todos los registros', async () => {
    // 1. Preparar el mock
    (EntidadModel.find as jest.Mock).mockResolvedValue([{ nombre: 'Test' }]);

    // 2. Llamar al método
    const result = await entidadService.getAll();

    // 3. Verificar
    expect(EntidadModel.find).toHaveBeenCalled();
    expect(result).toHaveLength(1);
    expect(result[0].nombre).toBe('Test');
  });
});
```

### Estructura de un test de integración (mockea el service)

```typescript
import { Request, Response } from 'express';
import entidadController from '../../src/controllers/entidad.controller';
import entidadService from '../../src/services/entidad.service';

jest.mock('../../src/services/entidad.service'); // ← mockea el service

// Helper para crear un Response falso
const mockResponse = () => {
  const res = {} as Response;
  res.status = jest.fn().mockReturnValue(res);
  res.json   = jest.fn().mockReturnValue(res);
  res.send   = jest.fn().mockReturnValue(res);
  return res;
};

describe('EntidadController', () => {
  beforeEach(() => jest.clearAllMocks());

  it('debe retornar 200 con la lista', async () => {
    const req = {} as Request;
    const res = mockResponse();

    (entidadService.getAll as jest.Mock).mockResolvedValue([{ id: '1' }]);

    await entidadController.getAll(req, res);

    expect(res.status).toHaveBeenCalledWith(200);
    expect(res.json).toHaveBeenCalledWith([{ id: '1' }]);
  });

  it('debe retornar 404 si no existe', async () => {
    const req = { params: { id: '1' } } as unknown as Request;
    const res = mockResponse();

    (entidadService.findById as jest.Mock).mockResolvedValue(null);

    await entidadController.getOne(req, res);

    expect(res.status).toHaveBeenCalledWith(404);
    expect(res.json).toHaveBeenCalledWith({ message: 'Registro no encontrado' });
  });
});
```

### Métodos de Mongoose más mockeados

| Llamada en el service          | Cómo mockearla en Jest                                           |
|-------------------------------|------------------------------------------------------------------|
| `Model.find()`                | `(Model.find as jest.Mock).mockResolvedValue([...])`             |
| `Model.findById(id)`          | `(Model.findById as jest.Mock).mockResolvedValue({...})`         |
| `Model.findOne({ campo })`    | `(Model.findOne as jest.Mock).mockResolvedValue({...})`          |
| `Model.create(data)`          | `(Model.create as jest.Mock).mockResolvedValue({...})`           |
| `Model.findOneAndUpdate()`    | `(Model.findOneAndUpdate as jest.Mock).mockResolvedValue({...})` |
| `Model.findOneAndDelete()`    | `(Model.findOneAndDelete as jest.Mock).mockResolvedValue({...})` |

### Matchers más usados

```typescript
expect(valor).toBe('exacto');                 // igualdad estricta
expect(valor).toEqual({ a: 1 });              // igualdad profunda (objetos)
expect(valor).toBeNull();
expect(valor).toBeTruthy();
expect(valor).toBeFalsy();
expect(arreglo).toHaveLength(3);
expect(fn).toHaveBeenCalled();
expect(fn).toHaveBeenCalledWith(arg1, arg2);
expect(fn).toHaveBeenCalledTimes(1);
expect(typeof valor).toBe('string');
```

---

## 12. Rutas documentadas

### Usuarios

| Método | Ruta         | Auth | Body requerido                   | Respuesta exitosa |
|--------|--------------|------|----------------------------------|-------------------|
| POST   | /users       | No   | `{ name, email, password }`      | 201 — usuario creado |
| POST   | /login       | No   | `{ email, password }`            | 200 — `{ token }` |
| GET    | /users       | Sí   | —                                | 200 — lista de usuarios |
| GET    | /users/:id   | Sí   | —                                | 200 — usuario / 404 |
| PUT    | /users/:id   | Sí   | `{ name?, email?, password? }`   | 200 — usuario actualizado |
| DELETE | /users/:id   | Sí   | —                                | 204 / 404 |

### Libros (ejemplo)

| Método | Ruta                        | Auth | Body requerido                          | Respuesta exitosa |
|--------|-----------------------------|------|-----------------------------------------|-------------------|
| POST   | /books                      | Sí   | `{ title, author, price, isSold? }`     | 201 — libro creado |
| GET    | /books                      | Sí   | —                                       | 200 — lista |
| GET    | /books/:id                  | Sí   | —                                       | 200 / 404 |
| GET    | /books/author/:id           | Sí   | —                                       | 200 — libros del autor |
| PUT    | /books/:id                  | Sí   | `{ title?, author?, price? }`           | 200 / 404 |
| DELETE | /books/:id                  | Sí   | —                                       | 204 / 404 |
| POST   | /books/:bookId/buy/:userId  | Sí   | —                                       | 200 — mensaje / 404 |

---

> **Tip final:** Cuando recibas el enunciado, mapea cada funcionalidad a una sección de esta guía. Identifica primero los métodos del service (qué operaciones de BD necesitas), luego los métodos del controller (qué rutas expones), y finalmente conéctalos en las rutas.
