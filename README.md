#[JPA 101](http://tomee.apache.org/jpa-concepts.html) 

If there's one thing you have to understand to successfully use JPA (Java Persistence API) it's the concept of a Cache. Almost everything boils down to the Cache at one point or another. Unfortunately the Cache is an internal thing and not exposed via the JPA API classes, so it not easy to touch or feel from a coding perspective.

Here's a quick cheat sheet of the JPA world:

* A Cache is a copy of data, copy meaning pulled from but living outside the database.
* Flushing a Cache is the act of putting modified data back into the database.
* A PersistenceContext is essentially a Cache. It also tends to have it's own non-shared database connection.
* An EntityManager represents a PersistenceContext (and therefore a Cache)
* An EntityManagerFactory creates an EntityManager (and therefore a PersistenceContext/Cache)

###Comparing RESOURCE_LOCAL and JTA persistence contexts

With <persistence-unit transaction-type="RESOURCE_LOCAL"> you are responsible for EntityManager (PersistenceContext/Cache) creating and tracking...

* You must use the EntityManagerFactory to get an EntityManager
* The resulting EntityManager instance is a PersistenceContext/Cache
* An EntityManagerFactory can be injected via the **@PersistenceUnit** annotation only (**NOT @PersistenceContext**)
* You are not allowed to use @PersistenceContext to refer to a unit of type RESOURCE_LOCAL
* You must use the **EntityTransaction** API to **begin/commit** around every call to your EntityManger
* Calling entityManagerFactory.createEntityManager() twice results in two separate EntityManager instances and therefor two separate PersistenceContexts/Caches.
* It is almost never a good idea to have more than one instance of an EntityManager in use (don't create a second one unless you've destroyed the first)

With <persistence-unit transaction-type="JTA"> the container will do EntityManager (PersistenceContext/Cache) creating and tracking...

* You can **NOT use the EntityManagerFactory** to get an **EntityManager**
* You can only get an EntityManager supplied by the container
* An EntityManager can be injected via the @PersistenceContext annotation only (not @PersistenceUnit)
* You are not allowed to use @PersistenceUnit to refer to a unit of type JTA
* The EntityManager given by the container is a reference to the PersistenceContext/Cache associated with a JTA Transaction.
* If no JTA transaction is in progress, the EntityManager cannot be used because there is no PersistenceContext/Cache.
* Everyone with an EntityManager reference to the same unit in the same transaction will automatically have a reference to the same PersistenceContext/Cache
* The PersistenceContext/Cache is flushed and cleared at JTA commit time

###Valid RESOURCE_LOCAL Unit usage

Servlets and EJBs can use RESOURCE_LOCAL persistence units through the EntityManagerFactory as follows:

        <?xml version="1.0" encoding="UTF-8" ?>
        <persistence xmlns="http://java.sun.com/xml/ns/persistence" version="1.0">
        
          <!-- Tutorial "unit" -->
          <persistence-unit name="Tutorial" transaction-type="RESOURCE_LOCAL">
            <non-jta-data-source>myNonJtaDataSource</non-jta-data-source>
            <class>org.superbiz.jpa.Account</class>
          </persistence-unit>
        
        </persistence>

And referenced as follows

        import javax.persistence.EntityManagerFactory;
        import javax.persistence.EntityManager;
        import javax.persistence.EntityTransaction;
        import javax.persistence.PersistenceUnit;
        
        public class MyEjbOrServlet ... {
        
            @PersistenceUnit(unitName="Tutorial")
            private EntityManagerFactory factory;
        
            // Proper exception handling left out for simplicity
            public void ejbMethodOrServletServiceMethod() throws Exception {
                EntityManager entityManager = factory.createEntityManager();
        
                EntityTransaction entityTransaction = entityManager.getTransaction();
        
                entityTransaction.begin();
        
                Account account = entityManager.find(Account.class, 12345);
        
                account.setBalance(5000);
        
                entityTransaction.commit();
            }
            ...
        }

###Valid JTA Unit usage:

