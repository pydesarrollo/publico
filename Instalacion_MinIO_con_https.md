# Instalación MinIO  versión 2025-04 en Ubuntu Server 22
---

### `1. Preparar el sistema`
Abre el terminal y actualiza
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget
```

### `2. Descargar MinIO versión específica`
MinIO publica sus releases en [https://github.com/minio/minio/releases](https://github.com/minio/minio/releases/).  
Ejemplo, la versión `RELEASE.2025-04-22T22-12-26Z`:
```bash
cd /usr/local/bin
sudo wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio.RELEASE.2025-04-22T22-12-26Z -O minio
sudo chmod +x minio
```
Si quieres la versión `RELEASE.2025-04-08T15-41-24Z` cambia la URL y el nombre del archivo.


### `3. Crear usuario y grupo para MinIO(recomendado)`
```bash
sudo useradd -r minio-user -s /sbin/nologin
```

### `4. Crear directorio de almacenamiento y dar permisos`
```bash
sudo mkdir -p /usr/local/share/minio
sudo chown minio-user:minio-user /usr/local/share/minio
```

### `5. Crear archivo de configuración de entorno`
Edita `/etc/default/minio` (crear si no existe):
```bash
sudo nano /etc/default/minio
```
Escribe:
```bash
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin123
MINIO_VOLUMES="/usr/local/share/minio"
MINIO_CONSOLE_ADDRESS=":9001"
```
Guarda y Cierra.

### `6. Crear servicio systemd para MinIO`
Crea `/etc/systemd/system/minio.service`:
```bash
sudo nano /etc/systemd/system/minio.service
```

Contenido:
```bash
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target

[Service]
User=minio-user
Group=minio-user
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_VOLUMES --console-address $MINIO_CONSOLE_ADDRESS
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
Guarda y cierra.

### `7. Recargar systemd y arrancar MinIO`
```bash
sudo systemctl daemon-reload
sudo systemctl enable minio
sudo systemctl start minio
```
Verifica que esté activo:
```
sudo systemctl status minio
```

### `8. Acceder a MinIO localmente`
* MinIO estará corriendo en puerto 9000 para el almacenamiento (API)
* Consola web en puerto 9001  

