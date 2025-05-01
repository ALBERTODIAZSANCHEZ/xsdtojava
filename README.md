# Generación de Clases Java a partir de XSD

![Java](https://img.shields.io/badge/Java-11+-blue)
![Maven](https://img.shields.io/badge/Maven-3.6+-red)
![JAXB](https://img.shields.io/badge/JAXB-2.3.3-orange)

## Descripción

Este repositorio muestra cómo generar automáticamente clases Java desde esquemas XML Schema (XSD) en tres escenarios:
1. **Esquema Simple**  
2. **Esquema Complejo**  
3. **Esquema Compuesto**

Se incluyen ejemplos sin Maven (usando la distribución independiente de JAXB) y con Maven (utilizando `jaxb2-maven-plugin`).

---

## Contenido del Repositorio

```text
├── pom.xml
├── README.md
└── src
    └── main
        └── resources
            ├── Person.xsd
            ├── maven-4.0.0.xsd
            ├── binding-maven.xml
            ├── Customer.xsd
            ├── catalog.cat
            └── imports
                ├── Address.xsd
                └── PhoneNumber.xsd
```

---

## Prerrequisitos

- Java 11 o superior instalado y en el `PATH`.
- Maven 3.6 o superior (para la variante con Maven).
- Acceso a internet (solo para descargar el schema de Maven).
- Terminal (PowerShell, CMD, Bash, VS Code Terminal, IntelliJ IDEA Terminal).

---

## Guía paso a paso

### 1. Clonar el repositorio

```bash
git clone https://github.com/tuUsuario/xsd_practica.git
cd xsd_practica
```

### 2. Caso A – XSD Simple

#### 2.1. Generación sin Maven

Descarga la distribución de JAXB RI (por ejemplo, versión 2.3.3), descomprímela y ejecuta:

```bash
~/jaxb-ri-2.3.3/bin/xjc   -d out/simple   -p com.example.person   src/main/resources/Person.xsd
```

Las clases generadas se ubicarán en `out/simple/com/example/person/`.

#### 2.2. Generación con Maven

```bash
mvn clean compile jaxb2:xjc
```

El plugin configurado en `pom.xml` generará las clases en `target/generated-sources/jaxb/`.

---

### 3. Caso B – XSD Complejo

1. Descarga el esquema de Maven si no está en `src/main/resources/maven-4.0.0.xsd`:
   ```bash
   wget -O src/main/resources/maven-4.0.0.xsd https://maven.apache.org/xsd/maven-4.0.0.xsd
   ```
2. Ejecuta:
   ```bash
   xjc -d out/complex        -b src/main/resources/binding-maven.xml        src/main/resources/maven-4.0.0.xsd
   ```
Las clases se generarán en `out/complex/com/example/maven/pom/`.

---

### 4. Caso C – XSD Compuesto

```bash
xjc -d out/composed     -catalog src/main/resources/catalog.cat     src/main/resources/Customer.xsd
```

Verás que se generan clases para `customer`, `address` y `phoneNumber` en carpetas separadas dentro de `out/composed/`.

---

## Estructura de paquetes generados

- `com.example.person`  
- `com.example.generated` (si usas Maven con paquete único)  
- `com.example.maven.pom`  
- `com.example.customer`  
- `com.example.address`  
- `com.example.phone`

---

## Errores encontrados

Durante todo el proceso me he topado con estos errores, y te los cuento en “lenguaje llano” como si fuera yo el que habla:

1. **`xjc: command not found`**  
   Al intentar generar las clases con  
   ```bash
   xjc -d out -p com.example.person Person.xsd
   ```  
   me salió “command not found” porque a partir de Java 11 ya no viene incluida la herramienta `xjc` en el JDK. Tuve que descargar la distribución independiente de JAXB para poder usarla.

2. **Falta de paquetes `javax.xml.bind` al usar Maven**  
   Con el plugin `jaxb2-maven-plugin` en el `pom.xml`, tras ejecutar  
   ```bash
   mvn clean compile jaxb2:xjc
   ```  
   apareció un error del tipo  
   ```
   package javax.xml.bind does not exist
   ```  
   Eso ocurre porque, aunque añadí la dependencia de la API de JAXB, el plugin no encontraba internamente la implementación de `xjc`. La solución fue incluir dentro del plugin las dependencias `jaxb-xjc` y `jaxb-impl`.

3. **Colisión de nombres en el XSD complejo**  
   Al generar el esquema de Maven (`maven-4.0.0.xsd`) me di cuenta de que había un tipo y un elemento ambos llamados `Properties`. Esto provocaba un error de “collision in the ObjectFactory class”, porque no sabe a cuál referirse. Lo resolví creando un fichero de binding donde renombraba el tipo complejo a `PropertiesType`.

4. **Importaciones no resueltas en el XSD compuesto**  
   Cuando pasé a trabajar con varios XSD (Customer + Address + PhoneNumber), `xjc` no encontraba las rutas relativas de los imports y lanzaba errores como  
   ```
   Cannot resolve the name 'addr:address'
   ```  
   Para solucionarlo tuve que usar un catálogo XML que mapea cada `schemaLocation` a su fichero local, y pasarle al comando la opción `-catalog`.

En cada uno de estos puntos, documenté el fallo, busqué la causa y probé la solución. Así aprendí no solo a generar las clases, sino también a gestionar dependencias, conflictos de nombres y referencias cruzadas.

---


---

## Soluciones a los errores

A continuación incluyo cómo resolví cada uno de los problemas encontrados:

1. **`xjc: command not found`**  
   **Solución:**  
   - Descargué la distribución independiente de JAXB (JAXB RI 2.3.3).  
   - Descomprimí el ZIP y desde la terminal ejecuté:
     ```bash
     ~/jaxb-ri-2.3.3/bin/xjc        -d out/simple        -p com.example.person        src/main/resources/Person.xsd
     ```
   - Así `xjc` funciona aunque no esté en el JDK.

2. **Falta de paquetes `javax.xml.bind` al usar Maven**  
   **Solución:**  
   - En el `<plugin>` `jaxb2-maven-plugin` de `pom.xml`, añadí las dependencias:
     ```xml
     <dependencies>
       <dependency>
         <groupId>org.glassfish.jaxb</groupId>
         <artifactId>jaxb-xjc</artifactId>
         <version>2.3.3</version>
       </dependency>
       <dependency>
         <groupId>com.sun.xml.bind</groupId>
         <artifactId>jaxb-impl</artifactId>
         <version>2.3.3</version>
       </dependency>
     </dependencies>
     ```
   - Después, `mvn clean compile jaxb2:xjc` generó las clases correctamente.

3. **Colisión de nombres en el XSD complejo**  
   **Solución:**  
   - Creé un fichero de binding `binding-maven.xml` en `src/main/resources`:
     ```xml
     <jxb:bindings xmlns:jxb="http://java.sun.com/xml/ns/jaxb"
                   xmlns:xs="http://www.w3.org/2001/XMLSchema"
                   jxb:version="2.1">
       <jxb:bindings schemaLocation="maven-4.0.0.xsd" node="/xs:schema">
         <jxb:schemaBindings>
           <jxb:package name="com.example.maven.pom"/>
         </jxb:schemaBindings>
         <jxb:bindings node="xs:complexType[@name='Properties']">
           <jxb:class name="PropertiesType"/>
         </jxb:bindings>
       </jxb:bindings>
     </jxb:bindings>
     ```
   - Luego ejecuté:
     ```bash
     xjc -d out/complex          -b src/main/resources/binding-maven.xml          src/main/resources/maven-4.0.0.xsd
     ```

4. **Importaciones no resueltas en el XSD compuesto**  
   **Solución:**  
   - Añadí un catálogo `catalog.cat` en `src/main/resources`:
     ```
     SYSTEM "imports/Address.xsd"     "imports/Address.xsd"
     SYSTEM "imports/PhoneNumber.xsd" "imports/PhoneNumber.xsd"
     ```
   - Y generé con:
     ```bash
     xjc -d out/composed          -catalog src/main/resources/catalog.cat          src/main/resources/Customer.xsd
     ```

Con estas soluciones, la generación de clases Java desde todos los XSD funciona sin errores.


## Contribuir

1. Haz un **fork** del repositorio.
2. Crea una rama (`git checkout -b feature/nueva-prueba`).
3. Realiza tus cambios y **commit** (`git commit -m 'Agregar caso de prueba XSD adicional'`).
4. Envía un **pull request**.

---

## Licencia

Este proyecto se distribuye bajo la licencia MIT.
