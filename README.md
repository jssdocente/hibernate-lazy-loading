### JPA e Hibernate: relaciones y atributos Lazy

En JPA los atributos y relaciones que forman parte de una entidad pueden ser obtenidas desde la base de datos de dos formas distintas: en el momento en que obtengamos la entidad (EAGER) o bien bajo demanda (LAZY), esto es, la primera vez que se acceda a dichas propiedades o relaciones. Este modo LAZY es posible en Hibernate gracias al uso de objetos [proxy](https://es.wikipedia.org/wiki/Proxy_(patrón_de_diseño)) que lanzan las consultas necesarias para la obtención de ese objeto, y su objetivo final es mejorar el rendimiento evitando obtener en nuestras aplicaciones datos que realmente no necesitamos (en ocasiones incluso podríamos terminar cargando casi toda la base de datos en memoria).



En este artículo vamos a revisar de forma práctica cómo hacer que Hibernate obtenga de forma LAZY las relaciones y atributos de nuestras entidades.

### Proyecto de ejemplo

Para “jugar” con la definición de relaciones en Hibernate -usaré la versión **5.4.21** de agosto de 2020- vamos a utilizar una aplicación Maven estándar cuyo código se reducirá simplemente a las entidades JPA que veremos más adelante. Obtendremos esta entidades desde una base de datos MySQL 5 en unos test de JUnit 4 cuyo único propósito será imprimir en el log las consultas realizadas.

La configuración de JPA se define en el fichero */META-INF/persistence.xml* tal y como ordena la especificación (aunque en Spring no es necesario).

```xml
<persistence xmlns="http://java.sun.com/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://java.sun.com/xml/ns/persistence http://java.sun.com/xml/ns/persistence/persistence_2_0.xsd"
             version="2.0">
    <persistence-unit name="BirdsPersistence" transaction-type="RESOURCE_LOCAL">

        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>

        <class>com.iesvi.blog.hibernatefetching.entities.Award</class>
        <class>com.iesvi.blog.hibernatefetching.entities.Bird</class>
        <class>com.iesvi.blog.hibernatefetching.entities.Breeder</class>
        <class>com.iesvi.blog.hibernatefetching.entities.Cage</class>
        <class>com.iesvi.blog.hibernatefetching.entities.Note</class>
        <class>com.iesvi.blog.hibernatefetching.entities.Specie</class>
        <class>com.iesvi.blog.hibernatefetching.entities.Treatment</class>

        <properties>
            <property name="javax.persistence.jdbc.url"
                      value="jdbc:mysql://localhost:3306/birds?autoReconnect=true&amp;useSSL=false&amp;serverTimezone=UTC"/>
            <property name="javax.persistence.jdbc.user" value="birds"/>
            <property name="javax.persistence.jdbc.password" value="password"/>
            <property name="hibernate.hbm2ddl.auto" value="update"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.MySQL55Dialect"/>
        </properties>

    </persistence-unit>
</persistence>
```

La configuración de la base de datos, incluyendo el usuario y el juego de datos de pruebas, se encuentra en el script *birds.sql*. He preparado un [Dockerfile](https://github.com/danielme-com/hibernate-lazy-loading/blob/master/Dockerfile) para poder crear y lanzar un contenedor con MySQL listo para ser utilizado por los tests. Se puede crear y levantar ejecutando los siguientes comandos desde el directorio raiz del proyecto.

```
docker build -t hibernate-lazy-mysql .
docker run -d -p 3306:3306 hibernate-lazy-mysql
```



### Modelo de entidades

Procedemos a crear un modelo de datos con entidades JPA que contengan a modo de ejemplo las relaciones más habituales.

![img](https://danielmedotcom.files.wordpress.com/2017/11/uml.png?w=656)



Este es el mapeo de relaciones inicialmente implementado:

- OneToOne unidireccional

   

  entre Bird y Specie.

  `@OneToOne`  `@JoinColumn``(name = ``"specie_id"``)``private` `Specie specie;`

- OneToOne bidireccional

   

  entre Bird y Cage.

  `@OneToOne``@JoinColumn``(name = ``"cage_id"``)``private` `Cage cage;`

  `@OneToOne``(mappedBy = ``"cage"``)``private` `Bird bird;`

- ManyToOne bidireccional

   

  entre Bird y Breeder.

  `@ManyToOne``@JoinColumn``(name = ``"breeder_id"``)``private` `Breeder breeder;`

  `@OneToMany``(mappedBy = ``"breeder"``)``private` `List<Bird> birds;`

- OneToMany bidireccional

   

  entre Bird y Award (equivalente al anterior)

  `@OneToMany``(mappedBy = ``"bird"``)``private` `List<Award> awards;`

  `@ManyToOne``@JoinColumn``(name = ``"bird_id"``)``private` `Bird bird;`

- OneToMany unidireccional

   

  entre Bird y Note. En este mapeo, si no se define la JoinColumn Hibernate crea una tabla intermedia.

  `@OneToMany``@JoinColumn``(name = ``"bird_id"``)``private` `List<Note> notes;`

- ManyToMany

   

  entre Bird y Treatment.

  `@ManyToMany``@JoinTable``(name = ``"bird_treatment"``, joinColumns = ``@JoinColumn``(name = ``"bird_id"``), ``  ``inverseJoinColumns = ``@JoinColumn``(name = ``"treatment_id"``))``  ``private` `Set<Treatment> treatments;`

  `@ManyToMany``(mappedBy = ``"treatments"``)``private` `Set<Bird> birds;`

- Por último, en la entidad Bird tenemos un atributo para almacenar un fichero, y queremos que este se obtenga de forma lazy cuando sea necesario.

  `@Basic``(fetch=FetchType.LAZY)``@Lob``private` `byte``[] picture;`



### Probando las relaciones

El proyecto de ejemplo nos permitirá comprobar el comportamiento por defecto de las relaciones y atributos definidos en nuestras entidades, y modificar este comportamiento para que su obtención pueda ser LAZY si así lo necesitamos. Me adelanto al resultado ya que Hibernate respeta lo especificado por el [estándar JPA](https://docs.oracle.com/javaee/7/api/index.html?javax/persistence/):

| **Relación** | **Obtención por defecto** |
| ------------ | ------------------------- |
| OneToMany    | LAZY                      |
| ManyToMany   | LAZY                      |
| OneToOne     | EAGER                     |
| ManyToOne    | EAGER                     |

Usaremos el siguiente test.

```java
package com.danielme.blog.hibernatefetching;

import com.iesvi.blog.hibernatefetching.entities.*;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import javax.persistence.EntityManager;
import javax.persistence.EntityManagerFactory;
import javax.persistence.Persistence;

public class FetchTest {

    private static final EntityManagerFactory entityManagerFactory = Persistence
            .createEntityManagerFactory("BirdsPersistence");
    private static EntityManager entityManager;

    private static final Logger log = LogManager.getLogger(FetchTest.class);

    @Before
    public void setup() {
        entityManager = entityManagerFactory.createEntityManager();
    }

    @After
    public void shutdown() {
        entityManager.close();
    }

    @Test
    public void testBird() {
        log.info("--------> Bird");
        Bird bird = entityManager.find(Bird.class, 1L);;
        log.info("--------> Specie unidirectional OneToOne");
        bird.getSpecie().getName();
        log.info("--------> Picture - Lob Basic");
        bird.getPicture();
        log.info("--------> Awards bidirectional OneToMany");
        bird.getAwards().size();
        log.info("--------> Treatments Many to many");
        bird.getTreatments().size();
        log.info("--------> Breeder bidirectional ManyToOne");
        bird.getBreeder().getName();
        log.info("--------> Cage bidirectional OneToOne");
        bird.getCage().getName();
        log.info("--------> Notes unidirectional OneToMany");
        bird.getNotes().size();
    }

    @Test
    public void testAward() {
        log.info("--------> Award");
        Award award = entityManager.find(Award.class, 1L);
        log.info("--------> Bird bidirectional @ManyToOne");
        award.getBird().getBand();
    }

    @Test
    public void testBreeder() {
        log.info("--------> Breeder");
        Breeder breeder = entityManager.find(Breeder.class, 1L);
        log.info("--------> Birds bidirectional OneToMany");
        breeder.getBirds().size();
    }

    @Test
    public void testCage() {
        log.info("--------> Cage");
        Cage cage = entityManager.find(Cage.class, 1L);
        log.info("--------> Bird bidirectional OneToOne");
        cage.getBird().getBand();
    }

    @Test
    public void testTeatments() {
        entityManager.find(Treatment.class, 1L).getBirds().size();
    }

}
```

El comportamiento de Hibernate lo vamos a revisar en el log, configurado con log4j2 del siguiente modo para que puedan verse las consultas ejecutadas contra la base de datos y los mensajes informativos que hemos puesto en el propio test.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
    <Appenders>
        <Console name="STDOUT" target="SYSTEM_OUT">
            <PatternLayout pattern="%c{1} %highlight{%level} - %msg%n"/>
        </Console>
    </Appenders>
    <Loggers>
        <Logger name="org.hibernate.SQL" level="debug"/>
        <Logger name="com.danielme" level="info"/>
        <Root level="warn">
            <AppenderRef ref="STDOUT"/>
        </Root>
    </Loggers>
</Configuration>
```

Ejecutando los tests podemos observar el comportamiento de cada mapeo.

### OneToOne

Todas las relaciones de este tipo se comportan por defecto de modo EAGER. Por ejemplo, al recuperarse un bird (primera operación de FetchTest#testBird()) nos traemos mediante joins la especie y la jaula:

```sql
select
        bird0_.id as id1_1_0_,
        bird0_.band as band2_1_0_,
        bird0_.breeder_id as breeder_4_1_0_,
        bird0_.cage_id as cage_id5_1_0_,
        bird0_.picture as picture3_1_0_,
        bird0_.specie_id as specie_i6_1_0_,
        breeder1_.id as id1_3_1_,
        breeder1_.name as name2_3_1_,
        cage2_.id as id1_4_2_,
        cage2_.name as name2_4_2_,
        specie3_.id as id1_6_3_,
        specie3_.name as name2_6_3_ 
    from
        bird bird0_ 
    left outer join
        breeder breeder1_ 
            on bird0_.breeder_id=breeder1_.id 
    left outer join
        cage cage2_ 
            on bird0_.cage_id=cage2_.id 
    left outer join
        specie specie3_ 
            on bird0_.specie_id=specie3_.id 
    where
        bird0_.id=?
```

Esto es debido a que Hibernate necesita saber si existe un objeto en el otro extremo de la relación. De hecho, aunque marquemos la relación con Fetch Lazy, Hibernate seguirá trayéndola.

```
@OneToOne``(fetch=FetchType.LAZY)
```

La anotación @OneToOne admite el atributo [optional](https://docs.oracle.com/javaee/7/api/javax/persistence/OneToOne.html#optional--) que permite indicar si la relación debe existir siempre o puede ser nula. Si indicamos que la relación no puede ser nula, Hibernate ya no tiene que comprobar su existencia y la relación puede obtenerse de forma Lazy. Esto sólo funciona en el extremo de la relación en el que tenemos la clave ajena, es decir, donde no tenemos definido el atributo mappedBy.

En nuestro ejemplo, podemos hacer Lazy la relación unidireccional entre Bird y Specie con el siguiente mapeo.

**Bird**

```java
@OneToOne(optional=false, fetch=FetchType.LAZY)
@JoinColumn(name = "specie_id")
private Specie specie;
```

Podemos comprobar en el log que el join con la especie que aparecia en la consulta anterior ha desaparecido.

```sql
select
       bird0_.id as id1_1_0_,
       bird0_.band as band2_1_0_,
       bird0_.breeder_id as breeder_4_1_0_,
       bird0_.cage_id as cage_id5_1_0_,
       bird0_.picture as picture3_1_0_,
       bird0_.specie_id as specie_i6_1_0_,
       breeder1_.id as id1_3_1_,
       breeder1_.name as name2_3_1_,
       cage2_.id as id1_4_2_,
       cage2_.name as name2_4_2_ 
   from
       bird bird0_ 
   left outer join
       breeder breeder1_ 
           on bird0_.breeder_id=breeder1_.id 
   left outer join
       cage cage2_ 
           on bird0_.cage_id=cage2_.id 
   where
       bird0_.id=?
```

Y que la especie se obtiene posteriormente cuando se demanda

```sql
FetchTest:51 - --------> Specie unidirectional OneToOne
SQL:92 - 
    select
        specie0_.id as id1_6_0_,
        specie0_.name as name2_6_0_ 
    from
        specie specie0_ 
    where
        specie0_.id=?
```

Lo mismo es aplicable a la relación de Bird con Cage pero sólo en el lado de Bird.

```sql
FetchTest:61 - --------> Cage bidirectional OneToOne
SQL:92 - 
    select
        cage0_.id as id1_4_0_,
        cage0_.name as name2_4_0_,
        bird1_.id as id1_1_1_,
        bird1_.band as band2_1_1_,
        bird1_.breeder_id as breeder_4_1_1_,
        bird1_.cage_id as cage_id5_1_1_,
        bird1_.picture as picture3_1_1_,
        bird1_.specie_id as specie_i6_1_1_,
        breeder2_.id as id1_3_2_,
        breeder2_.name as name2_3_2_ 
    from
        cage cage0_ 
    left outer join
        bird bird1_ 
            on cage0_.id=bird1_.cage_id 
    left outer join
        breeder breeder2_ 
            on bird1_.breeder_id=breeder2_.id 
    where
        cage0_.id=?
```

Para poder hacer Lazy el extremo Cage de la relación tenemos que recurrir a la anotación propia de Hibernate [LazyToOne](https://docs.jboss.org/hibernate/orm/5.2/javadocs/org/hibernate/annotations/LazyToOne.html).

```java
@OneToOne(fetch = FetchType.LAZY, optional = true, mappedBy = "cage")
@LazyToOne(LazyToOneOption.NO_PROXY)
private Bird bird;
```

Lamentablemente, tenemos que aplicar bytecode enhancement (lo veremos al final de este artículo) para hacer funcionar LazyToOne, aunque en versiones anteriores a Hibernate 5.1 se podía simplemente implementar en la entidad la interfaz [FieldHandled](https://docs.jboss.org/hibernate/orm/5.0/javadocs/org/hibernate/bytecode/internal/javassist/FieldHandled.html). [Aquí](http://luciamanescau.blogspot.com.es/2012/07/hibernate-onetoone-y-falsos-lazy.html) tenéis un ejemplo.

Como alternativa al uso del LazyToOne, debemos evaluar si realmente la relación debe ser bidireccional o si podemos hacer otro mapeo alternativo. Esta última solución podemos aplicarla en nuestro mapeo haciendo que el extremo de la relación en Cage sea un OneToMany y, por tanto, una colección que por defecto es LAZY. Desde fuera de la clase ocultaremos este “apaño” y trabajaremos con un objeto Bird si escribimos el set y get apropiado.

***cage***

```java
@OneToMany(mappedBy = "cage")
private List<Bird> birds;
 
public Bird getBird() {
       return birds != null && !birds.isEmpty() ? birds.get(0) : null;
   }
 
   public void setBird(Bird bird) {
       if (birds == null) {
           birds = new ArrayList<>();
       }
       birds.set(0, bird);
   }  
}
```

Con este mapeo tenemos una relación OneToOne bidireccional y LAZY en ambos extremos (esto es posible porque en Bird hemos mapeado Cage como optional=false)

**Bird**

```java
@OneToOne(optional=false, fetch=FetchType.LAZY)
@JoinColumn(name = "cage_id")
private Cage cage;
```

Pero, ¿qué sucede si marcamos un atributo como obligatorio (optional=false) pero realmente no lo es (puede valer nulo)? Pues que si establecemos el atributo a null Hibernate lanzará una excepción.

```
org.hibernate.PropertyValueException: not-null property references a null or transient value
```

Por tanto, si en la relación bidireccional que estamos revisando Cage puede ser nulo, en Bird utilizar OneToOne tal cual ya no es una opción si queremos que la relación en ese lado sea también LAZY. Nuevamente será necesario recurrir al bytecode enhancement o bien a un mapeo alternativo, que en este caso consiste en mapear la relación de Bird hacia Cage con @ManyToOne.

**Bird**

```java
@ManyToOne(fetch=FetchType.LAZY)
@JoinColumn(name = "cage_id")
private Cage cage;
```

De este modo, hemos “simulado” una relación one-to-one bidireccional que es tanto lazy como opcional en los dos extremos utilizando @OneToMany y @ManyToOne.

### OneToMany

Se comporta por defecto como LAZY, podemos verlo en la relación de Bird con Award y con Notas.

```sql
FetchTest:55 - --------> Awards bidirectional OneToMany
SQL:92 - 
    select
        awards0_.bird_id as bird_id3_0_0_,
        awards0_.id as id1_0_0_,
        awards0_.id as id1_0_1_,
        awards0_.bird_id as bird_id3_0_1_,
        awards0_.title as title2_0_1_ 
    from
        award awards0_ 
    where
        awards0_.bird_id=?
FetchTest:63 - --------> Notes unidirectional OneToMany
SQL:92 - 
    select
        notes0_.bird_id as bird_id3_5_0_,
        notes0_.id as id1_5_0_,
        notes0_.id as id1_5_1_,
        notes0_.name as name2_5_1_ 
    from
        note notes0_ 
    where
        notes0_.bird_id=?
```

### ManyToOne

Estas relaciones se comportan por defecto de forma EAGER y al obtenerse un Bird siempre se incluye el Breeder.

```sql
FetchTest:49 - --------> Bird
SQL:92 - 
    select
        bird0_.id as id1_1_0_,
        bird0_.band as band2_1_0_,
        bird0_.breeder_id as breeder_4_1_0_,
        bird0_.cage_id as cage_id5_1_0_,
        bird0_.picture as picture3_1_0_,
        bird0_.specie_id as specie_i6_1_0_,
        breeder1_.id as id1_3_1_,
        breeder1_.name as name2_3_1_ 
    from
        bird bird0_ 
    left outer join
        breeder breeder1_ 
            on bird0_.breeder_id=breeder1_.id 
    where
        bird0_.id=?
```

A diferencia de las relaciones OneToOne, en este tipo de relación sí se comporta como Lazy simplemente indicándolo.

```java
@ManyToOne(fetch=FetchType.LAZY)
@JoinColumn(name = "breeder_id")
private Breeder breeder;
```

```sql
FetchTest:49 - --------> Bird
SQL:92 - 
    select
        bird0_.id as id1_1_0_,
        bird0_.band as band2_1_0_,
        bird0_.breeder_id as breeder_4_1_0_,
        bird0_.cage_id as cage_id5_1_0_,
        bird0_.picture as picture3_1_0_,
        bird0_.specie_id as specie_i6_1_0_ 
    from
        bird bird0_ 
    where
        bird0_.id=?
FetchTest:59 - --------> Breeder bidirectional ManyToOne
SQL:92 - 
    select
        breeder0_.id as id1_3_0_,
        breeder0_.name as name2_3_0_ 
    from
        breeder breeder0_ 
    where
        breeder0_.id=?

```

Lo mismo es aplicable a la relación entre Award y Bird.

**award**

```java
@ManyToOne(fetch=FetchType.LAZY)
@JoinColumn(name = "bird_id")
private Bird bird;
```



### ManyToMany

Se obtienen por defecto en modo LAZY.

```sql
FetchTest:57 - --------> Treatments Many to many
SQL:92 - 
    select
        treatments0_.bird_id as bird_id1_2_0_,
        treatments0_.treatment_id as treatmen2_2_0_,
        treatment1_.id as id1_7_1_,
        treatment1_.name as name2_7_1_ 
    from
        bird_treatment treatments0_ 
    inner join
        treatment treatment1_ 
            on treatments0_.treatment_id=treatment1_.id 
    where
        treatments0_.bird_id=?
```

### Atributos

Aunque los atributos o propiedades pueden configurarse como LAZY con la anotación Basic, Hibernate los obtiene siempre.

```sql
FetchTest:49 - --------> Bird
SQL:92 - 
    select
        bird0_.id as id1_1_0_,
        bird0_.band as band2_1_0_,
        bird0_.breeder_id as breeder_4_1_0_,
        bird0_.cage_id as cage_id5_1_0_,
        bird0_.picture as picture3_1_0_,
        bird0_.specie_id as specie_i6_1_0_ 
    from
        bird bird0_ 
    where
        bird0_.id=?
```

Esto supone un problema si tenemos atributos que contengan binarios o textos muy grandes y que por tanto generalmente sólo queremos obtener de forma explícita cuando sea estrictamente necesario. Aunque, en mi opinión, los ficheros directamente es mejor no guardarlos en una base de datos.

En nuestro ejemplo cada Bird puede tener una imagen, y si simplemente obtenemos un listado de los mismos estaremos obteniendo esos ficheros con el problema de rendimiento que esto puede suponer.

Para habilitar la obtención LAZY en los atributos, nuevamente tenemos recurrir al bytecode enhancement. Pero tenemos la alternativa de crear una entidad aparte para mapear los ficheros y obtenerlos con una relación que admita LAZY, como por ejemplo ManyToOne.

A modo de ejemplo, vamos a crear una entidad Picture y establecer una relación unidireccional, opcional y LAZY desde Bird hasta Picture.

![img](https://danielmedotcom.files.wordpress.com/2017/11/jpa-manytoone.gif?w=656)

```java
package com.danielme.blog.hibernatefetching.entities;
 
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Lob;
import javax.persistence.Table;
 
@Entity
@Table(name = "picture")
public class Picture {
 
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
     
    @Lob   
        private byte[] picture;
 
    public Long getId() {
        return id;
    }
 
    public void setId(Long id) {
        this.id = id;
    }
 
    public byte[] getPicture() {
        return picture;
    }
 
    public void setPicture(byte[] picture) {
        this.picture = picture;
    }   
     
}
```

Para relacionar la entidad Bird con Picture se puede utilizar un mapeo ManyToOne ya que según vimos anteriormente un OneToOne opcionales no puede ser LAZY.

```java
@ManyToOne(fetch=FetchType.LAZY)
@JoinColumn(name = "picture_id")
private Picture picture;
```

### Bytecode Enhancement

Esta técnica permite procesar el [bytecode](https://es.wikipedia.org/wiki/Bytecode_Java) generado por el compilador para añadir de forma automática nuevas funcionalidades al .class que finalmente ejecutaremos. Hibernate desde la versión 4.2.9 proporciona un [plugin para Maven](https://docs.jboss.org/hibernate/orm/5.0/topical/html/bytecode/BytecodeEnhancement.html), también disponible para Gradle y Ant, que permite aplicar automáticamente Bytecode enhancement a nuestro modelo mediante el uso de la librería [javassist](http://jboss-javassist.github.io/javassist/). Para utilizarlo tan sólo tenemos que añadirlo a nuestro pom activando la opción *enableLazyInitialization* que es la que necesitamos para poder realizar la carga Lazy de cualquier relación.

```xml
<plugin>
    <groupId>org.hibernate.orm.tooling</groupId>
    <artifactId>hibernate-enhance-maven-plugin</artifactId>
    <version>${hibernate.version}</version>
    <executions>
        <execution>
            <phase>compile</phase>
            <goals>
                <goal>enhance</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <enableLazyInitialization>true</enableLazyInitialization>
    </configuration>
</plugin>
```

Si decompilamos nuestras entidades podemos ver los cambios realizados por este plugin.

[Cage.class](https://danielme.com/2018/01/03/hibernatejpa-relaciones-y-atributos-lazy-bytecode-enhacement/#)

Ahora ya funcionará *@Basic(fetch=FetchType.LAZY)*

```
FetchTest:48 - ``--------> Bird``SQL:92 - ``  ``select``    ``bird0_.id ``as` `id1_1_0_,``    ``bird0_.band ``as` `band2_1_0_,``    ``bird0_.breeder_id ``as` `breeder_4_1_0_,``    ``bird0_.cage_id ``as` `cage_id5_1_0_,``    ``bird0_.specie_id ``as` `specie_i6_1_0_ ``  ``from``    ``bird bird0_ ``  ``where``    ``bird0_.id=?``FetchTest:52 - ``--------> Picture - Lob Basic``SQL:92 - ``  ``select``    ``bird_.picture ``as` `picture3_1_ ``  ``from``    ``bird bird_ ``  ``where``    ``bird_.id=?
```

Y *@LazyToOne(LazyToOneOption.NO_PROXY)* en Cage.

```sql
FetchTest:48 - --------> Bird
SQL:92 - 
    select
        bird0_.id as id1_1_0_,
        bird0_.band as band2_1_0_,
        bird0_.breeder_id as breeder_4_1_0_,
        bird0_.cage_id as cage_id5_1_0_,
        bird0_.specie_id as specie_i6_1_0_ 
    from
        bird bird0_ 
    where
        bird0_.id=?
FetchTest:52 - --------> Picture - Lob Basic
SQL:92 - 
    select
        bird_.picture as picture3_1_ 
    from
        bird bird_ 
    where
        bird_.id=?
```

Sin embargo, la aplicación del bytecode enhancement tiene un impacto grande en los demás mapeos:

Ahora todas las relaciones que no sean colecciones y queramos cargar como LAZY deben anotarse con @LazyToOne con la opción NO_PROXY

La primera vez que se solicite una propiedad LAZY de la entidad que no sea una colección, se traen todos los atributos LAZY. Por ejemplo, si la primera relación LAZY que obtenemos de Bird es Specie:

```sql
FetchTest:84 - --------> Cage
SQL:92 - 
    select
        cage0_.id as id1_4_0_,
        cage0_.name as name2_4_0_ 
    from
        cage cage0_ 
    where
        cage0_.id=?
FetchTest:86 - --------> Bird bidirectional OneToOne
SQL:92 - 
    select
        bird0_.id as id1_1_0_,
        bird0_.band as band2_1_0_,
        bird0_.breeder_id as breeder_4_1_0_,
        bird0_.cage_id as cage_id5_1_0_,
        bird0_.specie_id as specie_i6_1_0_ 
    from
        bird bird0_ 
    where
        bird0_.cage_id=?

```

Para evitar esto tenemos que recurrir a la anotación [LazyGroup](https://docs.jboss.org/hibernate/orm/5.2/javadocs/org/hibernate/annotations/LazyGroup.html) que permite agrupar los atributos/relaciones LAZY que deben obtenerse la primera vez que se solicite obtener uno de los atributos de dicho grupo. Lamentablemente esta anotación sólo está disponible a partir de Hibernate 5.1 tal y recoge [la entrada en JIRA](https://hibernate.atlassian.net/browse/HHH-10267).

Teniendo en cuenta estos dos puntos, podemos reescribir la entidad Bird para que todas sus relaciones sean LAZY si aplicamos bytecode enhancement con los siguientes mapeos. Para cada atributo se define un LazyGroup que permite obtener cada uno de forma independiente.

```sql
FetchTest:50 - --------> Specie unidirectional OneToOne
SQL:92 - 
    select
        bird_.breeder_id as breeder_4_1_,
        bird_.cage_id as cage_id5_1_,
        bird_.picture as picture3_1_,
        bird_.specie_id as specie_i6_1_ 
    from
        bird bird_ 
    where
        bird_.id=?
SQL:92 - 
    select
        breeder0_.id as id1_3_0_,
        breeder0_.name as name2_3_0_ 
    from
        breeder breeder0_ 
    where
        breeder0_.id=?
SQL:92 - 
    select
        cage0_.id as id1_4_0_,
        cage0_.name as name2_4_0_ 
    from
        cage cage0_ 
    where
        cage0_.id=?
SQL:92 - 
    select
        specie0_.id as id1_6_0_,
        specie0_.name as name2_6_0_ 
    from
        specie specie0_ 
    where
        specie0_.id=?
```

### Conclusiones

Gracias a bytecode enhancement podemos conseguir que cualquier relación y atributo se obtenga de forma LAZY aunque este no sea el comportamiento implementado por Hibernate. No obstante, esta técnica tiene efectos colaterales por lo que mi recomendación personal es intentar en la medida de lo posible utilizar los mapeos alternativos que hemos visto como por ejemplo simular las relaciones OneToOne como OneToMany y ManyToOne. Y en todos los casos, **evitar mapear aquellas relaciones que realmente no necesitemos**.
