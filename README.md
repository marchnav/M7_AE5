-------------M5_AE7------------Bootcamp ORM — Django + MySQL (README)

Proyecto didáctico que implementa operaciones con Django ORM, consultas SQL crudas, índices, anotaciones, cursores y procedimientos almacenados en MySQL. Incluye prácticas de seguridad para no exponer secretos en el repositorio.

------------------------------------------------------------
Estructura principal
------------------------------------------------------------
bootcamp_orm/
├─ config/
│  ├─ settings.py           # Config con variables de entorno (dotenv) y MySQL
│  ├─ urls.py
│  └─ wsgi.py / asgi.py
├─ productos/
│  ├─ migrations/           # 0001 (modelo), 0002 (índice)
│  ├─ models.py             # Modelo Producto
│  ├─ admin.py / views.py / ...
├─ manage.py
├─ .env                     # (NO subir) credenciales locales
├─ .env.example             # plantilla de variables
└─ .gitignore               # exclusiones seguras

------------------------------------------------------------
Modelo principal (productos/models.py)
------------------------------------------------------------
from django.db import models

class Producto(models.Model):
    nombre = models.CharField(max_length=100)
    precio = models.DecimalField(max_digits=5, decimal_places=2)
    disponible = models.BooleanField(default=True)

    class Meta:
        ordering = ["nombre"]
        indexes = [
            models.Index(fields=["nombre"], name="idx_producto_nombre"),
        ]

    def __str__(self):
        return f"{self.nombre} (${self.precio})"

------------------------------------------------------------
Seguridad del repositorio
------------------------------------------------------------
- NO subas .env. Usa .env.example para documentar variables.
- .gitignore incluye: .venv/, __pycache__/, .env*, *.sqlite3, etc.
- Variables sensibles se cargan con python-dotenv.

Plantilla .env.example:
DB_NAME=bootcamp_orm_db
DB_USER=bootcamp_user
DB_PASSWORD=<<REEMPLAZA_CON_TU_CLAVE>>
DB_HOST=localhost
DB_PORT=3306

------------------------------------------------------------
Requisitos
------------------------------------------------------------
- Python 3.11+ (funciona con 3.13)
- MySQL 8.x (server y cliente)
- Paquetes:
  - Django>=5.0,<6
  - mysqlclient
  - python-dotenv

------------------------------------------------------------
Puesta en marcha (resumen)
------------------------------------------------------------
1) Crear y activar entorno virtual.
2) Instalar dependencias (o las listadas arriba).
3) Configurar MySQL: BD y usuario con permisos sobre bootcamp_orm_db.
4) Crear .env a partir de .env.example (no subirlo).
5) Migrar:
   python manage.py migrate
6) (Opcional) Crear superusuario:
   python manage.py createsuperuser
7) Levantar servidor:
   python manage.py runserver

------------------------------------------------------------
Datos de prueba (shell de Django)
------------------------------------------------------------
from productos.models import Producto

Producto.objects.bulk_create([
    Producto(nombre="Ajedrez de Madera", precio="79.90", disponible=True),
    Producto(nombre="Auriculares Pro", precio="129.99", disponible=True),
    Producto(nombre="Alfombra Gamer", precio="49.90", disponible=False),
    Producto(nombre="Cable USB-C", precio="9.99", disponible=True),
    Producto(nombre='Notebook 14"', precio="499.00", disponible=True),
    Producto(nombre="Adaptador HDMI", precio="15.50", disponible=False),
])

------------------------------------------------------------
Entregables del Bootcamp (cómo se resolvió cada punto)
------------------------------------------------------------
1) Recuperando registros con Django ORM
from productos.models import Producto
Producto.objects.all()

2) Aplicando filtros
# Precio > 50
Producto.objects.filter(precio__gt=50)
# Nombre empieza con "A"
Producto.objects.filter(nombre__startswith="A")
# Solo disponibles
Producto.objects.filter(disponible=True)

