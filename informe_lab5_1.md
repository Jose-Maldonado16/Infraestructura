# Laboratorio 5.1: Hardening Integral y Seguridad TLS

**Universidad San Francisco Xavier de Chuquisaca**  
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes (SIS313)  
**Docente:** Ing. Marcelo Quispe Ortega  
**Semestre:** 1/2026  
**Grupo:** Triska  
**Dominio:** grupo-lab51.local  

## Sección 3: Práctica en Grupo

### Arquitectura del Grupo Triska

| VM | Integrante | IP | Rol |
|---|---|---|---|
| Servidor Web | Jose Maldonado | 10.124.88.183 | Nginx + TLS + UFW |
| Base de Datos | Sebastian Allon | 10.124.88.186 | MariaDB + UFW |

Ambas VMs configuradas en **modo Bridge** para comunicación directa en la red del laboratorio.

---

### Configuración VM Base de Datos — Sebastian Allon (10.124.88.186)

#### Red estática

```yaml
network:
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
```

#### Hardening SSH

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers sebas
```

#### UFW Base de Datos

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.124.88.0/24 to any port 2222/tcp
sudo ufw allow from 10.124.88.183 to any port 3306/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

> El puerto 3306 solo está permitido desde la IP del servidor web (10.124.88.183). Ningún otro equipo puede acceder a la base de datos.

#### MariaDB

```bash
sudo apt install mariadb-server -y
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```

Configuración de `bind-address` en `/etc/mysql/mariadb.conf.d/50-server.cnf`:

```
bind-address = 10.124.88.186
```

Usuario de aplicación creado:

```sql
CREATE DATABASE lab51db;
CREATE USER 'appuser'@'10.124.88.183' IDENTIFIED BY 'ContraseñaSegura123!';
GRANT ALL PRIVILEGES ON lab51db.* TO 'appuser'@'10.124.88.183';
FLUSH PRIVILEGES;
```

![UFW status](capturas/1.jpg)

---

### Pruebas de Seguridad Cruzadas

#### Prueba 1 — TLS desde VM Jose hacia Sebastian

```bash
openssl s_client -connect 10.124.88.186:443 -tls1_2 </dev/null
openssl s_client -connect 10.124.88.186:443 -tls1_1 </dev/null
```

Resultado esperado: TLS 1.2 funciona, TLS 1.1 rechazado.

![TLS](capturas/2.jpg)
![TLS](capturas/3.jpg)

#### Prueba 2 — Cabeceras de seguridad desde VM Jose hacia Sebastian

```bash
curl -k -I https://10.124.88.186
```

Resultado esperado: `Strict-Transport-Security` y `X-Frame-Options` presentes.

![HTTPS](capturas/10.jpg)

#### Prueba 3 — Segmentación de red — Puerto 3306 debe fallar

Desde la VM de Jose hacia la DB de Sebastian:

```bash
nc -vz -w 2 10.124.88.186 3306
```

Resultado esperado: `Connection timed out` o `No route to host` — UFW bloqueando correctamente.

![3306](capturas/5.jpg)

#### Prueba 4 — SSH ofuscado desde VM Jose hacia Sebastian

Puerto 22 debe fallar:

```bash
ssh sebas@10.124.88.186
```

Puerto 2222 debe funcionar:

```bash
ssh -p 2222 sebas@10.124.88.186
```

![SSH](capturas/6.jpg)
![SSH](capturas/7.jpg)

#### Prueba 5 — TLS desde VM Sebastian hacia Jose

```bash
openssl s_client -connect 10.124.88.183:443 -tls1_2 </dev/null
openssl s_client -connect 10.124.88.183:443 -tls1_1 </dev/null
```

#### Prueba 6 — Cabeceras de seguridad desde VM Sebastian hacia Jose

```bash
curl -k -I https://10.124.88.183
```

![HTTPS](capturas/11.jpg)

#### Prueba 7 — SSH ofuscado desde VM Sebastian hacia Jose

Puerto 22 debe fallar:

```bash
ssh maldonado-jose@10.124.88.183
```

Puerto 2222 debe funcionar:

```bash
ssh -p 2222 maldonado-jose@10.124.88.183
```

![SHH](capturas/8.jpg)
![SHH](capturas/9.jpg)

---

## Conclusiones

Este laboratorio permitió implementar una infraestructura segura con múltiples capas de defensa aplicando el principio de **defensa en profundidad**:

- **Hardening SSH** elimina los vectores de ataque más comunes como acceso root y autenticación por contraseña.
- **UFW con política deny por defecto** asegura que solo el tráfico explícitamente autorizado pueda ingresar.
- **TLS 1.2/1.3** garantiza que las comunicaciones web estén cifradas, rechazando protocolos obsoletos e inseguros.
- **Cabeceras de seguridad HTTP** protegen a los usuarios contra ataques como clickjacking e inyección de tipos MIME.
- **Segmentación de red** asegura que la base de datos sea inaccesible desde internet o desde equipos no autorizados, reduciendo la superficie de ataque.

La combinación de estas medidas refleja las prácticas utilizadas en entornos de producción reales en administración de sistemas y DevOps.
