# practicaDNSok
---

## Proyecto DNS Maestro-Esclavo con Vagrant

### Descripción

Este proyecto configura un entorno DNS con Bind9 en un servidor maestro y un servidor esclavo para gestionar la zona `sistema.test`, utilizando Vagrant para virtualizar ambos servidores. El sistema permite resoluciones directas e inversas, alias y configuración de servidor de correo.

### Requisitos

- **Vagrant** y **VirtualBox** para la virtualización.
- **Git** para el control de versiones.
- **Bind9** como software de servidor DNS en ambos servidores virtuales.

### Contenido del Repositorio

- **`Vagrantfile`**: Define y configura las máquinas virtuales `tierra` (maestro) y `venus` (esclavo) en la red `192.168.57.0/24`.
- **`configuracion-servidor-dns.pdf`**: Documento guía para la configuración detallada de un servidor DNS.
- **`test.sh`**: Script para realizar comprobaciones automáticas del DNS en ambas máquinas.
- **`README.md`**: Documentación de la práctica.
- **`.gitignore`**: Ignora archivos innecesarios como `.vagrant/`, archivos de respaldo (`*.bak`, `*.backup`) y logs (`*.log`).
- **`LICENSE`**: Licencia de uso.

### Infraestructura de Red

| Servidor | FQDN                  | IP             |
|----------|------------------------|----------------|
| Maestro  | tierra.sistema.test    | 192.168.57.103 |
| Esclavo  | venus.sistema.test     | 192.168.57.102 |
| Cliente  | mercurio.sistema.test  | 192.168.57.101 |
| Correo   | marte.sistema.test     | 192.168.57.104 |

### Configuración del DNS

#### 1. Configuración de `named.conf.options`

En el archivo `/etc/bind/named.conf.options` se definen las opciones generales para ambos servidores:

- Solo escucha en IPv4:
  ```plaintext
  OPTIONS="-u bind -4"
  ```

- Definición de la ACL para limitar las consultas a la red `192.168.57.0/24`:
  ```plaintext
  acl confiables {
      192.168.57.0/24;
  };
  ```

- Configuración de `dnssec-validation`, el reenviador y las opciones de recursión:
  ```plaintext
  options {
      directory "/var/cache/bind";
      listen-on port 53 { 192.168.57.103; };
      allow-transfer { none; };
      recursion yes;
      allow-recursion { confiables; };
      dnssec-validation yes;
      forwarders { 208.67.222.222; };
  };
  ```

#### 2. Configuración de `named.conf.local`

Este archivo se usa para definir las zonas en el servidor maestro (`tierra`):

```plaintext
zone "sistema.test" {
    type master;
    file "/etc/bind/db.sistema.test";
    allow-transfer { 192.168.57.102; };  # Permitir transferencia al esclavo
};

zone "57.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.sistema.test.rev";
    allow-transfer { 192.168.57.102; };  # Transferencia inversa al esclavo
};
```

En el servidor esclavo (`venus`), solo se necesita declarar la zona como tipo `slave` apuntando al maestro.

#### 3. Archivos de Zona

##### Archivo `db.sistema.test` (resolución directa):

```plaintext
$TTL    86400
@       IN  SOA     tierra.sistema.test. admin.sistema.test. (
                    2024102501 ; Serial
                    3600       ; Refresh
                    1800       ; Retry
                    604800     ; Expire
                    86400 )    ; Negative Cache TTL

@       IN  NS      tierra.sistema.test.
@       IN  NS      venus.sistema.test.

tierra  IN  A       192.168.57.103
venus   IN  A       192.168.57.102
mercurio IN  A      192.168.57.101
marte   IN  A       192.168.57.104

ns1     IN  CNAME   tierra.sistema.test.
ns2     IN  CNAME   venus.sistema.test.
mail    IN  CNAME   marte.sistema.test.

@       IN  MX 10   mail.sistema.test.
```

##### Archivo `db.sistema.test.rev` (resolución inversa):

```plaintext
$TTL    86400
@       IN  SOA     tierra.sistema.test. admin.sistema.test. (
                    2024102501 ; Serial
                    3600       ; Refresh
                    1800       ; Retry
                    604800     ; Expire
                    86400 )    ; Negative Cache TTL

@       IN  NS      tierra.sistema.test.

103     IN  PTR     tierra.sistema.test.
102     IN  PTR     venus.sistema.test.
101     IN  PTR     mercurio.sistema.test.
104     IN  PTR     marte.sistema.test.
```

### Instrucciones de Ejecución

1. **Clona el repositorio y navega a la carpeta del proyecto**:
   ```bash
   git clone <URL_DEL_REPOSITORIO>
   cd <NOMBRE_DEL_PROYECTO>
   ```

2. **Inicia las máquinas virtuales con Vagrant**:
   ```bash
   vagrant up
   ```

3. **Configura y reinicia Bind**:
   - En `tierra`, verifica y reinicia Bind:
     ```bash
     vagrant ssh tierra
     sudo named-checkconf
     sudo systemctl restart bind9
     ```
   - En `venus`, reinicia Bind y verifica:
     ```bash
     vagrant ssh venus
     sudo systemctl restart bind9
     ```

4. **Ejecuta el script de prueba**:
   Ejecuta el script `test.sh` para comprobar el funcionamiento del DNS:
   ```bash
   ./test.sh 192.168.57.103
   ```

### Comprobaciones

- **Registros A**: Asegúrate de que cada nombre (`tierra`, `venus`, `mercurio`, `marte`) resuelve su IP correcta.
- **Registros PTR (inversa)**: Verifica que las IPs resuelvan los nombres correspondientes.
- **Alias**: Confirma que `ns1.sistema.test` y `ns2.sistema.test` resuelvan a `tierra.sistema.test` y `venus.sistema.test`, respectivamente.
- **Servidor MX**: Comprueba que `mail.sistema.test` resuelve a `marte.sistema.test`.
- **Transferencia de Zona**: Verifica en los logs del esclavo (`venus`) que recibe la zona desde el maestro (`tierra`) o usa el comando `dig AXFR` en `venus` para confirmar.

### Licencia

Este proyecto está bajo la licencia [MIT].
