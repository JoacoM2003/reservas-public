# 🗓️ Sistema de Gestión de Turnos & Reservas

![FastAPI](https://img.shields.io/badge/FastAPI-005571?style=for-the-badge&logo=fastapi)
![React](https://img.shields.io/badge/react-%2320232b.svg?style=for-the-badge&logo=react&logoColor=%2361DAFB)
![TypeScript](https://img.shields.io/badge/typescript-%23007acc.svg?style=for-the-badge&logo=typescript&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)

Una solución integral y profesional para la gestión de turnos, recursos y servicios, diseñada con un enfoque en la escalabilidad, mantenibilidad y rendimiento óptimo.

## 🚀 Sobre el Proyecto

Este proyecto nació como una evolución de una aplicación sencilla de gestión de turnos. Con el objetivo de profundizar en patrones de diseño avanzados y arquitecturas robustas, el sistema fue rediseñado utilizando **Arquitectura Hexagonal (Puertos y Adaptadores)** y tecnologías de punta en el ecosistema de Python y JavaScript.

### ¿Qué resuelve?
La aplicación permite a organizaciones (gimnasios, consultorios, centros deportivos, etc.) gestionar de manera centralizada:
- **Reserva de Recursos**: Control de disponibilidad de instalaciones, máquinas o espacios físicos.
- **Bloqueos y Excepciones**: Gestión de bloqueos temporales por mantenimiento, feriados o indisponibilidad imprevista.
- **Gestión de Servicios y Proveedores**: Configuración de prestadores, tipos de servicios con duraciones personalizadas y roles específicos.
- **Control de Horarios**: Flexibilidad total para definir ventanas de atención y disponibilidad por recurso.
- **Automatización**: Procesamiento de tareas en segundo plano (notificaciones, limpiezas de estado) mediante workers asíncronos.
- **Seguridad**: Autenticación robusta y control de acceso.

---

## 🏗️ Arquitectura: ¿Por qué Hexagonal?

La decisión de migrar de una arquitectura monolítica tradicional (MVC) a una **Arquitectura Hexagonal** se basó en la necesidad de desacoplar la lógica de negocio de los detalles técnicos (infraestructura).

### Beneficios obtenidos:
1. **Independencia Tecnológica**: El núcleo de la aplicación (Dominio) no conoce si los datos vienen de una API, una base de datos SQL o archivos de texto.
2. **Testabilidad**: Permite testear la lógica de negocio de forma aislada sin necesidad de levantar bases de datos o servicios externos.
3. **Mantenibilidad**: Facilita la evolución del proyecto. Si mañana decidimos cambiar PostgreSQL por MongoDB o FastAPI por otro framework, el impacto en las reglas de negocio será mínimo.

### 📂 Estructura del Proyecto (Árbol de Directorios)

Para entender cómo está construida la aplicación sin necesidad de leer el código, aquí tienes un mapa de la organización del proyecto:

```text
Turnos/
├── backend/                  # API REST construida con FastAPI (Arquitectura Hexagonal)
│   ├── app/
│   │   ├── api/              # Capa de Presentación: Endpoints y Controladores (Routers de FastAPI)
│   │   ├── application/      # Capa de Aplicación: Casos de Uso (Lógica de orquestación)
│   │   ├── core/             # Configuraciones globales, dependencias, seguridad (autenticación JWT, config)
│   │   ├── domain/           # Capa de Dominio: Entidades puras y Puertos (Interfaces sin dependencias de frameworks)
│   │   ├── infrastructure/   # Capa de Infraestructura: Adaptadores a BD (SQLAlchemy), Redis, Celery, y Repositorios
│   │   ├── worker/           # Definición de tareas asíncronas de Celery (ej: liberación de turnos caducados, envíos asíncronos)
│   │   └── main.py           # Instancia de FastAPI y punto de entrada de la aplicación
│   ├── alembic/              # Configuración y Scripts de migración de esquema en la BD (Database Versioning)
│   └── Dockerfile            # Configuración para creación de imagen de contenedor Docker para el backend
│
├── frontend/                 # SPA construida con React (Redirecciones y Vistas), TypeScript y Vite
│   ├── src/
│   │   ├── components/       # Componentes visuales genéricos y reutilizables (Botones, Listas, Navegación)
│   │   ├── contexts/         # Estado global de la aplicación React (Contexto de Sesión de Usuario, AuthContext)
│   │   ├── hooks/            # Custom Hooks de React para aislar lógica de componentes (ej: validaciones, peticiones)
│   │   ├── pages/            # Componentes de Vistas o Páginas principales de la aplicación, asociadas al enrutador
│   │   ├── services/         # Funciones que interactúan con la API remota del Backend y manejan su comunicación
│   │   ├── types/            # Definiciones de estructura de datos (Interfaces estáticas de TypeScript)
│   │   └── utils/            # Funciones de ayuda general, como parsers de fechas, o cálculos utilitarios
│   ├── nginx.conf            # Reglas de proxy web / enrutamiento interno Nginx para frontend compilado
│   └── Dockerfile            # Construcción del frontend (Vite Build) y servido con proxy inverso de Nginx
│
└── docker-compose.yml        # Orquestación de infraestructura local completa (Backend, Frontend, PostgreSQL, Redis, Celery Worker)
```

### 🧩 Flujo de Datos y Convivencia de Capas
La arquitectura hexagonal promueve que las dependencias apunten de afuera hacia adentro (hacia el Dominio). Un flujo típico dentro del backend (por ejemplo, para "Reservar un Turno") se visualiza así:

1. **El Usuario llama a la API**: El frontend (React) hace una petición HTTP `POST` a `/turnos`. El adaptador en `backend/app/api/` recibe la solicitud y usa Pydantic para validar los datos que entran.
2. **Se Invoca al Caso de Uso**: El controlador de la API no contiene reglas del negocio; directamente le entrega los datos a un servicio de `application/` que orquesta la operación "Reservar".
3. **El Dominio Valida las Reglas**: El caso de uso verifica mediante objetos del `domain/` si el turno cumple con las políticas del negocio (ej. el proveedor no atiende, ya pasó el horario, se bloqueó esa fecha). El Dominio ignora **completamente** que existe una base de datos o FastAPI.
4. **La Infraestructura Persiste los Datos**: Si la regla de negocio del Dominio se cumple, el Caso de Uso usa una "Interfaz" (Puerto) para guardar la reserva. Quien en realidad realiza esa grabación final es un Repositorio SQL dentro de `infrastructure/`, el cual implementa dicho puerto. También en esta etapa se pueden enviar tareas al worker en background (`worker/`).

---


## 🛠️ Stack Tecnológico

### Backend (FastAPI)
- **FastAPI**: Elegido por su alto rendimiento (basado en Starlette y Pydantic), soporte nativo para `async/await` y generación automática de documentación Swagger/OpenAPI.
- **SQLAlchemy 2.0 + Alembic**: Gestión de persistencia con el ORM más potente de Python y migraciones controladas de base de datos.
- **Celery + Redis**: Arquitectura orientada a eventos y procesamiento de tareas asíncronas para una experiencia de usuario fluida sin bloqueos.
- **Pydantic v2**: Validación de datos y esquemas con tipado estático riguroso.

### Frontend (React + Vite)
- **React 18**: Biblioteca líder para interfaces dinámicas y componentes reutilizables.
- **TypeScript**: Garantiza la integridad de los datos en todo el flujo del frontend.
- **Vite**: Herramienta de construcción ultrarrápida para una experiencia de desarrollo superior.
- **Tailwind CSS**: Diseño moderno, responsivo y "Utility-first".

---

## 🔧 Instalación y Despliegue

El proyecto está completamente **Dockerizado**, lo que garantiza que funcione de la misma manera en cualquier entorno.

### Requisitos previos:
- Docker y Docker Compose instalado.

### Pasos para ejecutar:

1. Clonar el repositorio.
2. Crear los archivos `.env` en las carpetas `backend/` y `frontend/` (puedes basarte en los `.env-example`).
3. Ejecutar el comando mágico:
   ```bash
   docker-compose up --build
   ```

La aplicación estará disponible en:
- **Frontend**: [http://localhost:8080](http://localhost:8080)
- **API Docs (Swagger)**: [http://localhost:8000/docs](http://localhost:8000/docs)

---

## 📈 Evolución y Aprendizaje

Este proyecto representa un salto cualitativo e n mi desarrollo profesional. El sistema original fue concebido como una aplicación monolítica utilizando **Django (MVC)**, centrada en la funcionalidad rápida. Puedes ver esa versión aquí:
🔗 [Turnero (Legacy - Django/MVC)](https://github.com/JoacoM2003/turnero)

Este repositorio es el resultado de rediseñar esa solución desde cero para aplicar conceptos avanzados de ingeniería de software:
- **Migración de Django a FastAPI**: Buscando mayor rendimiento y soporte asíncrono.
- **Cambio de MVC a Arquitectura Hexagonal**: Para asegurar que la lógica de negocio sea el centro del sistema, independiente de frameworks o bases de datos.
- **Escalabilidad**: Implementación de Workers y Redis para tareas pesadas, algo que no estaba presente en la versión inicial.

---
Hecho por [Joaquín](https://github.com/JoacoM2003)
