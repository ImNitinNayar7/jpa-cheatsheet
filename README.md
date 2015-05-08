
#####Entity Transient fields:
All fields not annotated **javax.persistence.Transient** will be persisted to the data store.

    //This field will not be persisted in Database
    @Transient 
    private int example; 
    
This has a semantic difference from the keyword **transient**.  The **@Transient** annotation tells the JPA provider to not persist any (non-transient) attribute. The other tells the serialization framework to not serialize an attribute. You might want to have a @Transient property and still serialize it.

#####Bean Validation:
Constraints applied to an Entity in its persistent fields(instance variables).

    @Entity
    public class Contact implements Serializable {
    
    private static final long serialVersionUID = 1L;
    
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    
    @NotNull
    protected String firstName;
    
    @NotNull
    protected String lastName;
    
    @Pattern(regexp="[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\\."
        +"[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*@"
        +"(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?",
             message="{invalid.email}")
    protected String email;
    
    @Pattern(regexp="^\\(?(\\d{3})\\)?[- ]?(\\d{3})[- ]?(\\d{4})$",
             message="{invalid.phonenumber}")
    protected String mobilePhone;
    
    @Pattern(regexp="^\\(?(\\d{3})\\)?[- ]?(\\d{3})[- ]?(\\d{4})$",
             message="{invalid.phonenumber}")
    protected String homePhone;
    
    @Temporal(javax.persistence.TemporalType.DATE)
    @Past
    protected Date birthday;
    //...
    }
    
The @NotNull annotation on the firstName and lastName fields specifies that those fields are now required. If a new Contact instance is created where firstName or lastName have not been initialized, Bean Validation will **throw a validation error** (. Similarly, if a previously created instance of Contact has been modified so that firstName or lastName are null, a validation error will be thrown.

The email field has a @Pattern constraint applied to it, with a complicated regular expression that matches most valid email addresses. If the value of email doesn’t match this regular expression, a validation error will be thrown.

The homePhone and mobilePhone fields have the same @Pattern constraints. The regular expression matches 10 digit telephone numbers in the United States and Canada of the form (xxx) xxx–xxxx.

The birthday field is annotated with the @Past constraint, which ensures that the value of birthday must be in the past.

#####EntityManager Interface
The EntityManager API **creates** and **removes** persistent entity instances, **finds** entities
by the entity's primary key, and allows queries to be run on entities.

A transaction-type of JTA assumes that a JTA data source will be provided—either as specified by the jta-data-source element or provided by the container. In general, in Java EE environments, a transaction-type of RESOURCE_LOCAL assumes that a non-JTA datasource will be provided.

In a **Java EE environment**, if this element is not specified, the default is **JTA**. In a **Java SE** environment, if this element is not specified, the default is **RESOURCE_LOCAL.**

If you deploy the following persistence.xml:

    <persistence>
      <persistence-unit name="integration" transaction-type="RESOURCE_LOCAL">
        <class>...AnEntity</class>
        <exclude-unlisted-classes>true</exclude-unlisted-classes>
        <properties>
          <property name="javax.persistence.jdbc.url" value="jdbc:derby:memory:testDB;create=true"/>
          <property name="javax.persistence.jdbc.driver" value="org.apache.derby.jdbc.EmbeddedDriver"/>
          <property name="eclipselink.ddl-generation" value="create-tables"/>
        </properties>
      </persistence-unit>
    </persistence>

...into a Java EE application server, you will also have to manage both: the EntityManager and it's JTA-transaction by yourself. This will end up with lots of plumbing, for example:

    @PersistenceUnit
    EntityManagerFactory emf;
    EntityManager em;
    
    @Resource
    UserTransaction utx;
    ...
    emf;
    em = emf.createEntityManager();
    
    try {
      utx.begin();
      em.persist(SomeEntity);
      em.merge(AnotherEntity);
      em.remove(ThirdEntity);
      utx.commit();
    
    } catch (Exception e) {
      utx.rollback();
    }

Instead of **RESOURCE_LOCAL**, you should use the **JTA** setting in production. With the JTA setting you don't have to specify the JDBC-connection and use a pre-configured JTA data source (usually a JNDI in the Wildfly, Glassfish) instead: 


    <persistence>
      <persistence-unit name="prod" transaction-type="JTA">
        <jta-data-source>jdbc/sample</jta-data-source>
        <properties>
          <property name="eclipselink.ddl-generation" value="drop-and-create-tables"/>
        </properties>
      </persistence-unit>
    </persistence>
    

To obtain an EntityManager instance, inject the entity manager into the application
component:

    @PersistenceContext
    EntityManager em;


This way the Container(Web Server) EntityManager instance's persistence context is automatically propagated by the container to all application components that use the EntityManager instance within a single JTA (Java Transaction API) transaction.
