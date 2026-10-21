# CLF - AWS CloudFormation

Este laboratorio utiliza AWS CloudFormation para desplegar y administrar una API HTTP como una unidad. La actividad comienza con el diagnóstico y la corrección de un despliegue fallido desde AWS Management Console, continúa con una actualización fallida y su recuperación mediante change sets, incorpora la detección de drift y termina con una nueva actualización del mismo stack desde AWS CloudShell. AWS Management Console se utilizará con la interfaz en inglés.

## Resultados esperados

Al completar el laboratorio podrá:

1. Distinguir un template de CloudFormation, un stack y los recursos administrados por el stack
2. Reconocer parameters, conditions, resources, referencias y outputs dentro de un template YAML
3. Distinguir los recursos declarados en un template de los recursos auxiliares que AWS crea automáticamente
4. Diagnosticar una creación fallida mediante los eventos del stack y corregir directamente el template
5. Crear un stack desde AWS Management Console y seguir su avance mediante los eventos registrados por CloudFormation
6. Utilizar los recursos y outputs de un stack para comprobar una API HTTP
7. Crear, calcular, revisar y ejecutar un change set antes de actualizar una infraestructura
8. Diagnosticar una actualización fallida mediante los eventos del stack y reconocer el rollback realizado por CloudFormation
9. Corregir el template y recuperar la actualización mediante un nuevo change set
10. Introducir una modificación fuera de CloudFormation y detectar el drift producido en el stack
11. Editar, validar, actualizar y consultar un stack mediante AWS CLI
12. Eliminar como una unidad los recursos creados por el template

## Preparación previa

Antes del laboratorio:

- Confirmar que su cuenta AWS personal se encuentra operativa y que su identidad administrativa de uso regular está protegida con MFA y tiene asociada la managed policy `AdministratorAccess`
- Descargar [`infrastructure.yaml`](infrastructure.yaml)
- Confirmar que completó el laboratorio de AWS CLI o que puede utilizar AWS CloudShell, `--query` y los formatos de salida de AWS CLI

## Contexto

Publicar una Lambda function mediante una API HTTP requiere crear varios recursos y relacionarlos mediante identificadores y ARN. API Gateway necesita una API, un stage, routes e integrations. Lambda necesita una function y un execution role. Además, una resource-based policy debe permitir que API Gateway invoque la function correspondiente. Crear cada elemento por separado obliga a mantener manualmente estas relaciones y a repetir el procedimiento cada vez que se necesita desplegar nuevamente la infraestructura.

CloudFormation permite describir el conjunto completo en un template y administrarlo como un stack. A partir de esta definición, el servicio determina las dependencias, crea los recursos en el orden necesario y conserva el estado requerido para actualizarlos o eliminarlos posteriormente. De esta manera, el despliegue deja de depender de una secuencia extensa de operaciones manuales y puede repetirse a partir del mismo archivo.

En este laboratorio se desplegará una API HTTP que utiliza una Lambda function para entregar el estado del sistema e incluye un bucket S3 cuyo nombre se incorpora a la configuración y a la respuesta de la function. El primer despliegue no podrá completarse, por lo que será necesario analizar los eventos del stack y corregir el template antes de crear la infraestructura. Posteriormente, se intentará ampliar el sistema con una segunda route que devuelve un mensaje recibido mediante la URL. Esta actualización tampoco podrá completarse inicialmente y permitirá observar el rollback realizado por CloudFormation antes de calcular un nuevo change set con una segunda corrección.

Una HTTP API requiere recursos relacionados, como routes, integrations y stages. Estos recursos pueden declararse explícitamente en un template, aunque CloudFormation también permite que algunos sean desplegados automáticamente a partir de una definición simplificada. El template disponible describe el sistema completo como una única infraestructura.

## Actividad

### 1. Analizar el template

#### 1.1 Establecer la Región

1. Ingresar a AWS Management Console con la identidad administrativa de uso regular
2. Seleccionar **South America (São Paulo)** y comprobar que la Región corresponda a `sa-east-1`

