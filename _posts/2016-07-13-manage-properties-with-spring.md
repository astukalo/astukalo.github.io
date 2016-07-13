---
layout: post
title: Managing properties for multiple environments with Spring
tags: [java, properties, spring, environments, gradle]
comments: true
---

It is quite common to have a java application configuration in properties files.
If you don't use containers and service discovery, you may end up having multiple property files for dev, test, prod environments.

Sometimes you also need to change the behaviour of your program for a host or a user without rebuilding it, i.e. to enable/disable a feature,
to integrate with other instance of another service and so on and so forth. Say, you just override some properties, restart the app and voila it works.

In this post I'll provide you my approach of dealing with this problem with the help of spring and gradle.<!--more-->

So, this is the Spring java config:

```java
import static xyz.a5s7.AppConfig.DEFAULT_CONFIG_DIR;

@Configuration
@PropertySources(
    {
        @PropertySource(value = "file:${CONFIG_DIR:" + DEFAULT_CONFIG_DIR + "}/config.properties"),
        @PropertySource(value = "file:${CONFIG_DIR:" + DEFAULT_CONFIG_DIR + "}/${HOST}-config.properties", ignoreResourceNotFound = true),
        @PropertySource(value = "file:${CONFIG_DIR:" + DEFAULT_CONFIG_DIR + "}/${USER}-${HOST}-config.properties", ignoreResourceNotFound = true),
        @PropertySource(value = "file:${CONFIG_DIR:" + DEFAULT_CONFIG_DIR + "}/${username}-${computername}-config.properties", ignoreResourceNotFound = true)
    }
)
public class AppConfig {
    //this default directory is mostly for debugging & dev needs
    static final String DEFAULT_CONFIG_DIR = "src/main/external_resources";

    private @Value("${property1}") String property1;
    private @Value("${property2}") int property2;
    private @Value("${property3}") String property3;

    @Bean
    public MyBean bean() {
    }

    @Bean
    public static PropertySourcesPlaceholderConfigurer propertyConfig() {
        return new PropertySourcesPlaceholderConfigurer();
    }
}
```

Let's take a closer look at it.

Captain Obvious would say that ```@PropertySources``` is a well known approach to declare property sources in spring world.

```DEFAULT_CONFIG_DIR``` is where we have all our configuration in the form of properties file.
I keep them in ```src/main/external_resources``` and while creating distribution package I copy them to ```CONFIG_DIR``` using gradle (below).

This ```${CONFIG_DIR:" + DEFAULT_CONFIG_DIR + "}``` expression means 'if environment property CONFIG_DIR present, then use it,
else use DEFAULT_CONFIG_DIR'. The latter constant I defined in the ```AppConfig``` class for the purpose of reuse, maintainability and convenience.
While debugging or running my app from IDE I don't need to declare environment variable, i.e. ```-DCONFIG_DIR="/home/anton/projects/test"```

```${HOST}-config.properties```, ```${USER}-${HOST}-config.properties``` are used to override all or **some** properties declared in ```config.properties```
 for a user or a host. ```${username}-${computername}-config.properties``` is not a duplicate, it is Windows specific.
 These files are optional and I use ```ignoreResourceNotFound = true``` to ignore them if they don't exist.

In order to resolve placeholders in ```@Value``` annotations I registered a ```PropertySourcesPlaceholderConfigurer``` bean.

If use still use xml Spring configs, then you can achieve the same the following way:


```xml
<bean id="propLocations" class="java.util.ArrayList">
    <constructor-arg>
        <list>
            <value>file:#{systemEnvironment['CONFIG_DIR'] ?: 'src/main/external_resources'}/config.properties</value>
            <value>file:#{systemEnvironment['CONFIG_DIR'] ?: 'src/main/external_resources'}/${HOST}-config.properties</value>
            <value>file:#{systemEnvironment['CONFIG_DIR'] ?: 'src/main/external_resources'}/${USER}-${HOST}-config.properties</value>
            <value>file:#{systemEnvironment['CONFIG_DIR'] ?: 'src/main/external_resources'}/${username}-${computername}-config.properties</value>
        </list>
    </constructor-arg>
</bean>

<bean id="property.configurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations" ref="propLocations"/>
    <property name="ignoreResourceNotFound" value="true"/>
    <property name="ignoreUnresolvablePlaceholders" value="true"/>
    <property name="valueSeparator" value="?"/>
</bean>
```

So far, we have done with spring configuration. We just need to externalize properties files. I copy them with gradle.
Here it is the script (I have omitted dependencies):


``` groovy
apply plugin: 'java'
apply plugin: 'application'

sourceCompatibility = 1.8
targetCompatibility = 1.8
mainClassName = "xyz.a5s7.Main"

distributions {
    main {
        contents {
            into('config') {
                from("$projectDir/src/main/external_resources")
                from ("$projectDir/src/main/resources") {
                    include 'logback.xml'
                }
            }
        }
    }
}

jar {
    manifest {
        attributes(
                'Main-Class': "$mainClassName",
                'Class-Path': configurations.compile.collect { it.getName() }.join(' ')
        )
    }
}
```

If you run ```gradle clean installDist```, distribution plugin will create a ```config``` directory within ```install``` and will copy properties file there.

That's pretty much it. Alternatively, you can take a look at [Spring's profiles](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config-profile-specific-properties),
if you don't have much environments and you are ok with setting profile name as an environment variable. 