EJBs can use JTA persistence units through the EntityManager as follows:

        <?xml version="1.0" encoding="UTF-8" ?>
        <persistence xmlns="http://java.sun.com/xml/ns/persistence" version="1.0">
        
          <!-- Tutorial "unit" -->
          <persistence-unit name="Tutorial" transaction-type="JTA">
            <jta-data-source>myJtaDataSource</jta-data-source>
            <non-jta-data-source>myNonJtaDataSource</non-jta-data-source>
            <class>org.superbiz.jpa.Account</class>
          </persistence-unit>
        
        </persistence>
    
    
And referenced as follows:
    
        
        import javax.ejb.Stateless;
        import javax.ejb.TransactionAttribute;
        import javax.ejb.TransactionAttributeType;
        import javax.persistence.EntityManager;
        import javax.persistence.PersistenceContext;
        
        @Stateless
        public class MyEjb implements MyEjbInterface {
        
            @PersistenceContext(unitName = "Tutorial")
            private EntityManager entityManager;
        
            // Proper exception handling left out for simplicity
            @TransactionAttribute(TransactionAttributeType.REQUIRED)
            public void ejbMethod() throws Exception {
        
            Account account = entityManager.find(Account.class, 12345);
        
            account.setBalance(5000);
        
            }
        }


*

____________________________________

###EntityManager Interface
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
    
* Another example with resource local.
**persistence.xml**

        <persistence xmlns="http://java.sun.com/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:schemaLocation="http://java.sun.com/xml/ns/persistence persistence_1_0.xsd" version="1.0">
            <persistence-unit name="acme" transaction-type="RESOURCE_LOCAL">
                <non-jta-data-source>amce</non-jta-data-source>
            </persistence-unit>
        </persistence>

The transaction:

        EntityManager em = createEntityManager();
        em.getTransaction().begin();
        Employee employee = em.find(Employee.class, id);
        employee.setSalary(employee.getSalary() + 1000);
        em.getTransaction().commit();
        em.close();

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

*Example with JTA resource: 
persistence.xml
        <persistence xmlns="http://java.sun.com/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://java.sun.com/xml/ns/persistence persistence_1_0.xsd" version="1.0">
            <persistence-unit name="acme" transaction-type="JTA">
                <jta-data-source>acme</jta-data-source>
            </persistence-unit>
        </persistence>

The transaction:

        UserTransaction transaction = (UserTransaction)new InitialContext().lookup("java:comp/UserTransaction");
        transaction.begin();
        EntityManager em = getEntityManager();
        Employee employee = em.find(Employee.class, id);
        employee.setSalary(employee.getSalary() + 1000);
        transaction.commit();




###Entity Transient fields:
All fields not annotated **javax.persistence.Transient** will be persisted to the data store.

    //This field will not be persisted in Database
    @Transient 
    private int example; 
    
This has a semantic difference from the keyword **transient**.  The **@Transient** annotation tells the JPA provider to not persist any (non-transient) attribute. The other tells the serialization framework to not serialize an attribute. You might want to have a @Transient property and still serialize it.



###Bean Validation:
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

###CRUD Operations

##### Persisting Entity Instances (CREATE)

The following method performs a persist operation:

        @PersistenceContext
        EntityManager em;
        //...
        public LineItem createLineItem(CustomerOrder order, Product product, int quantity) {
            
            LineItem li = new LineItem(order, product, quantity);
            order.getLineItems().add(li);
            em.persist(li);
            return li;
        }     

The persist operation is propagated to all entities related to the calling entity that
have the cascade element set to ALL or PERSIST in the relationship annotation:
        
        @OneToMany(cascade=ALL, mappedBy="order")
        public Collection<LineItem> getLineItems() {
            return lineItems;
        }



##### Finding Entity Instances (READ)
The EntityManager.find method is used to look up entities in the data store by the
entity's **primary key**:

    @PersistenceContext
    EntityManager em;
    
    public void enterOrder(int custID, CustomerOrder newOrder) {
        Customer cust = em.find(Customer.class, custID);
        cust.getOrders().add(newOrder);
        newOrder.setCustomer(cust);
    }

