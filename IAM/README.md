# 🛡️ Laboratorio AWS IAM: Gestión de Identidades, Permisos y Menor Privilegio

Documentación y evidencia práctica de administración de usuarios, políticas gestionadas, perfiles de AWS CLI y el principio de menor privilegio en **Amazon Web Services (IAM)**.

- **Entorno:** `Coder@rwadmin-9053301005`
- **Cuenta AWS:** `642890121638`
- **Región:** `us-east-2`

## 📋 Estructura del Repositorio

Las capturas se almacenan dentro de una carpeta llamada `imagenes/`:

```text
.
├── README.md
└── imagenes/
    ├── paso3-crear-usuario.png
    ├── paso6-renombrar-usuario.png
    ├── paso6-asignar-politica.png
    ├── paso12-accion-permitida.png
    └── paso19-eliminar-usuario.png
```

## 🚀 Desarrollo y Evidencias por Pasos

### Paso 3: Creación del usuario inicial (`juan-perez`)

Se crea el usuario inicial en IAM sin políticas ni grupos asociados.

```text
Coder@rwadmin-9053301005:~$ aws iam create-user --user-name juan-perez
{
    "User": {
        "Path": "/",
        "UserName": "juan-perez",
        "UserId": "AIDAZLLZ4YGTBZMKCIPZX",
        "Arn": "arn:aws:iam::642890121638:user/juan-perez",
        "CreateDate": "2026-11-20T22:44:53+00:00"
    }
}
```

![Creación del usuario juan-perez](imagenes/paso3-crear-usuario.png)

### Paso 6: Renombramiento a `lector-juan` y asignación de `AmazonEC2ReadOnlyAccess`

Se actualiza el nombre del usuario a `lector-juan` y se adjunta la política administrada de AWS para lectura en EC2.

```text
Coder@rwadmin-9053301005:~$ aws iam update-user --user-name juan-perez --new-user-name lector-juan
Coder@rwadmin-9053301005:~$ aws iam attach-user-policy --user-name lector-juan --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
Coder@rwadmin-9053301005:~$ aws iam list-attached-user-policies --user-name lector-juan
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonEC2ReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess"
        }
    ]
}
```

![Renombramiento del usuario a lector-juan](imagenes/paso6-renombrar-usuario.png)

![Asignación de AmazonEC2ReadOnlyAccess](imagenes/paso6-asignar-politica.png)

### Paso 8: Configuración de perfil `lector` y verificación de identidad

> **⚠️ Nota de seguridad:** Las claves secretas (`SecretAccessKey`) han sido omitidas de este documento y de las capturas para evitar la exposición de credenciales.

```text
Coder@rwadmin-9053301005:~$ aws sts get-caller-identity --profile lector
{
    "UserId": "AIDAZLLZ4YGTBZMKCIPZX",
    "Account": "642890121638",
    "Arn": "arn:aws:iam::642890121638:user/lector-juan"
}
```

### Paso 12: Prueba de acción permitida (`ec2:DescribeInstances`)

Al tener adjunta la política `AmazonEC2ReadOnlyAccess`, la consulta de instancias EC2 responde correctamente sin errores de autorización.

```text
Coder@rwadmin-9053301005:~$ aws ec2 describe-instances --profile lector
{
    "Reservations": []
}
```

![Acción EC2 permitida](imagenes/paso12-accion-permitida.png)

### Paso 13: Prueba de acción no permitida (`iam:ListUsers`)

Siguiendo el principio de menor privilegio, cualquier intento de listar usuarios de IAM es bloqueado explícitamente con `AccessDenied`.

```text
Coder@rwadmin-9053301005:~$ aws iam list-users --profile lector

aws: [ERROR]: An error occurred (AccessDenied) when calling the ListUsers operation: User: arn:aws:iam::642890121638:user/lector-juan is not authorized to perform: iam:ListUsers on resource: arn:aws:iam::642890121638:user/ because no identity-based policy allows the iam:ListUsers action.
```

### Paso 15: Revocación de permisos y verificación de bloqueo

Tras desvincular la política `AmazonEC2ReadOnlyAccess` desde el perfil administrador, el usuario pierde inmediatamente el acceso de lectura en EC2 (`UnauthorizedOperation`).

