Tenant Extension
================

eXo Platform extension allowing run a single workspace (tenant) from eXo Cloud on-premise on standalone eXo Platform Tomcat server.

Prerequisites
=============

Ensure following is done before installing and running the extension:
* This extension requires MySQL dump of a target tenant from eXo Cloud. The dump should be applied to a clean database before running the extension. Request a backup via eXo Cloud support channel.
* Obtain clean eXo Platform Enterprise or Express edition Tomcat bundle, it should of a version currently used by eXo Cloud. You have to use the same version as eXo Cloud runs for initial migration and then only upgrade to higher versions. 
* Install MySQL of latest supported by eXo Cloud version for your infrastructure. 
* Use the same Java version as by current eXo Cloud.

This Tenant Extension is independent on the eXo Platform version until it doesn't change location of JCR repository-configuration.xml files and JCR organization service component configuration with related dependecies. Respectively check the above prerequisites with the current eXo Cloud version and apply accordingly.

Currently eXo Cloud uses Platform 4.1-M2, it needs MySQL 5.5 and Java 1.6. 

Introduction
============

eXo Cloud manages several servers of eXo Platform running all tenants data in MySQL databases. eXo Platform's JCR configured to store all data in a database, it hasn't a value storage for large files. Thus when you have a MySQL dump of your tenant - it contains all your data in it. 

eXo Cloud uses JCR organization service, not a default one based on PicketLink IDM. This implementation stores all organization records in a tenant JCR repository. Thus all your users, groups and membershipts also in a MySQL dump of your tenant. 

There are several extra add-ons installed for all tenants in eXo Cloud. They may don't exist in clean Platform you downloaded for on-premise installation. Check with current eXo Cloud version for required add-ons before installing this extension. Each add-on acts differently in the Platform and additional study or consulting may be required to figure out when to install it, before or after the migration of your tenant data.

As for eXo Cloud 4.1-M2, it uses Cloud Drive and Video Calls add-ons. They both can be installed after the migration and upgrade to required Platform version.

How to use
==========

Apply MySQL dump to your clean database, `TENANT_DB` here:

    mysql -uDBUSER -pDBPASSWORD TENANT_DB < DB-YOUR-TENANT-DUMP.sql
    

Configure your Platform datasource for your database, we assume MySQL runs on localhost:3306:

    <!-- eXo JCR Datasource for portal -->
    <Resource name="exo-jcr_portal" auth="Container" type="javax.sql.DataSource"
      initialSize="5" maxActive="50" minIdle="5" maxIdle="15" maxWait="10000"
      validationQuery="SELECT 1" validationQueryTimeout="5" 
      testWhileIdle="true" testOnBorrow="true" testOnReturn="false"
      timeBetweenEvictionRunsMillis="30000" minEvictableIdleTimeMillis="60000"
      removeAbandoned="true" removeAbandonedTimeout="300" logAbandoned="false"
      poolPreparedStatements="true"
      username="DBUSER" password="DBUSER" driverClassName="com.mysql.jdbc.Driver" url="jdbc:mysql://localhost:3306/TENANT_DB?autoReconnect=true" /> 
         
  
You also may comment out eXo IDM Datasource to ensure the PicketLink IDM organization service will not be loaded (it will fail without the datasource).

Next step to configure JCR dialect in `configuration.properties` of your Platform. Note that since Platform 4.1-RC1 you need to use `exo.properties` file to customize defaults of the platform. In this file you also have to setup properly mailing (SMTP server) for user notifications.
Set JCR dialect to `mysql-utf8` as eXo Cloud uses. Disable JCR value storage if you don't plan custom JCR workspaces in addition to what your tenant has.

    # JCR dialect.
    # auto : enabled auto detection
    exo.jcr.datasource.dialect=mysql-utf8
    # Storage location of JCR values
    # - true (default): All JCR values are stored in file system.
    # - false: All JCR values are stored as BLOBs in the database.
    exo.jcr.storage.enabled=false
    


Install Tenant Extension. You can install it from central catalog via Add-ons Manager tool: `addons.sh` for Platform 4.1-M2 or `addon` for 4.1-RC1. If you have  bundle of the addon locally you also can simply unarchive it and copy its content, `lib` and `webapps` folders, to the root of your Platform Tomcat folder.

Now you are ready to start your Platform server. Start it and ensure its log doesn't contain errors. When server will be started successfully, stop it and start it again. After this you can use your tenant on-premise without limitations.

Post-installation steps
=======================

eXo Cloud introduces custom portlets and/or gadgets which not available in standalone Platform. For example, Colleagues Invitation gadget will appear after the migration as it was persisted in JCR (all gadgets do this), but it doesn't work as required web-services don't exist locally. You need to remove such portlets/gadgets after the migration. This can be done from user-interface under Administrator account via [Edit](http://docs.exoplatform.com/PLF40/PLFUserGuide.AdministeringeXoPlatform.ManagingPages.EditingPage.html) menu. 


What is inside the Tenant Extension
===================================

This section for those who want to get inside the extension and customize its configuration. This section assumes that you understand [Portal Extension](http://docs.exoplatform.com/PLF40/PLFDevGuide.eXoPlatformExtensions.html) mechanism.

General idea of the extension is to reproduce an environment of eXo Cloud required for a single tenant from the cloud. On storage level each tenant it is single JCR repository - it's mandatory requirement that all data should be stored in JCR, in its "current" repository. Refer to Platform documenation in [JCR practices](http://docs.exoplatform.com/PLF40/JCR.UsingJCR.JCRApplicationPractices.html) for more details. Starting from GateIn portal to all Platform apps (Content, Social, Celendar etc) all they keep required JCR configurations in own files (usually in WAR files of the app extension). 
By default all JCR repositories use value storage for large files, but it is not what eXo Cloud requires and for this reason the cloud needs dedicated configurations. 

To reproduce the cloud environment in on-premise Platform need overload all default JCR repository configurations with appropriate settings: don't use value storage in workspaces, use `jcrlocks` table name for lock manager. For this purpose we gathered all repository configs from the Platform, updated them and placed in the WAR of the extension on the same paths that in original WARs. Then we set the same hight priority for `PortalContainerDefinitionChangePlugin` as in the cloud. This way we tell the Platform to load our extension the last and thus it will override all configurations loaded previously, including repository configurations.

All required configurations are in the webapp of the extension. Except of JCR repository overriddes there are following components:
* `conf/organization/idm-configuration.xml` - IDM organization service disabled by overloading its configuration file with empty content in this file. Doing this we remove not only IDM implemenation but all related component extensions for Hibernate service
* `conf/cloud/cloud-configuration.xml` - settings specific for eXo Cloud:
    * JCR organization service
    * Password encrypter and organization service listener using it to protect user passwords in the JCR content of the service
    * JCR storage configuration for organization service
    * Limitation services for uploading and WCM UI (can be used to extend upload limits or remove them at all)

As for repository configurations their location cannot be changed or relocated to external files as they have to overload the original locations in Platform apps. The same actual for IDM organization disablement. Cloud configuration can be moved to any place where you configure your components.


Support
=======

Post your issues related to the configurations here on Github. Refer to eXo Cloud support for MySQL backup and current versions.