##### UPDATE Entity Instances
        em.merge();

##### DELETE Entity Instances
In the following example, all LineItem entities associated with the order are also
removed, as CustomerOrder.getLineItems has cascade=ALL set in the relationship
annotation:

        public void removeOrder(Integer orderId) {
            try {
                CustomerOrder order = em.find(CustomerOrder.class, orderId);
                em.remove(order);
            }catch...



###Persistence Units
A persistence unit defines a set of all entity classes that are managed by EntityManager instances in an application. This set of entity classes represents the data contained within a single data store.

Persistence units are defined by the persistence.xml configuration file. The following is an example persistence.xml file:

        <persistence>
            <persistence-unit name="OrderManagement">
                <description>This unit manages orders and customers. It does not rely on any 
                     vendor-specific features and can therefore be deployed to any persistence provider.
                </description>
                <jta-data-source>jdbc/MyOrderDB</jta-data-source>
                <jar-file>MyOrderApp.jar</jar-file>
                <class>com.widgets.CustomerOrder</class>
                <class>com.widgets.Customer</class>
            </persistence-unit>
        </persistence>

The jta-data-source (for JTA-aware data sources) and non-jta-data-source (for non-JTA-aware data sources) 
elements specify the global JNDI name of the data source to be used by the container.

Persistent units can be packaged as part of a WAR or EJB JAR file or can be packaged
as a JAR file that can then be included in an WAR or EAR file.
*  If you package the persistent unit as a set of classes in an EJB JAR file,
persistence.xml should be put in the EJB JAR's META-INF directory.
* If you package the persistence unit as a set of classes in a WAR file,
persistence.xml should be located in the WAR file's WEB-INF/classes/META-INF
directory.
* If you package the persistence unit in a JAR file that will be included in a WAR or
EAR file, the JAR file should be located in either
   *  The WEB-INF/lib directory of a WAR
   *  Or the EAR file's library directory

###  Schema Creation
The persistence provider can be configured to automatically create the database tables,
load data into the tables, and remove the tables during application deployment using
standard properties in the application's deployment descriptor. **These tasks are
typically used during the development phase of a release, not against a production
database.**

The following is an example of a persistence.xml deployment descriptor that
specifies that the provider should drop all database artifacts using a provided script,
create the artifacts with a provided script, and load data from a provided script when
the application is deployed:

        <?xml version="1.0" encoding="UTF-8"?>
        <persistence version="2.1" xmlns="http://xmlns.jcp.org/xml/ns/persistence"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
        http://xmlns.jcp.org/xml/ns/persistence/persistence_2_1.xsd">
            <persistence-unit name="examplePU" transaction-type="JTA">
                <jta-data-source>java:global/ExampleDataSource</jta-data-source>
                <properties>
                    <property name="javax.persistence.schema-generation.database.action"
                    value="drop-and-create"/>
                    <property name="javax.persistence.schema-generation.create-source"
                    value="script"/>
                    <property name="javax.persistence.schema-generation.create-script-source"
                    value="META-INF/sql/create.sql" />
                    <property name="javax.persistence.sql-load-script-source"
                    value="META-INF/sql/data.sql" />
                    <property name="javax.persistence.schema-generation.drop-source"
                    value="script" />
                    <property name="javax.persistence.schema-generation.drop-script-source"
                    value="META-INF/sql/drop.sql" />
                </properties>
            </persistence-unit>
        </persistence>

| Setting          | Description                                                                    |
| ---------------- |:------------------------------------------------------------------------------:|
| create           | The provider will create the database artifacts on application deployment. The artifacts will remain unchanged after application redeployment.                                                    |
| none             | No schema creation or deletion will take place                                 |
| drop-and-create  |  Any artifacts in the database will be deleted, and the provider will create the database artifacts on deployment.                     |
| drop             |  Any artifacts in the database will be deleted on application deployment.      |