> Todos los recursos del laboratorio se crearán en `sa-east-1`. Mantener esta Región durante toda la actividad.

#### 1.2 Reconocer la definición

Abrir [`infrastructure.yaml`](infrastructure.yaml) y reconocer sus secciones principales:

- `Parameters` recibe el RUT normalizado, el mensaje entregado por la route de estado y la decisión de habilitar la route adicional
- `Conditions` determina qué recursos condicionales deben formar parte del stack
- `Resources` declara el execution role, el bucket, las Lambda functions, la API, los permisos de invocación y los recursos necesarios para incorporar la segunda route
- `Outputs` publica las URL y los nombres que permiten comprobar la infraestructura desplegada

En `Resources`, identificar los siguientes mecanismos de relación:

- `!Ref` entrega el valor de un parameter o el identificador principal de un recurso
- `!GetAtt` obtiene un atributo de un recurso, como el ARN de una Lambda function
- `!Sub` construye nombres, ARN y URL a partir de parameters, pseudo parameters y recursos
- `Condition` incorpora un recurso solamente cuando se cumple la condición indicada

Observar que el template no necesita conocer previamente los identificadores físicos que AWS asignará a la API, las integrations o el bucket. CloudFormation obtiene esos valores durante el despliegue y resuelve las referencias antes de crear cada recurso dependiente.

`ArchiveBucket` tampoco especifica un nombre ni configuraciones adicionales. S3 asigna un nombre físico y aplica los valores por omisión de los buckets nuevos, incluyendo cifrado SSE-S3, bloqueo de acceso público y ACL deshabilitadas. El nombre asignado quedará disponible mediante un output.

### 2. Crear el stack desde AWS Management Console

#### 2.1 Iniciar la creación

1. Buscar y abrir **CloudFormation**
2. Ingresar a **Stacks**
3. Seleccionar **Create stack → With new resources (standard)**
4. En **Prepare template**, seleccionar **Choose an existing template**
5. En **Specify template**, seleccionar **Upload a template file**
6. Seleccionar **Choose file**, cargar `infrastructure.yaml` y continuar con **Next**
7. En **Stack name**, escribir `tel351-clf`
8. Configurar los parameters:

   | Parameter | Valor |
   | --- | --- |
   | `StudentId` | Su RUT normalizado, sin puntos ni guion y con `k` minúscula cuando corresponda |
   | `StatusMessage` | Mantener el valor por omisión |
   | `EnableEcho` | `false` |

9. Seleccionar **Next**
10. Mantener las opciones del stack con sus valores por omisión y seleccionar **Next**
11. Revisar la configuración
12. En **Capabilities**, reconocer que el template creará un recurso IAM y marcar la confirmación solicitada por CloudFormation
13. Seleccionar **Submit**

El template solicita la creación del execution role requerido por Lambda como parte del stack. La confirmación reconoce esta capacidad del template; no reemplaza la revisión de los permisos declarados.

#### 2.2 Diagnosticar la creación fallida

1. Abrir la pestaña **Events** del stack
2. Actualizar la vista hasta que aparezca un evento `CREATE_FAILED`
3. Identificar el **Logical ID** del recurso que no pudo crearse
4. Abrir el detalle del evento y revisar la información disponible
5. Identificar en los registros qué configuración impidió crear el recurso
6. Seguir los eventos hasta que el stack alcance `ROLLBACK_COMPLETE`
7. Reconocer los recursos que alcanzaron `CREATE_COMPLETE` antes de ser eliminados durante el rollback

#### 2.3 Corregir el template y crear el stack

1. Abrir el archivo `infrastructure.yaml` descargado antes del laboratorio
2. Buscar el recurso identificado mediante los eventos del stack
3. Corregir la configuración que impidió crear el recurso
4. Guardar el archivo sin modificar su extensión ni su indentación
5. Regresar al stack que alcanzó `ROLLBACK_COMPLETE`
6. Seleccionar **Delete** y confirmar la eliminación
7. Esperar hasta que el stack desaparezca de la lista de stacks activos
8. Repetir el procedimiento de **Create stack → With new resources (standard)**
9. Cargar la versión corregida de `infrastructure.yaml`
10. Utilizar el mismo nombre del stack y los mismos valores de los parameters
11. Mantener las opciones con sus valores por omisión
12. Marcar nuevamente la confirmación solicitada en **Capabilities**
13. Seleccionar **Submit**
14. Abrir **Events** y seguir la creación de los recursos
15. Esperar hasta que el stack alcance `CREATE_COMPLETE`

