# Evidencia – Actividad: Tu primer servidor en EC2 (Simulación)

**Estudiante:** Valentina Atencio Díaz  
**Actividad:** Tu primer servidor en EC2  
**Fecha:** 27 de noviembre de 2026  
**Región AWS:** `us-east-2` (Ohio)  
**ID de Instancia:** `i-0a9b8c7d6e5f4a3b2`  
**Security Group:** `web-valentina` (`sg-0123456789abcdef0`)  
**Bucket S3:** `practica-roles-valentina-2026`  

---

## 1. Evidencias de Ejecución (Pasos Solicitados)

### Paso 12: Reglas del Security Group (`web-valentina`)
Consulta de las reglas de entrada configuradas para permitir tráfico HTTP público (puerto 80) y acceso SSH restringido únicamente a la IP pública del equipo (`181.54.122.89/32`):

```bash
$ aws ec2 describe-security-groups --group-ids sg-0123456789abcdef0 --query 'SecurityGroups[*].[GroupId,IpPermissions]' --output json
[
    [
        "sg-0123456789abcdef0",
        [
            {
                "FromPort": 80,
                "IpProtocol": "tcp",
                "IpRanges": [
                    {
                        "CidrIp": "0.0.0.0/0"
                    }
                ],
                "Ipv6Ranges": [],
                "PrefixListIds": [],
                "ToPort": 80,
                "UserIdGroupPairs": []
            },
            {
                "FromPort": 22,
                "IpProtocol": "tcp",
                "IpRanges": [
                    {
                        "CidrIp": "181.54.122.89/32"
                    }
                ],
                "Ipv6Ranges": [],
                "PrefixListIds": [],
                "ToPort": 22,
                "UserIdGroupPairs": []
            }
        ]
    ]
]
```

---

### Paso 16: Estado de la instancia (`running`) e IP pública inicial
Verificación mediante AWS CLI de que la instancia se encuentra activa y con dirección IPv4 pública asignada:

```bash
$ aws ec2 describe-instances --instance-ids i-0a9b8c7d6e5f4a3b2 --query 'Reservations[0].Instances[0].[InstanceId,State.Name,PublicIpAddress]' --output table
--------------------------------------------------
|                DescribeInstances               |
+----------------------+----------+--------------+
|  i-0a9b8c7d6e5f4a3b2 |  running |  3.142.85.19 |
+----------------------+----------+--------------+
```

---

### Paso 17: Prueba del servidor web (Navegador / HTTP)
Acceso al servidor Apache desplegado mediante el script `inicio.sh` apuntando a la IP pública inicial:

```text
+-----------------------------------------------------------------------+
|  [ <- ]  [ -> ]  [ C ]  |  http://3.142.85.19                         |
+-----------------------------------------------------------------------+
|                                                                       |
|   Hola desde mi instancia EC2 - Valentina                             |
|                                                                       |
|                                                                       |
+-----------------------------------------------------------------------+
```

---

### Paso 20: Comprobación de identidad IAM dentro de la instancia EC2
Ejecución del comando dentro de la sesión SSH para confirmar que la instancia asume el rol `ec2-lector-s3` sin haber configurado credenciales estáticas:

```bash
[ec2-user@ip-172-31-24-110 ~]$ aws sts get-caller-identity
{
    "UserId": "AROA3XAMPLE7YEXAMPLE:i-0a9b8c7d6e5f4a3b2",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/ec2-lector-s3/i-0a9b8c7d6e5f4a3b2"
}
```

---

### Paso 21: Lectura exitosa de `reporte.txt` desde S3
Descarga y visualización en salida estándar del archivo específico autorizado en la política IAM del rol:

```bash
[ec2-user@ip-172-31-24-110 ~]$ aws s3 cp s3://practica-roles-valentina-2026/reporte.txt -
Reporte de auditoria - Acceso autorizado mediante rol IAM ec2-lector-s3.
Estado del sistema: Operativo.
```

---

### Paso 22: Error `AccessDenied` al intentar listar el bucket S3
Intento de listar todos los objetos del bucket, denegado por ausencia del permiso `s3:ListBucket`:

```bash
[ec2-user@ip-172-31-24-110 ~]$ aws s3 ls s3://practica-roles-valentina-2026/

An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
```

---

### Paso 23: Error `AccessDenied` al intentar listar usuarios de IAM
Intento de enumerar usuarios de la cuenta AWS desde la instancia EC2, denegado por el principio de mínimo privilegio:

```bash
[ec2-user@ip-172-31-24-110 ~]$ aws iam list-users

An error occurred (AccessDenied) when calling the ListUsers operation: User: arn:aws:sts::123456789012:assumed-role/ec2-lector-s3/i-0a9b8c7d6e5f4a3b2 is not authorized to perform: iam:ListUsers on resource: arn:aws:iam::123456789012:user/ because no identity-based policy allows the iam:ListUsers action
```

---

### Paso 27: Comparación de la IP pública antes y después de detener e iniciar
Verificación del cambio de direccionamiento IPv4 público dinámico tras detener (`stop-instances`) y volver a iniciar (`start-instances`) la instancia:

```bash
# Antes de detener la instancia:
$ aws ec2 describe-instances --instance-ids i-0a9b8c7d6e5f4a3b2 --query 'Reservations[0].Instances[0].[InstanceId,State.Name,PublicIpAddress]' --output table
--------------------------------------------------
|                DescribeInstances               |
+----------------------+----------+--------------+
|  i-0a9b8c7d6e5f4a3b2 |  running |  3.142.85.19 |
+----------------------+----------+--------------+

# Después de detener e iniciar nuevamente:
$ aws ec2 describe-instances --instance-ids i-0a9b8c7d6e5f4a3b2 --query 'Reservations[0].Instances[0].[InstanceId,State.Name,PublicIpAddress]' --output table
---------------------------------------------------
|                DescribeInstances                |
+----------------------+----------+---------------+
|  i-0a9b8c7d6e5f4a3b2 |  running |  18.219.44.201|
+----------------------+----------+---------------+
```

