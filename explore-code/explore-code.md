# Explore the Application Code

## Introduction

This lab walks you through the application code allowing you to gain an understanding of how the code works.

> Note: You can choose any of the languages you wish to inspect.

Estimated time: 10 minutes

### Objectives

In this lab, you will:

* Explore Java code
* Explore Python Code
* Explore JavaScript Code
* Explore Go Code

### Prerequisites

* You should have completed the previous labs.

## Task 1: Open the code in the "Files" application

1. In the remote session, choose **`Activities`** and select the **`Files`** icon to open the **"Files"** application.
     
   ![Files](images/files.png "Files")
        
2. Double click **`coherence-demo`** to change to that directory.

   ![Directory](images/directory.png "Directory")

## Task 2: Explore the pom.xml, Configuration and Startup
                  
1. Application Startup

   The application is started using the following command:

      ```bash
      mvn -P grid-edition exec:exec
      ```

      This starts the application using the exec-maven-plugin which sets the following arguments in the project in [pom.xml](https://github.com/coherence-community/coherence-demo/blob/b632f832fe9860e9eb6fb454f13a4158367d0f23/pom.xml#L351):
 
      | Argument                                            | Usage                                                                                 |
      |-----------------------------------------------------|---------------------------------------------------------------------------------------|
      | -Dcoherence.log.level=7                             | Set the Coherence log level to 7 to provide more verbose output that the default of 5 |
      | -Dcoherence.management=all                          | Enables management for the cluster                                                    |
      | -Dcoherence.wka=127.0.0.1                           | Scopes the cluster to the localhost only                                              |
      | -Dcoherence.ttl=0                                   | Ensures cluster traffic does not go outside this VM                                   |
      | -Dcoherence.grpc.server.port=1408                   | Enable the gRPC Proxy on port 1408                                                    |
      | -Dcoherence.metrics.http.enabled=${metrics.enabled} | If set to true, enabled Coherence metrics, see the                                    |
                                                                                                                                                     
   The startup class is [com.oracle.coherence.demo.application.Launcher](https://github.com/coherence-community/coherence-demo/blob/1412/src/main/java/com/oracle/coherence/demo/application/Launcher.java) which does the following:
   
      * Determines the time zone and sets sensible primary and secondary cluster names as system properties, as well as Jaegar endpoint
      * Sets the following system property which indicates the cache configuration file to load.

         `System.setProperty("coherence.cacheconfig", "cache-config.xml")`
      * Calls Coherence.main(args) which is the main entry point for coherence.

 
2. Configuration Files

   As part of the build, the relevant cache configuration file (cache-config-grid-edition.xml) and override file (tangosol-coherence-override-grid-edition.xml) 
   are copied from `src/main/resources/` to the target directory. See [here](https://github.com/coherence-community/coherence-demo/blob/b632f832fe9860e9eb6fb454f13a4158367d0f23/pom.xml#L469).

   **Cache Configuration**

      The cache configuration file (cache-config.xml in our case) defines caches and other services, for the cluster. A few areas of particular interest are:
   
      1. An interceptor shown below, or on [GitHub](https://github.com/coherence-community/coherence-demo/blob/b632f832fe9860e9eb6fb454f13a4158367d0f23/src/main/resources/cache-config-grid-edition.xml#L46) 
         which is run on startup of the Coherence cluster: 
         ```xml  
         <interceptors>
           <interceptor>
             <instance>
                <class-name>com.oracle.coherence.demo.application.BootstrapInterceptor</class-name>
             </instance>
           </interceptor>
         </interceptors>
         ```
      2. Cache Scheme Mapping, shown below or on [GitHub](https://github.com/coherence-community/coherence-demo/blob/b632f832fe9860e9eb6fb454f13a4158367d0f23/src/main/resources/cache-config-grid-edition.xml#L51)
         defines the mapping from the cache name to a caching scheme and shows the domain classes, explained further below, used for our cache.
         ```xml
         <caching-scheme-mapping>
           <cache-mapping>
             <cache-name>Trade</cache-name>
             <scheme-name>federated-scheme</scheme-name>
             <key-type>java.lang.String</key-type>
             <value-type>com.oracle.coherence.demo.model.Trade</value-type>
           </cache-mapping>
           <cache-mapping>
             <cache-name>Price</cache-name>
             <scheme-name>federated-scheme</scheme-name>
             <key-type>java.lang.String</key-type>
             <value-type>com.oracle.coherence.demo.model.Price</value-type>
           </cache-mapping>
         </caching-scheme-mapping>
         ```
      3. Federated Service Definitions, shown in part below or on [GitHub](https://github.com/coherence-community/coherence-demo/blob/b632f832fe9860e9eb6fb454f13a4158367d0f23/src/main/resources/cache-config-grid-edition.xml#L71)
         which defines the actual federated scheme with its read-write backing map to write through to a database (in memory in our case). We can also see the topology is defined as `Active`, which is a reference to the topology in the override file below. 
         ```xml
           <federated-scheme>
             <scheme-name>federated-scheme</scheme-name>
              <backing-map-scheme>
                <read-write-backing-map-scheme>
                    <internal-cache-scheme>
                        <local-scheme>
                            <unit-calculator>BINARY</unit-calculator>
                        </local-scheme>
                    </internal-cache-scheme>
                    <write-max-batch-size>5000</write-max-batch-size>
                    <!-- Define the cache scheme. -->
                    <cachestore-scheme>
                        <class-scheme>
                            <class-name>
                                com.oracle.coherence.demo.cachestore.JpaCacheStore
                            </class-name>
                            <init-params>
                                ...
                            </init-params>
                        </class-scheme>
                    </cachestore-scheme>
                    <write-delay>2s</write-delay>
                </read-write-backing-map-scheme>
              </backing-map-scheme>
              <autostart>true</autostart>
              <address-provider>
                <local-address>
                  <address system-property="coherence.extend.address"></address>
                  <port system-property="coherence.federation.port">40000</port>
                </local-address>
              </address-provider>
              <topologies>
                <topology>
                   <name>Active</name>
                </topology>
            </topologies>
         </federated-scheme>
         ```
      4. A Http Proxy server which has the JAX-RS resources `ApplicationResourceConfig` which serves the application and `ServiceResourceConfig` which serves various REST endpoints through which the HTML/JavaScript application interacts.

   **Operational Override**

      The tangosol-coherence.xml operational deployment descriptor, or its override, specifies the operational and run-time settings that control clustering, communication, and data management services.
      
      For this demonstration we are configuring federation cluster members in the override file shown below, or on [GitHub](https://github.com/coherence-community/coherence-demo/blob/b632f832fe9860e9eb6fb454f13a4158367d0f23/src/main/resources/tangosol-coherence-override-grid-edition.xml#L38)
      or `src/main/resources/tangosol-coherence-override-grid-edition.xml`.

      ```xml
       <federation-config>
          <participants>   <!-- defines each participant or cluster in the federation setup -->
            <participant>
              <name system-property="primary.cluster">PrimaryCluster</name>
              <initial-action>start</initial-action>
              <remote-addresses>
                <socket-address>
                  <address system-property="primary.cluster.host">127.0.0.1</address>
                  <port    system-property="primary.cluster.port">7574</port>
                </socket-address>
              </remote-addresses>
            </participant>
            <participant>
              <name system-property="secondary.cluster">SecondaryCluster</name>
              <initial-action>pause</initial-action>
              <remote-addresses>
                <socket-address>
                  <address system-property="secondary.cluster.host">127.0.0.1</address>
                  <port    system-property="secondary.cluster.port">7575</port>
                </socket-address>
              </remote-addresses>
            </participant>
          </participants>
          <topology-definitions>
            <active-active>
              <name>Active</name> <!-- active/ active configuration -->
              <active system-property="primary.cluster">PrimaryCluster</active>
              <active system-property="secondary.cluster">SecondaryCluster</active>
            </active-active>
          </topology-definitions>
      </federation-config>
      ```

## Task 2: Explore the Java Code

1. Domain Classes

   There are two classes that hold the main data for the application:
        
   * `Price` - holds information regarding symbols and their current price
   
      Source code: `src/main/java/com/oracle/coherence/demo/model/Price.java` or [GitHub](https://github.com/coherence-community/coherence-demo/blob/1412/src/main/java/com/oracle/coherence/demo/model/Price.java)
   
      Class definition (excluding getters/ setters, etc)
   
      ```java
      @Entity
      @XmlRootElement(name = "price")
      @XmlAccessorType(XmlAccessType.PROPERTY)
      @PortableType(id = 1003)
      public class Price {
    
          /**
           * The symbol (ticker code) of the equity for the {@link Price}.
           */
          @Id
          private String symbol;

          /**
           * The price of the symbol.
          */
          private double price;
     
          ...
      }
      ```      
     
      The `Entity` and `Id` annotations are for JPA persistence and the `PortableType` annotation is used at compile time to instrument the class to
      enable Portable Object Format serialization which is a fast and compact serialization format used by Coherence. 

   * `Trade` - holds individual trades and the price they traded for
        
      Source Code: `src/main/java/com/oracle/coherence/demo/model/Trade.java` or [GitHub](https://github.com/coherence-community/coherence-demo/blob/1412/src/main/java/com/oracle/coherence/demo/model/Trade.java)
      
      Class definition (excluding getters/ setters, etc)
   
      ```java
      @Entity
      @XmlRootElement(name = "trade")
      @XmlAccessorType(XmlAccessType.PROPERTY)
      @PortableType(id = 1004)
      public class Trade {
          /**
           * The unique identifier for this trade.
           */
          @Id
          private String id;
      
          /**
           * The symbol (ticker code) of the equity for the {@link Trade}.
           */
          private String symbol;
      
          /**
           * The number of shares for the {@link Trade}.
           */
          private int quantity;
      
          /**
           * The price at which the shares in the {@link Trade} were acquired.
           */
          private double price;
          
          ...
      }
      ```     

2. Bootstrap Interceptor
 
   

There are various components to the Java based JAX-RS application. You can explore the various classes and packages below via the explorer or via the 
direct GitHub links.

> Note: Not all source or configuration files are displayed, just some that showcase important components.

**Source Files**

| Package                                             | Class                     | Usage                                         | GitHub Link                                                                                                                                                                |
|-----------------------------------------------------|---------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| src/main/java/com/oracle/coherence/demo/application | BootstrapInterceptor.java | Code executed on startup                      | [BootstrapInterceptor.java](https://github.com/coherence-community/coherence-demo/blob/1412/src/main/java/com/oracle/coherence/demo/application/BootstrapInterceptor.java) |
| src/main/java/com/oracle/coherence/demo/application | ChartDataResource.java    | JAX-RS Endpoint to carry out data aggregation | [ChartDataResource.java](https://github.com/coherence-community/coherence-demo/blob/1412/src/main/java/com/oracle/coherence/demo/application/ChartDataResource.java)       |
| src/main/java/com/oracle/coherence/demo/application | EventsResource.java       | JAX-RS Endpoint to listen for events          | [EventsResource.java](https://github.com/coherence-community/coherence-demo/blob/1412/src/main/java/com/oracle/coherence/demo/application/EventsResource.java)             |
| src/main/java/com/oracle/coherence/demo/application | Utilities.java            | Various utilities for working with cache data | [Utilities.java](https://github.com/coherence-community/coherence-demo/blob/1412/src/main/java/com/oracle/coherence/demo/application/Utilities.java)                       |
| src/main/java/com/oracle/coherence/demo/cachestore  |                           | Contains cache store JPA Code                 | [Link](https://github.com/coherence-community/coherence-demo/tree/1412/src/main/java/com/oracle/coherence/demo/cachestore)                                                 |
| src/main/java/com/oracle/coherence/demo/model       |                           | Contains cache model classes                  | [Link](https://github.com/coherence-community/coherence-demo/tree/1412/src/main/java/com/oracle/coherence/demo/model)                                                      |

**Configuration Files**  
 
| File                                             | Usage                                        | Usage                                                       | GitHub Link                                                                                                                                                     |
|--------------------------------------------------|----------------------------------------------|-------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| src/main/resources/cache-config-grid-edition.xml | cache-config-grid-edition.xml                | Contains cache config for federation scheme and cache store | [cache-config-grid-edition.xml](https://github.com/coherence-community/coherence-demo/blob/1412/src/main/resources/cache-config-grid-edition.xml)               |
| src/main/resources/cache-config-grid-edition.xml | tangosol-coherence-override-grid-edition.xml | Contains override config for federation setup               | [ache-config-grid-edition.xml](https://github.com/coherence-community/coherence-demo/blob/1412/src/main/resources/tangosol-coherence-override-grid-edition.xml) |
| src/main/resources/web                           | HTML and JavaScript for web application      |                                                             | [Link](https://github.com/coherence-community/coherence-demo/tree/1412/src/main/resources/web)                                                                  |

    
## Task 3: Explore the Python Code

The **Python** code is available in the following location:
  
* clients/py/main.py
* [main.py](https://github.com/coherence-community/coherence-demo/blob/1412/clients/py/main.py)

## Task 4: Explore the Javascript Code

The **JavaScript** code is available in the following location:
  
* clients/js/main.js
* [main.js](https://github.com/coherence-community/coherence-demo/blob/1412/clients/js/main.js)                       

## Task 5: Explore the Go Code

The **Go code** is available in the following location:
  
* clients/go/main.go
* [main.go](https://github.com/coherence-community/coherence-demo/blob/1412/clients/go/main.go)                       



## Acknowledgements

* **Author** - Tim Middleton
* **Contributors** - Ankit Pandey, Sid Joshi
* **Last Updated By/Date** - Ankit Pandey, November 2024