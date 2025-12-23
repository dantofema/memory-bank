# Artefactos Lean: E-commerce con Sincronización Meli & Andreani

## 1. Problem Statement
Los negocios locales en Argentina pierden eficiencia al gestionar stock de forma manual entre sus tiendas físicas/Mercado Libre y sus e-commerce propios, sumado a la complejidad de integrar logísticas locales como Andreani de forma automatizada.

## 2. Objective / Outcome Definition
**Resultado:** Un MVP funcional de tienda online que sincronice stock/descripciones con Mercado Libre y calcule envíos en tiempo real con Andreani.
**Definition of Done:** Un usuario puede comprar un producto, pagar de forma segura y el stock se descuenta automáticamente tanto en la tienda local como en Mercado Libre.

## 3. Scope Boundary
*   **Dentro de scope:** Catálogo, Carrito, Checkout, API Mercado Libre (Sincronización de stock), API Andreani (Cálculo de costo y etiquetas), Panel Administrativo (Filament).
*   **Fuera de scope:** Multi-moneda, sistema de cupones complejos, analítica avanzada de IA, soporte multi-idioma.

## 4. Technical Hypothesis
Al utilizar **Laravel con un sistema de procesamiento asíncrono (Redis/Horizon)**, podemos garantizar que la sincronización con Mercado Libre y Andreani no afecte la experiencia de usuario (latencia) durante el checkout.

## 5. Primary Flow (Happy Path)
Usuario busca producto -> Agrega al carrito -> Checkout con cálculo de envío Andreani -> Pago exitoso -> Actualización de stock local y en Mercado Libre (vía Job) -> Generación de guía de envío.

## 6. Minimal Data Model
*   **Products:** id, name, description, price, stock, meli_id.
*   **Orders:** id, user_id, total, status, shipping_id (Andreani).
*   **Users:** id, email, password, roles.
*   **Sync_Logs:** id, entity, status, response.

## 7. Key Technical Decisions
*   **Framework:** Laravel 11 para una estructura sólida y rápida.
*   **Admin Panel:** **FilamentPHP** para un backoffice eficiente y CRUDs complejos.
*   **Base de Datos:** PostgreSQL por su robustez en transacciones.
*   **Colas:** Redis + Laravel Horizon para gestionar los tiempos de respuesta de las APIs externas.

## 8. Security Baseline
*   Validación de todos los inputs en Requests.
*   Uso de **Policies y Gates** para acceso al panel administrativo.
*   Protección de credenciales de APIs (Meli/Andreani) mediante variables de entorno y encriptación.

## 9. Executable Task Definition
Configurar el entorno Laravel + Filament y modelar la entidad `Product` con los campos necesarios para la integración con Mercado Libre.