### 3. Revisar la infraestructura creada

#### 3.1 Revisar el stack

1. Reconocer en **Events** los registros correspondientes a la API, la Lambda function, el permiso de invocación, el execution role y el bucket
2. Abrir **Resources** y comparar el **Logical ID** utilizado por el template con el **Physical ID** asignado por AWS
3. Abrir **Parameters** y comprobar los valores utilizados durante el despliegue
4. Abrir **Template** y comprobar que CloudFormation conserva la definición asociada con el stack
5. Abrir **Outputs** y reconocer los valores publicados por el stack

#### 3.2 Revisar la API

1. Buscar y abrir **API Gateway**
2. Abrir la API creada por el stack
3. Reconocer la route `GET /status`, su Lambda integration y el stage `$default`
4. Regresar al stack en CloudFormation y abrir la pestaña **Outputs**
5. Copiar el valor de `StatusUrl`
6. Abrir la URL en una pestaña nueva del navegador
7. Comprobar que la respuesta contiene:
   - `status` igual a `operational`
   - el mensaje configurado mediante `StatusMessage`
   - el RUT normalizado entregado mediante `StudentId`
   - el nombre del bucket creado por el stack
8. Copiar `ApiUrl`, agregar `/echo/cloudformation` al final y abrir la URL resultante
9. Comprobar que API Gateway responde que la route no existe

La API se encuentra disponible, pero `EnableEchoCondition` controla la creación de la Lambda function, la integration, la route y el permiso necesarios para atender `GET /echo/{message}`.

### 4. Incorporar una route mediante un change set

#### 4.1 Calcular el cambio

1. Regresar al stack en CloudFormation
2. Seleccionar **Stack actions → Create change set**
3. Seleccionar **Use existing template**
4. En `EnableEcho`, seleccionar `true`
5. Mantener los demás parameters con sus valores actuales
6. Continuar hasta configurar el change set
7. Escribir `enable-echo` como nombre
8. Crear el change set y esperar mientras CloudFormation calcula las diferencias
9. Comprobar que el estado del change set alcance `CREATE_COMPLETE`

#### 4.2 Ejecutar y analizar el cambio

1. Revisar **Changes** antes de ejecutar el change set
2. Identificar los recursos que CloudFormation propone agregar
3. Comprobar que los recursos corresponden a la function, la integration, la route y el permiso de invocación asociados con `/echo/{message}`
4. Seleccionar **Execute change set** y confirmar la operación
5. Abrir **Events** y seguir los cambios de estado del stack y sus recursos
6. Identificar el primer recurso que no pudo completar la actualización
7. Abrir el detalle del evento e identificar qué configuración impidió crear el recurso
8. Seguir los eventos hasta que el stack alcance `UPDATE_ROLLBACK_COMPLETE`
9. Reconocer los recursos agregados durante la actualización que fueron eliminados durante el rollback
10. Abrir **Outputs** y comprobar que `StatusUrl` continúa disponible y que `EchoUrl` no fue incorporado
11. Abrir nuevamente `StatusUrl` y comprobar que la configuración anterior del sistema permanece operativa

#### 4.3 Corregir el template y calcular un nuevo cambio

1. Abrir el archivo `infrastructure.yaml` descargado antes del laboratorio
2. Buscar el recurso identificado mediante los eventos del stack
3. Corregir la configuración que impidió crear el recurso
4. Guardar el archivo sin modificar su extensión ni su indentación
5. Regresar al stack en CloudFormation
6. Seleccionar **Stack actions → Create change set**
7. Seleccionar **Replace existing template**, cargar la versión corregida de `infrastructure.yaml` y continuar con **Next**
8. Configurar `EnableEcho` con el valor `true` y mantener los demás parameters con sus valores actuales
9. Continuar hasta configurar el change set
10. Escribir `enable-echo-corrected` como nombre
11. Crear el change set y esperar hasta que alcance `CREATE_COMPLETE`
12. Revisar **Changes** y comprobar que la actualización propuesta incorpora los recursos esperados para la segunda route