```text
Coder@rwadmin-9053301005:~$ aws iam detach-user-policy --user-name lector-juan --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
Coder@rwadmin-9053301005:~$ aws ec2 describe-instances --profile lector

aws: [ERROR]: An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation. User: arn:aws:iam::642890121638:user/lector-juan is not authorized to perform: ec2:DescribeInstances because no identity-based policy allows the ec2:DescribeInstances action
```

### Paso 19: Eliminación ordenada de credenciales, usuario y perfil local

Se valida la restricción de IAM (`DeleteConflict`) al intentar borrar un usuario con llaves activas, y posteriormente se ejecuta la limpieza en el orden exigido: llave de acceso, usuario y perfil local en `~/.aws/`.

```text
# Reto: Intentar eliminar el usuario antes de borrar su llave de acceso
Coder@rwadmin-9053301005:~$ aws iam delete-user --user-name lector-juan
aws: [ERROR]: An error occurred (DeleteConflict) when calling the DeleteUser operation: Cannot delete entity, must delete access keys first.

# Eliminación correcta en orden
Coder@rwadmin-9053301005:~$ aws iam delete-access-key --user-name lector-juan --access-key-id AKIAZLLZ4YGTBHP2HC5V
Coder@rwadmin-9053301005:~$ aws iam list-attached-user-policies --user-name lector-juan
{
    "AttachedPolicies": []
}
Coder@rwadmin-9053301005:~$ aws iam delete-user --user-name lector-juan
Coder@rwadmin-9053301005:~$ aws configure list --profile lector
The config profile (lector) could not be found
```

![Eliminación de credenciales y limpieza](imagenes/paso19-eliminar-usuario.png)

## 💡 Preguntas de Reflexión

1. **¿Qué podía hacer el usuario justo después de crearlo, antes de adjuntarle una política? ¿Por qué?**

No podía realizar ninguna acción sobre ningún recurso o servicio de AWS. Esto ocurre porque AWS IAM funciona bajo el modelo de **denegación implícita por defecto** (*implicit deny*): toda identidad nueva nace sin permisos y requiere que se le asigne explícitamente una política que autorice cada operación.

2. **En el error del paso 13, ¿qué tres datos te dice el mensaje?**

- **Quién intentó ejecutar la acción (Principal):** `arn:aws:iam::642890121638:user/lector-juan`.
- **Qué acción fue rechazada (Action):** `iam:ListUsers`.
- **Sobre qué recurso y el motivo (Resource & Reason):** Sobre el recurso `arn:aws:iam::642890121638:user/` y aclara que ninguna política basada en identidad permite la acción `iam:ListUsers`.

3. **¿Por qué cambió el ARN al renombrar el usuario?**

Porque el ARN (*Amazon Resource Name*) incorpora el nombre legible del usuario al final de la ruta (`...:user/juan-perez` pasó a `...:user/lector-juan`). Sin embargo, su identificador interno único (`UserId`: `AIDAZLLZ4YGTBZMKCIPZX`) se mantuvo igual, ya que AWS utiliza ese ID interno para conservar la integridad de los permisos aunque cambie el nombre visible.

4. **¿Por qué AWS obliga a borrar las llaves antes de borrar el usuario?**

Es una medida de control de seguridad para evitar que queden credenciales activas o "huérfanas" (sin un dueño asociado) almacenadas en archivos locales, repositorios o scripts. Al exigir eliminar primero las llaves y políticas asociadas, AWS garantiza que no queden vías de acceso residuales en la cuenta.

5. **¿Qué pasaría si le hubieras dado `AdministratorAccess` en vez de solo lectura? ¿En qué caso real estaría justificado?**

Con `AdministratorAccess`, el usuario habría tenido control total sobre toda la cuenta de AWS: crear o borrar servidores, alterar redes, modificar facturación e incluso gestionar otros administradores o eliminar mi propio usuario. En un entorno real, solo estaría justificado asignárselo a un ingeniero DevOps, arquitecto cloud o administrador principal encargado de gestionar la infraestructura completa (idealmente mediante roles temporales con MFA en lugar de llaves fijas).