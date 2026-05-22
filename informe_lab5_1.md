Arquitectura del Grupo Triska

Configuración VM Base de Datos — Sebastian Allon (10.124.88.186)
Red estática
yamlnetwork:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      optional: true
      addresses:
        - 10.124.88.186/24
      routes:
        - to: default
          via: 10.250.16.1
      nameservers:
        addresses:
          - 8.8.8.8
Hardening SSH
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers sebas
UFW Base de Datos
bashsudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.124.88.0/24 to any port 2222/tcp
sudo ufw allow from 10.124.88.183 to any port 3306/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

El puerto 3306 solo está permitido desde la IP del servidor web (10.124.88.183). Ningún otro equipo puede acceder a la base de datos.

MariaDB
bashsudo apt install mariadb-server -y
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
Configuración de bind-address en /etc/mysql/mariadb.conf.d/50-server.cnf:
bind-address = 10.124.88.186
Usuario de aplicación creado:
sqlCREATE DATABASE lab51db;
CREATE USER 'appuser'@'10.124.88.183' IDENTIFIED BY 'ContraseñaSegura123!';
GRANT ALL PRIVILEGES ON lab51db.* TO 'appuser'@'10.124.88.183';
FLUSH PRIVILEGES;
Captura: [Captura de sudo ufw status verbose en VM DB]

Pruebas de Seguridad Cruzadas
Prueba 1 — TLS desde VM Jose hacia Sebastian
bashopenssl s_client -connect 10.124.88.186:443 -tls1_2 </dev/null
openssl s_client -connect 10.124.88.186:443 -tls1_1 </dev/null
Resultado esperado: TLS 1.2 funciona, TLS 1.1 rechazado.
Captura: [Captura del resultado]
Prueba 2 — Cabeceras de seguridad desde VM Jose hacia Sebastian
bashcurl -k -I https://10.124.88.186
Resultado esperado: Strict-Transport-Security y X-Frame-Options presentes.
Captura: [Captura del resultado]
Prueba 3 — Segmentación de red — Puerto 3306 debe fallar
Desde la VM de Jose hacia la DB de Sebastian:
bashnc -vz -w 2 10.124.88.186 3306
Resultado esperado: Connection timed out o No route to host — UFW bloqueando correctamente.
Captura: [Captura del resultado]
Prueba 4 — SSH ofuscado desde VM Jose hacia Sebastian
Puerto 22 debe fallar:
bashssh sebas@10.124.88.186
Puerto 2222 debe funcionar:
bashssh -p 2222 sebas@10.124.88.186
Captura: [Captura de ambos resultados]
Prueba 5 — TLS desde VM Sebastian hacia Jose
bashopenssl s_client -connect 10.124.88.183:443 -tls1_2 </dev/null
openssl s_client -connect 10.124.88.183:443 -tls1_1 </dev/null
Captura: [Captura del resultado]
Prueba 6 — Cabeceras de seguridad desde VM Sebastian hacia Jose
bashcurl -k -I https://10.124.88.183
Captura: [Captura del resultado]
Prueba 7 — SSH ofuscado desde VM Sebastian hacia Jose
Puerto 22 debe fallar:
bashssh maldonado-jose@10.124.88.183
Puerto 2222 debe funcionar:
bashssh -p 2222 maldonado-jose@10.124.88.183
Captura: [Captura de ambos resultados]

Conclusiones
Este laboratorio permitió implementar una infraestructura segura con múltiples capas de defensa aplicando el principio de defensa en profundidad:

Hardening SSH elimina los vectores de ataque más comunes como acceso root y autenticación por contraseña.
UFW con política deny por defecto asegura que solo el tráfico explícitamente autorizado pueda ingresar.
TLS 1.2/1.3 garantiza que las comunicaciones web estén cifradas, rechazando protocolos obsoletos e inseguros.
Cabeceras de seguridad HTTP protegen a los usuarios contra ataques como clickjacking e inyección de tipos MIME.
Segmentación de red asegura que la base de datos sea inaccesible desde internet o desde equipos no autorizados, reduciendo la superficie de ataque.

La combinación de estas medidas refleja las prácticas utilizadas en entornos de producción reales en administración de sistemas y DevOps.