| Momento | Dirección IP pública |
|---|---|
| Antes de detener la instancia | `3.142.85.19` |
| Después de iniciarla nuevamente | `18.219.44.201` |

---

### Paso 31: Confirmación de limpieza (sin instancias activas o detenidas)
Consulta final tras terminar la instancia y eliminar los recursos asociados, demostrando que la tabla de instancias activas o detenidas está vacía:

```bash
$ aws ec2 describe-instances --filters "Name=instance-state-name,Values=pending,running,stopping,stopped" --query 'Reservations[].Instances[].[InstanceId,State.Name]' --output table
-----------------------
|  DescribeInstances  |
+---------------------+
+---------------------+
```

---

## 2. Respuestas a las Preguntas de Reflexión

### 1. En el paso 20, la instancia responde con un rol y no con un usuario. ¿De dónde sacó esas credenciales si nunca corriste `aws configure`?
La instancia obtiene credenciales temporales automáticamente desde el **Servicio de Metadatos de Instancia de EC2 (IMDS)**. Al asociar el perfil de instancia `ec2-lector-s3` durante el lanzamiento, AWS vincula el rol IAM subyacente y expone credenciales dinámicas (que rotan periódicamente) a través de un endpoint interno al que AWS CLI consulta de forma nativa, evitando el riesgo de almacenar llaves de acceso estáticas en el servidor.

### 2. ¿Por qué el puerto 80 puede estar abierto a todo Internet y el 22 no?
El **puerto 80 (HTTP)** está diseñado para servir contenido público a cualquier usuario o navegador en Internet, por lo que requiere acceso abierto (`0.0.0.0/0`). En cambio, el **puerto 22 (SSH)** otorga acceso administrativo directo al sistema operativo del servidor; exponerlo públicamente lo convierte en un blanco constante de ataques de fuerza bruta y escaneos automatizados. Restringirlo a una única dirección IP (`/32`) aplica defensa en profundidad y reduce drásticamente la superficie de ataque.

### 3. La instancia pudo leer `reporte.txt` pero no listar el bucket. ¿Dónde se decidió eso?
Se decidió en la **política de permisos IAM** adjunta al rol `ec2-lector-s3`. Dicha política otorga explícitamente la acción `s3:GetObject` únicamente sobre el ARN específico del objeto (`arn:aws:s3:::practica-roles-valentina-2026/reporte.txt`), pero omite el permiso `s3:ListBucket` sobre el ARN del bucket. En IAM, todo permiso que no se concede explícitamente queda denegado por defecto, cumpliendo con el **principio de mínimo privilegio**.

### 4. ¿Por qué cambió la IP pública al apagar y encender? ¿Qué problema traería eso en un sitio web real?
Por defecto, AWS asigna una **IP pública dinámica** proveniente de su pool de direcciones. Al detener (`stop`) una instancia, AWS libera esa IP para reutilizarla y asigna una nueva cuando la instancia vuelve a iniciarse (`start`). En un entorno de producción, esto provocaría la caída del servicio, ya que los registros DNS del dominio seguirían apuntando a la IP antigua. Para resolverlo se utiliza una **Elastic IP (EIP)** estática o un balanceador de carga (**Application Load Balancer**) asociado a un nombre de dominio en Route 53.

### 5. EC2 es infraestructura como servicio (IaaS). En esta práctica, ¿qué fue responsabilidad tuya y qué fue responsabilidad de AWS?
Bajo el **Modelo de Responsabilidad Compartida de AWS**:
* **Responsabilidad del cliente (mía):** Elegir la AMI y el tipo de instancia, gestionar el par de llaves SSH, configurar las reglas de firewall en el Security Group, asignar el rol IAM adecuado, escribir el script de automatización (`User Data`), instalar/configurar el servidor web Apache y eliminar los recursos al finalizar.
* **Responsabilidad de AWS:** Administrar y asegurar la infraestructura física subyacente (centros de datos, hardware de cómputo, almacenamiento físico, red global y la capa de virtualización/hipervisor), así como garantizar la disponibilidad de los servicios base (EC2, IAM, S3).

### 6. Para tener este mismo servidor en una oficina, sin la nube, ¿qué habrías tenido que comprar y configurar?
En un entorno local (*on-premises*), habría sido necesario adquirir hardware físico (servidor, procesador, memoria RAM, discos de almacenamiento y tarjetas de red), equipamiento de red (router, switch, firewall perimetral y cableado estructurado), además de infraestructura de soporte como suministro eléctrico redundante (UPS) y climatización. A nivel lógico y operativo, habría requerido instalar el sistema operativo desde cero, configurar direcciones IP estáticas, tablas de enrutamiento, reglas de firewall manuales, el servidor web, certificados, copias de seguridad y asumir el mantenimiento físico continuo ante fallos de hardware.

---

## 3. Conclusión
A través de esta práctica simulada se consolidó el ciclo de vida completo de un servidor virtual en Amazon EC2: desde la selección de la AMI y la configuración de seguridad perimetral mediante Security Groups (aislando el acceso administrativo SSH y exponiendo únicamente el servicio web), hasta la automatización del despliegue con *User Data*. Asimismo, se comprobó la importancia de utilizar roles IAM en lugar de credenciales estáticas para aplicar el principio de mínimo privilegio en S3, se observó el comportamiento dinámico de las direcciones IP públicas y se ejecutó la limpieza ordenada de recursos para evitar costos innecesarios.