#### 4.4 Ejecutar y comprobar el nuevo cambio

1. Seleccionar **Execute change set** y confirmar la operación
2. Abrir **Events** y esperar hasta que el stack alcance `UPDATE_COMPLETE`
3. Abrir **Outputs** y copiar `EchoUrl`
4. Reemplazar `{message}` por `cloudformation` y abrir la URL resultante
5. Comprobar que la respuesta contiene el mensaje incorporado en la URL

El primer change set permitió observar que una actualización puede modificar varios recursos antes de encontrar una falla. El rollback devolvió el stack a su configuración operativa anterior, mientras que el segundo change set aplicó la definición corregida sobre el mismo stack.

### 5. Introducir y detectar drift

El drift corresponde a una diferencia entre la configuración esperada por CloudFormation y el estado actual de un recurso. Puede producirse cuando un recurso administrado por un stack es modificado directamente desde otro servicio, sin actualizar el template ni los parameters del stack.

#### 5.1 Modificar un recurso fuera de CloudFormation

1. Abrir **Resources** en el stack de CloudFormation
2. Buscar `StatusFunction` y seleccionar su **Physical ID** para abrir la Lambda function
3. Ingresar a **Configuration → Environment variables**
4. Seleccionar **Edit**
5. Cambiar solamente el valor de `MESSAGE` por `Changed-outside-CloudFormation`
6. Guardar la configuración
7. Abrir nuevamente `StatusUrl` y comprobar que la respuesta refleja el valor modificado

La Lambda function ya no coincide con la configuración conservada por CloudFormation, aunque el stack continúa en estado `UPDATE_COMPLETE`. Las modificaciones realizadas fuera de CloudFormation no cambian por sí solas el estado operativo del stack.

#### 5.2 Detectar el drift

1. Regresar al stack en CloudFormation
2. Seleccionar **Stack actions → Detect drift**
3. Esperar hasta que termine la detección
4. Seleccionar **Stack actions → View drift results**
5. Reconocer el estado de drift informado para el stack
6. Identificar el recurso modificado y abrir su detalle
7. Comparar los valores esperados y actuales informados por CloudFormation
8. Relacionar la diferencia detectada con la modificación realizada directamente en Lambda

No corregir todavía el recurso desde Lambda. La siguiente actualización volverá a aplicar sobre la infraestructura la configuración declarada por el stack.

### 6. Actualizar el stack mediante AWS CLI

#### 6.1 Preparar CloudShell

1. Buscar `CloudShell` en el buscador de AWS Management Console y abrir el servicio
2. Comprobar que la pestaña utiliza `sa-east-1`
3. En **Actions**, seleccionar **Upload file** y cargar la versión de `infrastructure.yaml` corregida durante las secciones 2 y 4
4. Comprobar que el archivo se encuentra disponible:

   ```bash
   ls -l infrastructure.yaml
   ```

5. Definir el nombre del stack y el mismo RUT normalizado utilizado durante su creación, reemplazando el valor de ejemplo:

   ```bash
   STUDENT_ID=12345678k
   STACK_NAME=tel351-clf
   ```

#### 6.2 Editar y validar el template

En esta etapa se modificará la memoria asignada a `StatusFunction`. La actualización debe conservar la Lambda function y los demás recursos del stack, reemplazar el valor introducido directamente en Lambda por el valor indicado mediante `StatusMessage` y mantener operativa la route `GET /status`. El resultado se comprobará después del despliegue mediante la respuesta de la API y la configuración informada por Lambda.

Abrir el archivo desde CloudShell:

```bash
nano infrastructure.yaml
```

Buscar `StatusFunction` y agregar la propiedad siguiente dentro de `Properties`, al mismo nivel que `Runtime`, `Handler` y `Role`:

