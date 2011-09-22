# SMX Demo

## Building

    mvn install

This will install it to your local maven repository.


## Prepare the datasource

The demo uses the *viiper-ds* datasource which must be made available to the
servicemix environment. Since this is dependent on the environment where we
would deploy our packages, this would be something prepared by the sysadmins.

In this case we need to use the oracle driver.

    karaf@root> osgi:install wrap:mvn:ojdbc/ojdbc5/10.2.0.1.0

Please check the actual version number as this is updated from time to time.

To define a datasource to be used, in this case *viiper_ds* we can create a small
spring configuration file which we deploy to the hot deploy folder <SMX_HOME>/deploy.

    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:osgi="http://www.springframework.org/schema/osgi"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
                               http://www.springframework.org/schema/osgi http://www.springframework.org/schema/osgi/spring-osgi.xsd
                               ">
      <bean id="viiper-ds" class="oracle.jdbc.pool.OracleDataSource" destroy-method="close">
        <property name="connectionCachingEnabled" value="true" />
        <property name="URL">
          <value>jdbc:oracle:thin:@orpmelei.apps.elex.be:1521:PMELEI</value>
        </property>
        <property name="user">
          <value>********</value>
        </property>
        <property name="password">
          <value>********</value>
        </property>
        <property name="connectionCacheProperties">
          <value>
            MinLimit:1
            MaxLimit:5
            InitialLimit:1
            ConnectionWaitTimeout:120
            InactivityTimeout:180
            ValidateConnection:true
          </value>
        </property>
      </bean>

      <osgi:service ref="viiper-ds" interface="javax.sql.DataSource"/>

    </beans>

This creates a small connection-pool using the Oracle provided implementation in the
driver package. For other drivers which do not come with a connection pool I
recommend to wrap them in the C3P0 connection-pool, but this demo uses oracle, so
its not needed.

Lets check if it worked :

    karaf@root> list
    ...
    [ 222] [Active     ] [            ] [       ] [   60] wrap_mvn_ojdbc_ojdbc5_11.2.0.1.0 (0)
    ...
    [ 239] [Active     ] [            ] [       ] [   60] oracle_pool.xml (0.0.0)

I called the spring xml file *oracle_pool.xml* amd we see it is *Active*. Similar for
the wrapped jdbc driver in the oracle provided ojdbc jar.

Note: we can refer to these bens in our camel spring configuration :

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:osgi="http://www.springframework.org/schema/osgi"
           xmlns:camel="http://camel.apache.org/schema/spring"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
                               http://www.springframework.org/schema/osgi http://www.springframework.org/schema/osgi/spring-osgi.xsd
                               http://camel.apache.org/schema/spring       http://camel.apache.org/schema/spring/camel-spring.xsd">

        <camel:camelContext>
            <camel:routeBuilder ref="route"/>
        </camel:camelContext>

        <bean id="route" class="com.melexis.Route"/>

        <osgi:reference id="viiper-ds" interface="javax.sql.DataSource" bean-name="viiper-ds"/>

        <bean id="transactionId" class="com.melexis.Route$TransactionId"/>
    </beans>

The *<osgi:reference ...>* element will import the datasource. To select the right datasource
use the *bean-name* attribute to specify the id in the datasource definition (which corresponds
to the content of the *ref* attribute in the *osgi:service* element). The id of the
*osgi:reference* element is the name the bean will be known in our application.

## Prepare the camel components on the container

Since we want to reuse as much as possible existing lego-blocks there is a big box of
blocks included in the ServiceMix container. However before we can use them, we must
take them out of the storage box.

In this case we need the additional blocks 'camel-stream' and 'camel-jdbc'.

    karaf@root> features:install camel-stream
    karaf@root> features:install camel-jdbc

Note that this only needs to be done once, and is something which is normally done by
the sysadmins, or by a dependency management package.

## Deploy our demo

Finally we can deploy the fruits of our labour.

In servicemix execute:

    karaf@root> osgi:install mvn:com.melexis/smxdemo/1.0.0-SNAPSHOT/jar
    karaf@root> osgi:list 242
    [ 242] [Installed  ] [            ] [       ] [   60] smxdemo (1.0.0.SNAPSHOT)


Which will install the bundle on your servicemix platform, as we can see by listing
the bundles.


and try to start it.

    karaf@root> osgi:start 242
    karaf@root> osgi:list
    [ 242] [Active     ] [            ] [       ] [   60] smxdemo (1.0.0.SNAPSHOT)


Yay!!! It runs.....

Well, it might not look to work for a while if you did not update the start
transaction id to some recent number. It will return 1000's of results and take
forever. (exercise to the reader : make the initialisation start from the current
transaction id when starting the route)
