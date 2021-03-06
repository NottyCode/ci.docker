# Overview

The images in this repository contain Open Liberty. For more information about Open Liberty, see the [Open Liberty Website](https://openliberty.io/) site. The Docker files are in [GitHub](https://github.com/OpenLiberty/ci.docker).

# Images

There are multiple images available in this repository.

The `kernel` image contains the Liberty kernel and can be used as the basis for custom built images that contain only the features required for a specific application. For example, the following Dockerfile starts with this image, copies in the `server.xml` that lists the features required by the application.

```dockerfile
FROM %%IMAGE%%:kernel
COPY server.xml /config/
```

The `microProfile1` image contains the features required to implement Eclipse MicroProfile 1.3. The `webProfile8` image contains the features required for Java EE8 Web Profile compliance. The `javaee8` image adds the features required for Java EE8 Full Platform compliance. The `javaee8` image is also tagged with `latest`. The `webProfile7` image contains the features required for Java EE7 Web Profile compliance. The `javaee7` image adds the features required for Java EE7 Full Platform compliance. The `springBoot1` and `springBoot2` images contain the features required for running Spring Boot 1.5 and 2.0 applications.

There are also additional images for different JVM combinations. Currently there are tags for java8 only, but there are two variants one based on IBM Java and Ubuntu and the other based on the IBM small footprint Java which is based on alpine linux. The naming structure for the variants is tag-javaversion-vandor/variant. This leads to webProfile8-java8-ibmsfj as one. At this time the full list of images are:

	kernel
	kernel-java8-ibm
	kernel-java8-ibmsfj
	microProfile1
	microProfile1-java8-ibm
	microProfile1-java8-ibmsfj
	springBoot1
	springBoot1-java8-ibm
	springBoot1-java8-ibmsfj
	springBoot2
	springBoot2-java8-ibm
	springBoot2-java8-ibmsfj
	webProfile7
	webProfile7-java8-ibm
	webProfile7-java8-ibmsfj
	webProfile8
	webProfile8-java8-ibm
	webProfile8-java8-ibmsfj
	javaee7
	javaee7-java8-ibm
	javaee7-java8-ibmsfj
	javaee8
	javaee8-java8-ibm
	javaee8-java8-ibmsfj

# Usage

The images are designed to support a number of different usage patterns. The following examples assume that [DefaultServletEngine.zip](https://github.com/WASdev/sample.servlet/releases/download/V1/DefaultServletEngine.zip) has been extracted to `/tmp`.

1.	Each image contains a default server configuration that specifies the corresponding features and exposes ports 9080 and 9443 for HTTP and HTTPS respectively. A .WAR file can therefore be mounted in the `dropins` directory of this server and run. The following example starts a container in the background running a .WAR file from the host file system with the HTTP and HTTPS ports mapped to 80 and 443 respectively.

	```console
	$ docker run -d -p 80:9080 -p 443:9443 \
	    -v /tmp/DefaultServletEngine/dropins/Sample1.war:/config/dropins/Sample1.war \
	    %%IMAGE%%:webProfile8
	```

	When the server is started, you can browse to http://localhost/Sample1/SimpleServlet on the Docker host.

2.	For greater flexibility over configuration, it is possible to mount an entire server configuration directory from the host and then specify the server name as a parameter to the run command. Note: This particular example server configuration provides only HTTP access.

	```console
	$ docker run -d -p 80:9080 \
	  -v /tmp/DefaultServletEngine:/config \
	  %%IMAGE%%:webProfile8
	```

3.	You can also build an application layer on top of this image by using either the default server configuration or a new server configuration. In this example, we have copied the `Sample1.war` from `/tmp/DefaultServletEngine/dropins` to the same directory as the following Dockerfile.

	```dockerfile
	FROM %%IMAGE%%:webProfile8
	ADD Sample1.war /config/dropins/
	```

	This can then be built and run as follows:

	```console
	$ docker build -t app .
	$ docker run -d -p 80:9080 -p 443:9443 app
	```

4.	You can mount a data volume container that contains the application and the server configuration on to the image. This has the benefit that it has no dependency on files from the host but still allows the application container to be easily re-mounted on a newer version of the application server image. This example assumes that you have copied the `/tmp/DefaultServletEngine` directory in to the same directory as the Dockerfile.

	Build and run the data volume container:

	```dockerfile
	FROM %%IMAGE%%:webProfile8
	ADD DefaultServletEngine /config
	```

	```console
	$ docker build -t app-image .
	$ docker run -d -v /config \
	    --name app app-image true
	```

	Run the Open Liberty image with the volumes from the data volume container mounted:

	```console
	$ docker run -d -p 80:9080 \
	  --volumes-from app %%IMAGE%%:webProfile8
	```

# Using `springBoot` images

The `springBoot` images introduce capabilities specific to the support of Spring Boot applications, including the `springBootUtility` used to separate Spring Boot applications into thin applications and dependency library caches. To elaborate these capabilities this section assumes the standalone Spring Boot 2.0.x application `hellospringboot.jar` exists in the `/tmp` directory.

1.	A Spring Boot application JAR deploys to the `dropins/spring` directory within the default server configuration, not the `dropins` directory. Liberty allows one Spring Boot application per server configuration. The following example starts a container running a Spring Boot application.

	```console
	$ docker run -d -p 8080:9080 \
	    -v /tmp/hellospringboot.jar:/config/dropins/spring/hellospringboot.jar \
	    %%IMAGE%%:springBoot2
	```

	Similarly, you can create a Spring Boot application layer over this image by adding the application JAR to the `dropins/spring` directory. In this example we copied `hellospringboot.jar` from `/tmp` to the same directory containing the following Dockerfile.

	```dockerfile
	FROM %%IMAGE%%:springBoot2
	COPY hellospringboot.jar /config/dropins/spring/
	```

	The custom image can be built and run as follows.

	```console
	$ docker build -t app .
	$ docker run -d -p 8080:9080 app
	```

2.	The `springBoot` images provide the library cache directory, `lib.index.cache`, which contains an indexed library cache created by the `springBootUtility` command. Use `lib.index.cache` to provide the library cache for a thin application.

	For example, run the following command to thin the `hellospringboot.jar` application.

	```console
	$ <wlp>/bin/springBootUtility thin \
	   --sourceAppPath=/tmp/hellospringboot.jar \
	   --targetLibCachePath=/tmp/lib.index.cache \
	   --targetThinAppPath=/tmp/thinhellospringboot.jar
	```

	You can run the thin application by mounting both the target application JAR and library cache when starting the container.

	```console
	$ docker run -d -p 8080:9080 \
	    -v /tmp/thinhellospringboot.jar:/config/dropins/spring/thinhellospringboot.jar \
	    -v /tmp/lib.index.cache:/lib.index.cache \
	    %%IMAGE%%:springBoot2
	```

	Similarly, you can use the `springBootUtility` command to create thin application and library cache layers over a `springBoot` image. The following example uses docker staging to efficiently build an image that deploys a fat Spring Boot application as two layers containing a thin application and a library cache.

	```dockerfile
	FROM %%IMAGE%%:springBoot2 as staging
	COPY hellospringboot.jar /staging/myFatApp.jar
	RUN springBootUtility thin \
	   --sourceAppPath=/staging/myFatApp.jar \
	   --targetThinAppPath=/staging/myThinApp.jar \
	   --targetLibCachePath=/staging/lib.index.cache
	FROM %%IMAGE%%:springBoot2
	COPY --from=staging /staging/lib.index.cache /lib.index.cache
	COPY --from=staging /staging/myThinApp.jar /config/dropins/spring/myThinApp.jar
	```

	For Spring Boot applications packaged with library dependencies that rarely change across continuous application updates, you can use the capabilities mentioned above to to share library caches across containers and to create even more efficient docker layers that leverage the docker build cache.

# Providing your own keystore/truststore

When an `open-liberty` image starts, it can generate a Liberty server XML snippet in `/config/configDropins/defaults/keystore.xml` that specifies a `keyStore` stanza with a generated password. This causes Open Liberty to generate a default keystore and truststore with a self-signed certificate when it starts. The `javaee8` and `javaee7` images do this automatically, but other images can request this by setting:

```console
ENV KEYSTORE_REQUIRED "true"
```

When providing your own keystore/truststore, this default behavior can be disabled by adding:

```console
ENV KEYSTORE_REQUIRED "false"
```

It is good practice to place the keystore customization in `/config/configDropins/defaults/keystore.xml` even when not generated since this makes it easier to find and makes moving to the websphere-liberty docker image simpler.

# Using IBM JRE Class data sharing

The IBM JRE provides a feature [Class data sharing](http://www-01.ibm.com/support/knowledgecenter/SSYKE2_8.0.0/com.ibm.java.lnx.80.doc/diag/understanding/shared_classes.html) which offers transparent and dynamic sharing of data between multiple Java Virtual Machines running on the same host by using shared memory backed by a file. When running the Liberty Docker image, it looks for the file at `/opt/ol/wlp//output/.classCache`. To benefit from Class data sharing, this location needs to be shared between containers either through the host or a data volume container.

Taking the application image from example 3 above, containers can share the host file location (containing the shared cache) `/tmp/open-liberty/classCache` as follows:

```console
docker run -d -p 80:9080 -p 443:9443 \
    -v /tmp/open-liberty/classCache:/opt/ol/wlp//output/.classCache app
```

Or, create a named data volume container that exposes a volume at the location of the shared file:

```console
docker run -e LICENSE=accept -v /opt/ol/wlp//output/.classCache \
    --name classcache %%IMAGE%% true
```

Then, run the Open Liberty image with the volumes from the data volume container classcache mounted as follows:

```console
docker run -d -p 80:9080 -p 443:9443 --volumes-from classcache app
```

# Running Open Liberty in read-only mode

Liberty writes to two different directories when running: `/opt/ol/wlp//output` and `/logs`. In order to run the Liberty image in read-only mode these may be mounted as temporary file systems. If using the provided image, the keystore will be generated on initial start up in the server configuration. This means that the server configuration directory either needs to be read-write or the keystore will need to be built into the image. In the example command `/config` is mounted as a read-write volume.

```console
docker run -d -p 80:9080 -p 443:9443 \
    --tmpfs /opt/ol/wlp//output --tmpfs /logs -v /config --read-only \
    %%IMAGE%%:javaee8
```

# Relationship between Open Liberty and WebSphere Liberty

WebSphere Liberty is a commercial distribution of Open Liberty. There is an official docker image for websphere-liberty. The websphere-liberty docker image predates the open-liberty one, so to make it simpler to move from open-liberty to websphere-liberty (or vice versa) the images are broadly compatible. It should be possible to move from one to the other with a simple FROM clause change. Some considerations for moving between them:

	Open Liberty installs into `/opt/ol` rather than `/opt/ibm`.
	Use the `/config` folder for accessing the server configuration.
	Use the `/output` folder for accessing the server output.
	When adding your own SSL configuration use the `/config/configDropins/defaults/keystore.xml`.