```yaml
MemorySize: 256
```

Guardar el archivo y salir del editor.

Ejecutar:

```bash
aws cloudformation validate-template \
    --template-body file://infrastructure.yaml
```

Reconocer en la respuesta:

- los parameters aceptados por el template
- la capability IAM requerida por sus recursos
- la descripción del template

La validación comprueba la estructura reconocida por CloudFormation, pero no demuestra que la infraestructura se comporte correctamente después del despliegue.

#### 6.3 Actualizar el stack

Utilizar el template modificado para actualizar el stack existente:

```bash
aws cloudformation deploy \
    --template-file infrastructure.yaml \
    --stack-name "$STACK_NAME" \
    --parameter-overrides \
        StudentId="$STUDENT_ID" \
        EnableEcho=true \
        StatusMessage=Updated-from-AWS-CLI \
    --capabilities CAPABILITY_IAM
```

`aws cloudformation deploy` crea y ejecuta el change set necesario para que el stack existente alcance la configuración indicada. `CAPABILITY_IAM` cumple desde AWS CLI la confirmación realizada durante la creación del stack en la consola.

#### 6.4 Consultar el resultado

Consultar el estado del stack:

```bash
aws cloudformation describe-stacks \
    --stack-name "$STACK_NAME" \
    --query 'Stacks[0].{Stack:StackName,Status:StackStatus}' \
    --output table
```

Mostrar sus outputs:

```bash
aws cloudformation describe-stacks \
    --stack-name "$STACK_NAME" \
    --query 'Stacks[0].Outputs[].{Output:OutputKey,Value:OutputValue}' \
    --output table
```

Mostrar los recursos administrados:

```bash
aws cloudformation list-stack-resources \
    --stack-name "$STACK_NAME" \
    --query 'StackResourceSummaries[].{LogicalId:LogicalResourceId,Type:ResourceType,Status:ResourceStatus}' \
    --output table
```

Obtener `StatusUrl` desde los outputs y comprobar la API:

```bash
STATUS_URL=$(aws cloudformation describe-stacks \
    --stack-name "$STACK_NAME" \
    --query "Stacks[0].Outputs[?OutputKey=='StatusUrl'].OutputValue | [0]" \
    --output text)

curl "$STATUS_URL"
```

Comprobar que `message` contiene `Updated-from-AWS-CLI`.

Consultar la configuración de la Lambda function modificada:

```bash
aws lambda get-function-configuration \
    --function-name tel351-clf-status \
    --query '{Function:FunctionName,MemoryMiB:MemorySize}' \
    --output table
```

Comprobar que `MemoryMiB` contiene `256`.

#### 6.5 Comprobar la reversión del drift

1. Regresar al stack en CloudFormation
2. Seleccionar **Stack actions → Detect drift**
3. Esperar hasta que termine la detección
4. Seleccionar **Stack actions → View drift results**
5. Comprobar que `StatusFunction` vuelve a encontrarse `IN_SYNC` con la definición administrada por el stack
6. Abrir nuevamente `StatusUrl` y comprobar que `message` contiene `Updated-from-AWS-CLI` en lugar del valor configurado directamente en Lambda

La actualización volvió a aplicar la configuración declarada y revirtió el cambio realizado directamente en Lambda. CloudFormation no solicitó una confirmación adicional para reemplazar esa modificación externa. Por esta razón, los cambios manuales sobre recursos administrados por un stack pueden perderse durante una actualización si no se incorporan previamente al template o a sus parameters.

## Limpieza

La limpieza se realiza después del bloque de laboratorio y es obligatoria.

1. Regresar al stack `tel351-clf` en CloudFormation
2. Seleccionar **Delete** y confirmar la eliminación
3. Abrir **Events** y seguir el procedimiento hasta que el stack desaparezca de la lista de stacks activos
4. Buscar y abrir **CloudWatch**
5. Ingresar a **Logs → Log groups**
6. Buscar y eliminar `/aws/lambda/tel351-clf-status`
7. Buscar y eliminar `/aws/lambda/tel351-clf-echo`