En el servidor(localhost)
* [http://localhost:9000](http://localhost:9000) → API MinIO (requiere cliente o SDK)
* [http://localhost:9001](http://localhost:9001) → Consola web MinIO (GUI)

Si quieres desde otro equipo en la red, usa la IP del servidor:

```
http://ip-del-servidor:9001
```
Para ingresar usa:
* Usuario: `minioadmin`
* Contraseña: `minioadmin123`

### `9. Instalación de Apache`

Abre terminal y actualiza 
```
sudo apt update
sudo apt upgrade -y
```

Instala Apache:
```bash
sudo apt install apache2 -y
```

Verifica el estado de Apache:
```bash
sudo systemctl status apache2
```

Habilita Apache para que inicie al arrancar:
```bash
sudo systemctl enable apache2
```

Permite tráfico HTTP/HTTPS en el firewall (opcional, si tienes UFW activo):
```bash
sudo ufw allow 'Apache Full'
sudo ufw reload
```

Prueba en el navegador:
```
http://<IP_DE_TU_SERVIDOR>
```

### `10. Crear Host Virtual de la  Consola Web MinIO `
Crea:
```bash
sudo nano /etc/apache2/sites-available/console.dirislimaeste.xyz.conf
```

Edita:
```bash
<VirtualHost *:80>
    ServerName console.dirislimaeste.xyz

    ProxyPreserveHost On
    ProxyPass / http://localhost:9001/
    ProxyPassReverse / http://localhost:9001/

    ErrorLog ${APACHE_LOG_DIR}/console.dirislimaeste.xyz_error.log
    CustomLog ${APACHE_LOG_DIR}/console.dirislimaeste.xyz_access.log combined
</VirtualHost>
```

Guarda y Cierra


### `11. Crear Host Virtual de la Api `
Crea:
```bash
sudo nano /etc/apache2/sites-available/dirislimaeste.xyz.conf
```

Edita:
```bash
<VirtualHost *:80>
    ServerName dirislimaeste.xyz

    ProxyPreserveHost On
    ProxyPass / http://localhost:9000/
    ProxyPassReverse / http://localhost:9000/

    ErrorLog ${APACHE_LOG_DIR}/dirislimaeste.xyz_error.log
    CustomLog ${APACHE_LOG_DIR}/dirislimaeste.xyz_access.log combined
</VirtualHost>
```

Guarda y Cierra

### `12. Activar Host Virtuales`
```bash
sudo a2ensite console.dirislimaeste.xyz.conf
sudo a2ensite dirislimaeste.xyz.conf
sudo service apache2 restart
```

### `13. Activar Módulo Proxy Inverso Apache`
```bash
sudo a2enmod proxy proxy_http
sudo service apache2 restart
```


### `14. Instalar Cerbot para Apache(Certificados SSL)`
```bash
sudo apt install certbot python3-certbot-apache
```

### `15. Obtener certificados digitales`
```bash
sudo certbot --apache -d dirislimaeste.xyz -d console.dirislimaeste.xyz
```

### `16. Actualizar variables de entorno de MinIO`
Edita  `sudo nano /etc/default/minio`:

```bash
MINIO_SERVER_URL=https://dirislimaeste.xyz
MINIO_BROWSER_REDIRECT_URL=https://console.dirislimaeste.xyz
```
Luego reinicia MinIO:
```bash
sudo systemctl restart minio
```

### `17. Validar Credenciales`
* Acceso a consola MinIO: https://console.dirislimaeste.xyz
* Acceso a endpoint S3: https://dirislimaeste.xyz

### `18. Habilitar Web Services Virtual Host Console`
Editar archivos:  
* `sudo nano /etc/apache2/sites-available/console.dirislimaeste.xyz.conf`
* `sudo nano /etc/apache2/sites-available/console.dirislimaeste.xyz-le-ssl.conf`

Agrega/Actualiza:  
```
    ProxyPass /ws/ ws://localhost:9001/ws/
    ProxyPassReverse /ws/ ws://localhost:9001/ws/
```

  ## Integración con Laravel 11
---

### `1. Crear Disco Personalizado`
config\filesystems.php
```php
'minio' => [
    'driver' => 's3',
    'key' => env('MIN_IO_ACCESS_KEY_ID'),
    'secret' => env('MIN_IO_SECRET_ACCESS_KEY'),
    'region' => env('MIN_IO_DEFAULT_REGION'),
    'bucket' => env('MIN_IO_BUCKET'),
    'url' => env('MIN_IO_URL'),
    'endpoint' => env('MIN_IO_ENDPOINT'),
    'use_path_style_endpoint' => filter_var(env('MIN_IO_USE_PATH_STYLE_ENDPOINT', true), FILTER_VALIDATE_BOOLEAN),
]
```
Explicación:
* driver: Driver S3
* key: Access Key Generado en MinIO
* secret: Contraseña del Access Key
* region: MinIO no utiliza región, pero es un valor requerido. Puedes dejar  `us-east-1`
* bucket: Nombre del Bucket creado en MinIO
* url: Campo Opcional para detallar la url del Servicio, por ejm: https://dirislimaeste.xyz
* endpoint: Necesaria para conectividad entre laravel y MinIO
* use_path_style_endpoint: Esta opción debe estar en true para que Laravel genere URLs correctas para MinIO.

### `2. Registrar variables de entorno en .env`
```bash
    MIN_IO_ACCESS_KEY_ID=VTJS7pSdO91tGBK4brXR
    MIN_IO_SECRET_ACCESS_KEY=RZ0ImuxqeirvIB3PgumRcl
    MIN_IO_ENDPOINT=https://dirislimaeste.xyz
    MIN_IO_DEFAULT_REGION=us-east-1
    MIN_IO_BUCKET=sigesa
    MIN_IO_USE_PATH_STYLE_ENDPOINT=true

```


### `3. Script de Prueba de Integración`
```php
    #1. Archivo txt
    $contenido = "Este es un archivo de prueba.\nFecha: " . now();
    $nombreArchivo = 'test_' . now() . '.txt';
    dump('Contenido de Txt: ' ,  $contenido);

    #2. Subir archivo a MinIO
    Storage::disk('archivodigital')->put($nombreArchivo, $contenido);

    dump('Archivo subido: ' . $nombreArchivo);

    #3. Validar si el archivo existe
    if (Storage::disk('archivodigital')->exists($nombreArchivo)) {
        dump('El archivo ' . $nombreArchivo . ' existe en el disco archivodigital.');
    } else {
        dump('El archivo ' . $nombreArchivo . ' no existe en el disco archivodigital.');
    }

    #4. Obtener URL temporal
    $url_temporal = Storage::disk('archivodigital')->temporaryUrl(
        $nombreArchivo,
        now()->addMinutes(10)
    );

    dump('URL temporal: ' , $url_temporal);

```

### `4. Resultado Prueba`
```bash
"Contenido de Txt: " // routes\web.php:24
"""
Este es un archivo de prueba.
Fecha: 2025-06-03 11:28:02
""" // routes\web.php:24
"Archivo subido: test_2025-06-03 11:28:02.txt" // routes\web.php:29
"El archivo test_2025-06-03 11:28:02.txt existe en el disco archivodigital." // routes\web.php:33
"URL temporal: " // routes\web.php:44
"https://dirislimaeste.xyz/sigesa/test_2025-06-03%2011%3A28%3A02.txt?X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=VTJS7pSdO91tGBK4brXR%2F20250603%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20250603T162803Z&X-Amz-SignedHeaders=host&X-Amz-Expires=600&X-Amz-Signature=9ca67553368b3a52e89f8f918c3cf5d87de496c752168416a7d7f139d08d3878
```