3) Ejecutando queries SQL desde Django (raw())
from productos.models import Producto
qs = Producto.objects.raw(
    "SELECT id, nombre, precio, disponible FROM productos_producto WHERE precio < 100"
)
for p in qs:
    print(p.id, p.nombre, p.precio, p.disponible)

4) Mapeando campos de consultas al modelo (incluye columna extra)
# Importante: la PK debe venir como id
qs = Producto.objects.raw("""    SELECT
      id, nombre, precio, disponible,
      (precio * 1.16) AS precio_con_impuesto
    FROM productos_producto
    WHERE precio < 200
""")
for p in qs:
    print(p.id, p.nombre, p.precio, getattr(p, "precio_con_impuesto", None))

5) Índices: qué son y creación en Django
- Un índice acelera búsquedas y filtros en un campo.
- Se definió en Meta.indexes sobre nombre.
- Migraciones: python manage.py makemigrations && python manage.py migrate
- Verificación del plan:
Producto.objects.filter(nombre__startswith="A").explain()

6) Exclusión de campos del modelo
# Defer: omite cargar el campo hasta que lo accedas
qs = Producto.objects.defer("disponible")
# Only: carga solo los campos listados
qs = Producto.objects.only("nombre", "precio")
# Django traerá los campos diferidos con una consulta adicional si los accedes.

7) Añadiendo anotaciones (annotate) — precio_con_impuesto (16%)
from decimal import Decimal
from django.db.models import F, DecimalField, ExpressionWrapper

qs = Producto.objects.annotate(
    precio_con_impuesto=ExpressionWrapper(
        F("precio") * Decimal("1.16"),
        output_field=DecimalField(max_digits=7, decimal_places=2),
    )
).values("id", "nombre", "precio", "precio_con_impuesto")

8) raw() con parámetros (evita SQL injection)
limite = 100
qs = Producto.objects.raw(
    "SELECT id, nombre, precio, disponible FROM productos_producto WHERE precio < %s",
    [limite],
)

9) SQL personalizado con connection.cursor() (INSERT/UPDATE/DELETE)
from django.db import connection, transaction
with transaction.atomic():
    with connection.cursor() as cursor:
        cursor.execute(
            "UPDATE productos_producto SET disponible = 0 WHERE precio < %s",
            [10],
        )

10) Conexiones y cursores (SELECT manual)
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute("""        SELECT id, nombre, precio
        FROM productos_producto
        ORDER BY precio DESC
        LIMIT 5
    """)
    filas = cursor.fetchall()

11) Procedimientos almacenados
-- Creación en MySQL:
USE bootcamp_orm_db;
DROP PROCEDURE IF EXISTS sp_productos_mayores_a_precio;
DELIMITER $$
CREATE PROCEDURE sp_productos_mayores_a_precio(IN p_min DECIMAL(5,2))
BEGIN
    SELECT id, nombre, precio, disponible
    FROM productos_producto
    WHERE precio >= p_min
    ORDER BY precio DESC;
END $$
DELIMITER ;

-- Invocación desde Django:
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute("CALL sp_productos_mayores_a_precio(%s)", [50])
    rows = cursor.fetchall()

# Nota: callproc() con MySQLdb puede no retornar el resultset directamente;
# ejecutar CALL ... y luego fetchall() es más fiable para leer filas.

------------------------------------------------------------
Comandos útiles
------------------------------------------------------------
- Migrar: python manage.py migrate
- Crear migración: python manage.py makemigrations
- Shell ORM: python manage.py shell
- Superusuario: python manage.py createsuperuser
- Servidor dev: python manage.py runserver

------------------------------------------------------------
Checklist de seguridad antes de subir
------------------------------------------------------------
- [ ] .env NO está en Git.
- [ ] Existe .env.example con variables sin secretos.
- [ ] ALLOWED_HOSTS configurado para despliegue.
- [ ] DEBUG=False en producción.
- [ ] Usuario MySQL con permisos limitados a esta BD.
- [ ] Backups si se requiere.

Licencia
Uso académico/educacional. 

