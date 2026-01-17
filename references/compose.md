# Docker-Best-Practices - Compose

**Pages:** 5

---

## Set, use, and manage variables in a Compose file with interpolation

**URL:** https://docs.docker.com/compose/env-file/

**Contents:**
- Set, use, and manage variables in a Compose file with interpolation
- Interpolation syntax
- Ways to set variables with interpolation
  - .env file
    - Additional information
    - .env file syntax
  - Substitute with --env-file
    - Additional information
  - local .env file versus <project directory> .env file
  - Substitute from the shell

A Compose file can use variables to offer more flexibility. If you want to quickly switch between image tags to test multiple versions, or want to adjust a volume source to your local environment, you don't need to edit the Compose file each time, you can just set variables that insert values into your Compose file at runtime.

Interpolation can also be used to insert values into your Compose file at runtime, which is then used to pass variables into your container's environment

Below is a simple example:

When you run docker compose up, the web service defined in the Compose file interpolates in the image webapp:v1.5 which was set in the .env file. You can verify this with the config command, which prints your resolved application config to the terminal:

Interpolation is applied for unquoted and double-quoted values. Both braced (${VAR}) and unbraced ($VAR) expressions are supported.

For braced expressions, the following formats are supported:

For more information, see Interpolation in the Compose Specification.

Docker Compose can interpolate variables into your Compose file from multiple sources.

Note that when the same variable is declared by multiple sources, precedence applies:

You can check variables and values used by Compose to interpolate the Compose model by running docker compose config --environment.

An .env file in Docker Compose is a text file used to define variables that should be made available for interpolation when running docker compose up. This file typically contains key-value pairs of variables, and it lets you centralize and manage configuration in one place. The .env file is useful if you have multiple variables you need to store.

The .env file is the default method for setting variables. The .env file should be placed at the root of the project directory next to your compose.yaml file. For more information on formatting an environment file, see Syntax for environment files.

If you define a variable in your .env file, you can reference it directly in your compose.yaml with the environment attribute. For example, if your .env file contains the environment variable DEBUG=1 and your compose.yaml file looks like this:

Docker Compose replaces ${DEBUG} with the value from the .env file

Be aware of Environment variables precedence when using variables in an .env file that as environment variables in your container's environment.

You can place your .env file in a location other than the root of your project's directory, and then use the --env-file option in the CLI so Compose can navigate to it.

Your .env file can be overridden by another .env if it is substituted with --env-file.

Substitution from .env files is a Docker Compose CLI feature.

It is not supported by Swarm when running docker stack deploy.

The following syntax rules apply to environment files:

Lines beginning with # are processed as comments and ignored.

Blank lines are ignored.

Unquoted and double-quoted (") values have interpolation applied.

Each line represents a key-value pair. Values can optionally be quoted.

Delimiter separating key and value can be either = or :.

Spaces before and after value are ignored.

Inline comments for unquoted values must be preceded with a space.

Inline comments for quoted values must follow the closing quote.

Single-quoted (') values are used literally.

Quotes can be escaped with \.

Common shell escape sequences including \n, \r, \t, and \\ are supported in double-quoted values.

Single-quoted values can span multiple lines. Example:

If you then run docker compose config, you'll see:

You can set default values for multiple environment variables, in an .env file and then pass the file as an argument in the CLI.

The advantage of this method is that you can store the file anywhere and name it appropriately, for example, This file path is relative to the current working directory where the Docker Compose command is executed. Passing the file path is done using the --env-file option:

An .env file can also be used to declare pre-defined environment variables used to control Compose behavior and files to be loaded.

When executed without an explicit --env-file flag, Compose searches for an .env file in your working directory (PWD) and loads values both for self-configuration and interpolation. If the values in this file define the COMPOSE_FILE pre-defined variable, which results in a project directory being set to another folder, Compose will load a second .env file, if present. This second .env file has a lower precedence.

This mechanism makes it possible to invoke an existing Compose project with a custom set of variables as overrides, without the need to pass environment variables by the command line.

You can use existing environment variables from your host machine or from the shell environment where you execute docker compose commands. This lets you dynamically inject values into your Docker Compose configuration at runtime. For example, suppose the shell contains POSTGRES_VERSION=9.3 and you supply the following configuration:

When you run docker compose up with this configuration, Compose looks for the POSTGRES_VERSION environment variable in the shell and substitutes its value in. For this example, Compose resolves the image to postgres:9.3 before running the configuration.

If an environment variable is not set, Compose substitutes with an empty string. In the previous example, if POSTGRES_VERSION is not set, the value for the image option is postgres:.

postgres: is not a valid image reference. Docker expects either a reference without a tag, like postgres which defaults to the latest image, or with a tag such as postgres:15.

**Examples:**

Example 1 (yaml):
```yaml
$ cat .env
TAG=v1.5
$ cat compose.yaml
services:
  web:
    image: "webapp:${TAG}"
```

Example 2 (yaml):
```yaml
$ cat .env
TAG=v1.5
$ cat compose.yaml
services:
  web:
    image: "webapp:${TAG}"
```

Example 3 (unknown):
```unknown
docker compose up
```

Example 4 (yaml):
```yaml
webapp:v1.5
```

---

## Volumes

**URL:** https://docs.docker.com/storage/volumes/

**Contents:**
- Volumes
- When to use volumes
- A volume's lifecycle
- Mounting a volume over existing data
- Named and anonymous volumes
- Syntax
  - Options for --mount
  - Options for --volume
- Create and manage volumes
- Start a container with a volume

Volumes are persistent data stores for containers, created and managed by Docker. You can create a volume explicitly using the docker volume create command, or Docker can create a volume during container or service creation.

When you create a volume, it's stored within a directory on the Docker host. When you mount the volume into a container, this directory is what's mounted into the container. This is similar to the way that bind mounts work, except that volumes are managed by Docker and are isolated from the core functionality of the host machine.

Volumes are the preferred mechanism for persisting data generated by and used by Docker containers. While bind mounts are dependent on the directory structure and OS of the host machine, volumes are completely managed by Docker. Volumes are a good choice for the following use cases:

Volumes are not a good choice if you need to access the files from the host, as the volume is completely managed by Docker. Use bind mounts if you need to access files or directories from both containers and the host.

Volumes are often a better choice than writing data directly to a container, because a volume doesn't increase the size of the containers using it. Using a volume is also faster; writing into a container's writable layer requires a storage driver to manage the filesystem. The storage driver provides a union filesystem, using the Linux kernel. This extra abstraction reduces performance as compared to using volumes, which write directly to the host filesystem.

If your container generates non-persistent state data, consider using a tmpfs mount to avoid storing the data anywhere permanently, and to increase the container's performance by avoiding writing into the container's writable layer.

Volumes use rprivate (recursive private) bind propagation, and bind propagation isn't configurable for volumes.

A volume's contents exist outside the lifecycle of a given container. When a container is destroyed, the writable layer is destroyed with it. Using a volume ensures that the data is persisted even if the container using it is removed.

A given volume can be mounted into multiple containers simultaneously. When no running container is using a volume, the volume is still available to Docker and isn't removed automatically. You can remove unused volumes using docker volume prune.

If you mount a non-empty volume into a directory in the container in which files or directories exist, the pre-existing files are obscured by the mount. This is similar to if you were to save files into /mnt on a Linux host, and then mounted a USB drive into /mnt. The contents of /mnt would be obscured by the contents of the USB drive until the USB drive was unmounted.

With containers, there's no straightforward way of removing a mount to reveal the obscured files again. Your best option is to recreate the container without the mount.

If you mount an empty volume into a directory in the container in which files or directories exist, these files or directories are propagated (copied) into the volume by default. Similarly, if you start a container and specify a volume which does not already exist, an empty volume is created for you. This is a good way to pre-populate data that another container needs.

To prevent Docker from copying a container's pre-existing files into an empty volume, use the volume-nocopy option, see Options for --mount.

A volume may be named or anonymous. Anonymous volumes are given a random name that's guaranteed to be unique within a given Docker host. Just like named volumes, anonymous volumes persist even if you remove the container that uses them, except if you use the --rm flag when creating the container, in which case the anonymous volume associated with the container is destroyed. See Remove anonymous volumes.

If you create multiple containers consecutively that each use anonymous volumes, each container creates its own volume. Anonymous volumes aren't reused or shared between containers automatically. To share an anonymous volume between two or more containers, you must mount the anonymous volume using the random volume ID.

To mount a volume with the docker run command, you can use either the --mount or --volume flag.

In general, --mount is preferred. The main difference is that the --mount flag is more explicit and supports all the available options.

You must use --mount if you want to:

The --mount flag consists of multiple key-value pairs, separated by commas and each consisting of a <key>=<value> tuple. The order of the keys isn't significant.

Valid options for --mount type=volume include:

The --volume or -v flag consists of three fields, separated by colon characters (:). The fields must be in the correct order.

In the case of named volumes, the first field is the name of the volume, and is unique on a given host machine. For anonymous volumes, the first field is omitted. The second field is the path where the file or directory is mounted in the container.

The third field is optional, and is a comma-separated list of options. Valid options for --volume with a data volume include:

Unlike a bind mount, you can create and manage volumes outside the scope of any container.

If you start a container with a volume that doesn't yet exist, Docker creates the volume for you. The following example mounts the volume myvol2 into /app/ in the container.

The following -v and --mount examples produce the same result. You can't run them both unless you remove the devtest container and the myvol2 volume after running the first one.

Use docker inspect devtest to verify that Docker created the volume and it mounted correctly. Look for the Mounts section:

This shows that the mount is a volume, it shows the correct source and destination, and that the mount is read-write.

Stop the container and remove the volume. Note volume removal is a separate step.

The following example shows a single Docker Compose service with a volume:

Running docker compose up for the first time creates a volume. Docker reuses the same volume when you run the command subsequently.

You can create a volume directly outside of Compose using docker volume create and then reference it inside compose.yaml as follows:

For more information about using volumes with Compose, refer to the Volumes section in the Compose specification.

When you start a service and define a volume, each service container uses its own local volume. None of the containers can share this data if you use the local volume driver. However, some volume drivers do support shared storage.

The following example starts an nginx service with four replicas, each of which uses a local volume called myvol2.

Use docker service ps devtest-service to verify that the service is running:

You can remove the service to stop the running tasks:

Removing the service doesn't remove any volumes created by the service. Volume removal is a separate step.

If you start a container which creates a new volume, and the container has files or directories in the directory to be mounted such as /app/, Docker copies the directory's contents into the volume. The container then mounts and uses the volume, and other containers which use the volume also have access to the pre-populated content.

To show this, the following example starts an nginx container and populates the new volume nginx-vol with the contents of the container's /usr/share/nginx/html directory. This is where Nginx stores its default HTML content.

The --mount and -v examples have the same end result.

After running either of these examples, run the following commands to clean up the containers and volumes. Note volume removal is a separate step.

For some development applications, the container needs to write into the bind mount so that changes are propagated back to the Docker host. At other times, the container only needs read access to the data. Multiple containers can mount the same volume. You can simultaneously mount a single volume as read-write for some containers and as read-only for others.

The following example changes the previous one. It mounts the directory as a read-only volume, by adding ro to the (empty by default) list of options, after the mount point within the container. Where multiple options are present, you can separate them using commas.

The --mount and -v examples have the same result.

Use docker inspect nginxtest to verify that Docker created the read-only mount correctly. Look for the Mounts section:

Stop and remove the container, and remove the volume. Volume removal is a separate step.

When you mount a volume to a container, you can specify a subdirectory of the volume to use, with the volume-subpath parameter for the --mount flag. The subdirectory that you specify must exist in the volume before you attempt to mount it into a container; if it doesn't exist, the mount fails.

Specifying volume-subpath is useful if you only want to share a specific portion of a volume with a container. Say for example that you have multiple containers running and you want to store logs from each container in a shared volume. You can create a subdirectory for each container in the shared volume, and mount the subdirectory to the container.

The following example creates a logs volume and initiates the subdirectories app1 and app2 in the volume. It then starts two containers and mounts one of the subdirectories of the logs volume to each container. This example assumes that the processes in the containers write their logs to /var/log/app1 and /var/log/app2.

With this setup, the containers write their logs to separate subdirectories of the logs volume. The containers can't access the other container's logs.

When building fault-tolerant applications, you may need to configure multiple replicas of the same service to have access to the same files.

There are several ways to achieve this when developing your applications. One is to add logic to your application to store files on a cloud object storage system like Amazon S3. Another is to create volumes with a driver that supports writing files to an external storage system like NFS or Amazon S3.

Volume drivers let you abstract the underlying storage system from the application logic. For example, if your services use a volume with an NFS driver, you can update the services to use a different driver. For example, to store data in the cloud, without changing the application logic.

When you create a volume using docker volume create, or when you start a container which uses a not-yet-created volume, you can specify a volume driver. The following examples use the rclone/docker-volume-rclone volume driver, first when creating a standalone volume, and then when starting a container which creates a new volume.

If your volume driver accepts a comma-separated list as an option, you must escape the value from the outer CSV parser. To escape a volume-opt, surround it with double quotes (") and surround the entire mount parameter with single quotes (').

For example, the local driver accepts mount options as a comma-separated list in the o parameter. This example shows the correct way to escape the list.

The following example assumes that you have two nodes, the first of which is a Docker host and can connect to the second node using SSH.

On the Docker host, install the rclone/docker-volume-rclone plugin:

This example mounts the /remote directory on host 1.2.3.4 into a volume named rclonevolume. Each volume driver may have zero or more configurable options, you specify each of them using an -o flag.

This volume can now be mounted into containers.

If the volume driver requires you to pass any options, you must use the --mount flag to mount the volume, and not -v.

The following example shows how you can create an NFS volume when creating a service. It uses 10.0.0.10 as the NFS server and /var/docker-nfs as the exported directory on the NFS server. Note that the volume driver specified is local.

You can mount a Samba share directly in Docker without configuring a mount point on your host.

The addr option is required if you specify a hostname instead of an IP. This lets Docker perform the hostname lookup.

You can mount a block storage device, such as an external drive or a drive partition, to a container. The following example shows how to create and use a file as a block storage device, and how to mount the block device as a container volume.

The following procedure is only an example. The solution illustrated here isn't recommended as a general practice. Don't attempt this approach unless you're confident about what you're doing.

Under the hood, the --mount flag using the local storage driver invokes the Linux mount syscall and forwards the options you pass to it unaltered. Docker doesn't implement any additional functionality on top of the native mount features supported by the Linux kernel.

If you're familiar with the Linux mount command, you can think of the --mount options as forwarded to the mount command in the following manner:

To explain this further, consider the following mount command example. This command mounts the /dev/loop5 device to the path /external-drive on the system.

The following docker run command achieves a similar result, from the point of view of the container being run. Running a container with this --mount option sets up the mount in the same way as if you had executed the mount command from the previous example.

You can't run the mount command inside the container directly, because the container is unable to access the /dev/loop5 device. That's why the docker run command uses the --mount option.

The following steps create an ext4 filesystem and mounts it into a container. The filesystem support of your system depends on the version of the Linux kernel you are using.

Create a file and allocate some space to it:

Build a filesystem onto the disk.raw file:

Create a loop device:

losetup creates an ephemeral loop device that's removed after system reboot, or manually removed with losetup -d.

Run a container that mounts the loop device as a volume:

When the container starts, the path /external-drive mounts the disk.raw file from the host filesystem as a block device.

When you're done, and the device is unmounted from the container, detach the loop device to remove the device from the host system:

Volumes are useful for backups, restores, and migrations. Use the --volumes-from flag to create a new container that mounts that volume.

For example, create a new container named dbstore:

When the command completes and the container stops, it creates a backup of the dbdata volume.

With the backup just created, you can restore it to the same container, or to another container that you created elsewhere.

For example, create a new container named dbstore2:

Then, un-tar the backup file in the new container’s data volume:

You can use these techniques to automate backup, migration, and restore testing using your preferred tools.

A Docker data volume persists after you delete a container. There are two types of volumes to consider:

To automatically remove anonymous volumes, use the --rm option. For example, this command creates an anonymous /foo volume. When you remove the container, the Docker Engine removes the /foo volume but not the awesome volume.

If another container binds the volumes with --volumes-from, the volume definitions are copied and the anonymous volume also stays after the first container is removed.

To remove all unused volumes and free up space:

**Examples:**

Example 1 (unknown):
```unknown
docker volume create
```

Example 2 (unknown):
```unknown
docker volume prune
```

Example 3 (unknown):
```unknown
volume-nocopy
```

Example 4 (unknown):
```unknown
$ docker run --mount type=volume,src=<volume-name>,dst=<mount-path>
$ docker run --volume <volume-name>:<mount-path>
```

---

## Use Compose in production

**URL:** https://docs.docker.com/compose/production/

**Contents:**
- Use Compose in production
  - Modify your Compose file for production
  - Deploying changes
  - Running Compose on a single server
- Next steps

When you define your app with Compose in development, you can use this definition to run your application in different environments such as CI, staging, and production.

The easiest way to deploy an application is to run it on a single server, similar to how you would run your development environment. If you want to scale up your application, you can run Compose apps on a Swarm cluster.

You may need to make changes to your app configuration to make it ready for production. These changes might include:

For this reason, consider defining an additional Compose file, for example compose.production.yaml, with production-specific configuration details. This configuration file only needs to include the changes you want to make from the original Compose file. The additional Compose file is then applied over the original compose.yaml to create a new configuration.

Once you have a second configuration file, you can use it with the -f option:

See Using multiple compose files for a more complete example, and other options.

When you make changes to your app code, remember to rebuild your image and recreate your app's containers. To redeploy a service called web, use:

This first command rebuilds the image for web and then stops, destroys, and recreates just the web service. The --no-deps flag prevents Compose from also recreating any services that web depends on.

You can use Compose to deploy an app to a remote Docker host by setting the DOCKER_HOST, DOCKER_TLS_VERIFY, and DOCKER_CERT_PATH environment variables appropriately. For more information, see pre-defined environment variables.

Once you've set up your environment variables, all the normal docker compose commands work with no further configuration.

**Examples:**

Example 1 (yaml):
```yaml
restart: always
```

Example 2 (unknown):
```unknown
compose.production.yaml
```

Example 3 (unknown):
```unknown
compose.yaml
```

Example 4 (unknown):
```unknown
$ docker compose -f compose.yaml -f compose.production.yaml up -d
```

---

## Volumes

**URL:** https://docs.docker.com/engine/storage/volumes/

**Contents:**
- Volumes
- When to use volumes
- A volume's lifecycle
- Mounting a volume over existing data
- Named and anonymous volumes
- Syntax
  - Options for --mount
  - Options for --volume
- Create and manage volumes
- Start a container with a volume

Volumes are persistent data stores for containers, created and managed by Docker. You can create a volume explicitly using the docker volume create command, or Docker can create a volume during container or service creation.

When you create a volume, it's stored within a directory on the Docker host. When you mount the volume into a container, this directory is what's mounted into the container. This is similar to the way that bind mounts work, except that volumes are managed by Docker and are isolated from the core functionality of the host machine.

Volumes are the preferred mechanism for persisting data generated by and used by Docker containers. While bind mounts are dependent on the directory structure and OS of the host machine, volumes are completely managed by Docker. Volumes are a good choice for the following use cases:

Volumes are not a good choice if you need to access the files from the host, as the volume is completely managed by Docker. Use bind mounts if you need to access files or directories from both containers and the host.

Volumes are often a better choice than writing data directly to a container, because a volume doesn't increase the size of the containers using it. Using a volume is also faster; writing into a container's writable layer requires a storage driver to manage the filesystem. The storage driver provides a union filesystem, using the Linux kernel. This extra abstraction reduces performance as compared to using volumes, which write directly to the host filesystem.

If your container generates non-persistent state data, consider using a tmpfs mount to avoid storing the data anywhere permanently, and to increase the container's performance by avoiding writing into the container's writable layer.

Volumes use rprivate (recursive private) bind propagation, and bind propagation isn't configurable for volumes.

A volume's contents exist outside the lifecycle of a given container. When a container is destroyed, the writable layer is destroyed with it. Using a volume ensures that the data is persisted even if the container using it is removed.

A given volume can be mounted into multiple containers simultaneously. When no running container is using a volume, the volume is still available to Docker and isn't removed automatically. You can remove unused volumes using docker volume prune.

If you mount a non-empty volume into a directory in the container in which files or directories exist, the pre-existing files are obscured by the mount. This is similar to if you were to save files into /mnt on a Linux host, and then mounted a USB drive into /mnt. The contents of /mnt would be obscured by the contents of the USB drive until the USB drive was unmounted.

With containers, there's no straightforward way of removing a mount to reveal the obscured files again. Your best option is to recreate the container without the mount.

If you mount an empty volume into a directory in the container in which files or directories exist, these files or directories are propagated (copied) into the volume by default. Similarly, if you start a container and specify a volume which does not already exist, an empty volume is created for you. This is a good way to pre-populate data that another container needs.

To prevent Docker from copying a container's pre-existing files into an empty volume, use the volume-nocopy option, see Options for --mount.

A volume may be named or anonymous. Anonymous volumes are given a random name that's guaranteed to be unique within a given Docker host. Just like named volumes, anonymous volumes persist even if you remove the container that uses them, except if you use the --rm flag when creating the container, in which case the anonymous volume associated with the container is destroyed. See Remove anonymous volumes.

If you create multiple containers consecutively that each use anonymous volumes, each container creates its own volume. Anonymous volumes aren't reused or shared between containers automatically. To share an anonymous volume between two or more containers, you must mount the anonymous volume using the random volume ID.

To mount a volume with the docker run command, you can use either the --mount or --volume flag.

In general, --mount is preferred. The main difference is that the --mount flag is more explicit and supports all the available options.

You must use --mount if you want to:

The --mount flag consists of multiple key-value pairs, separated by commas and each consisting of a <key>=<value> tuple. The order of the keys isn't significant.

Valid options for --mount type=volume include:

The --volume or -v flag consists of three fields, separated by colon characters (:). The fields must be in the correct order.

In the case of named volumes, the first field is the name of the volume, and is unique on a given host machine. For anonymous volumes, the first field is omitted. The second field is the path where the file or directory is mounted in the container.

The third field is optional, and is a comma-separated list of options. Valid options for --volume with a data volume include:

Unlike a bind mount, you can create and manage volumes outside the scope of any container.

If you start a container with a volume that doesn't yet exist, Docker creates the volume for you. The following example mounts the volume myvol2 into /app/ in the container.

The following -v and --mount examples produce the same result. You can't run them both unless you remove the devtest container and the myvol2 volume after running the first one.

Use docker inspect devtest to verify that Docker created the volume and it mounted correctly. Look for the Mounts section:

This shows that the mount is a volume, it shows the correct source and destination, and that the mount is read-write.

Stop the container and remove the volume. Note volume removal is a separate step.

The following example shows a single Docker Compose service with a volume:

Running docker compose up for the first time creates a volume. Docker reuses the same volume when you run the command subsequently.

You can create a volume directly outside of Compose using docker volume create and then reference it inside compose.yaml as follows:

For more information about using volumes with Compose, refer to the Volumes section in the Compose specification.

When you start a service and define a volume, each service container uses its own local volume. None of the containers can share this data if you use the local volume driver. However, some volume drivers do support shared storage.

The following example starts an nginx service with four replicas, each of which uses a local volume called myvol2.

Use docker service ps devtest-service to verify that the service is running:

You can remove the service to stop the running tasks:

Removing the service doesn't remove any volumes created by the service. Volume removal is a separate step.

If you start a container which creates a new volume, and the container has files or directories in the directory to be mounted such as /app/, Docker copies the directory's contents into the volume. The container then mounts and uses the volume, and other containers which use the volume also have access to the pre-populated content.

To show this, the following example starts an nginx container and populates the new volume nginx-vol with the contents of the container's /usr/share/nginx/html directory. This is where Nginx stores its default HTML content.

The --mount and -v examples have the same end result.

After running either of these examples, run the following commands to clean up the containers and volumes. Note volume removal is a separate step.

For some development applications, the container needs to write into the bind mount so that changes are propagated back to the Docker host. At other times, the container only needs read access to the data. Multiple containers can mount the same volume. You can simultaneously mount a single volume as read-write for some containers and as read-only for others.

The following example changes the previous one. It mounts the directory as a read-only volume, by adding ro to the (empty by default) list of options, after the mount point within the container. Where multiple options are present, you can separate them using commas.

The --mount and -v examples have the same result.

Use docker inspect nginxtest to verify that Docker created the read-only mount correctly. Look for the Mounts section:

Stop and remove the container, and remove the volume. Volume removal is a separate step.

When you mount a volume to a container, you can specify a subdirectory of the volume to use, with the volume-subpath parameter for the --mount flag. The subdirectory that you specify must exist in the volume before you attempt to mount it into a container; if it doesn't exist, the mount fails.

Specifying volume-subpath is useful if you only want to share a specific portion of a volume with a container. Say for example that you have multiple containers running and you want to store logs from each container in a shared volume. You can create a subdirectory for each container in the shared volume, and mount the subdirectory to the container.

The following example creates a logs volume and initiates the subdirectories app1 and app2 in the volume. It then starts two containers and mounts one of the subdirectories of the logs volume to each container. This example assumes that the processes in the containers write their logs to /var/log/app1 and /var/log/app2.

With this setup, the containers write their logs to separate subdirectories of the logs volume. The containers can't access the other container's logs.

When building fault-tolerant applications, you may need to configure multiple replicas of the same service to have access to the same files.

There are several ways to achieve this when developing your applications. One is to add logic to your application to store files on a cloud object storage system like Amazon S3. Another is to create volumes with a driver that supports writing files to an external storage system like NFS or Amazon S3.

Volume drivers let you abstract the underlying storage system from the application logic. For example, if your services use a volume with an NFS driver, you can update the services to use a different driver. For example, to store data in the cloud, without changing the application logic.

When you create a volume using docker volume create, or when you start a container which uses a not-yet-created volume, you can specify a volume driver. The following examples use the rclone/docker-volume-rclone volume driver, first when creating a standalone volume, and then when starting a container which creates a new volume.

If your volume driver accepts a comma-separated list as an option, you must escape the value from the outer CSV parser. To escape a volume-opt, surround it with double quotes (") and surround the entire mount parameter with single quotes (').

For example, the local driver accepts mount options as a comma-separated list in the o parameter. This example shows the correct way to escape the list.

The following example assumes that you have two nodes, the first of which is a Docker host and can connect to the second node using SSH.

On the Docker host, install the rclone/docker-volume-rclone plugin:

This example mounts the /remote directory on host 1.2.3.4 into a volume named rclonevolume. Each volume driver may have zero or more configurable options, you specify each of them using an -o flag.

This volume can now be mounted into containers.

If the volume driver requires you to pass any options, you must use the --mount flag to mount the volume, and not -v.

The following example shows how you can create an NFS volume when creating a service. It uses 10.0.0.10 as the NFS server and /var/docker-nfs as the exported directory on the NFS server. Note that the volume driver specified is local.

You can mount a Samba share directly in Docker without configuring a mount point on your host.

The addr option is required if you specify a hostname instead of an IP. This lets Docker perform the hostname lookup.

You can mount a block storage device, such as an external drive or a drive partition, to a container. The following example shows how to create and use a file as a block storage device, and how to mount the block device as a container volume.

The following procedure is only an example. The solution illustrated here isn't recommended as a general practice. Don't attempt this approach unless you're confident about what you're doing.

Under the hood, the --mount flag using the local storage driver invokes the Linux mount syscall and forwards the options you pass to it unaltered. Docker doesn't implement any additional functionality on top of the native mount features supported by the Linux kernel.

If you're familiar with the Linux mount command, you can think of the --mount options as forwarded to the mount command in the following manner:

To explain this further, consider the following mount command example. This command mounts the /dev/loop5 device to the path /external-drive on the system.

The following docker run command achieves a similar result, from the point of view of the container being run. Running a container with this --mount option sets up the mount in the same way as if you had executed the mount command from the previous example.

You can't run the mount command inside the container directly, because the container is unable to access the /dev/loop5 device. That's why the docker run command uses the --mount option.

The following steps create an ext4 filesystem and mounts it into a container. The filesystem support of your system depends on the version of the Linux kernel you are using.

Create a file and allocate some space to it:

Build a filesystem onto the disk.raw file:

Create a loop device:

losetup creates an ephemeral loop device that's removed after system reboot, or manually removed with losetup -d.

Run a container that mounts the loop device as a volume:

When the container starts, the path /external-drive mounts the disk.raw file from the host filesystem as a block device.

When you're done, and the device is unmounted from the container, detach the loop device to remove the device from the host system:

Volumes are useful for backups, restores, and migrations. Use the --volumes-from flag to create a new container that mounts that volume.

For example, create a new container named dbstore:

When the command completes and the container stops, it creates a backup of the dbdata volume.

With the backup just created, you can restore it to the same container, or to another container that you created elsewhere.

For example, create a new container named dbstore2:

Then, un-tar the backup file in the new container’s data volume:

You can use these techniques to automate backup, migration, and restore testing using your preferred tools.

A Docker data volume persists after you delete a container. There are two types of volumes to consider:

To automatically remove anonymous volumes, use the --rm option. For example, this command creates an anonymous /foo volume. When you remove the container, the Docker Engine removes the /foo volume but not the awesome volume.

If another container binds the volumes with --volumes-from, the volume definitions are copied and the anonymous volume also stays after the first container is removed.

To remove all unused volumes and free up space:

**Examples:**

Example 1 (unknown):
```unknown
docker volume create
```

Example 2 (unknown):
```unknown
docker volume prune
```

Example 3 (unknown):
```unknown
volume-nocopy
```

Example 4 (unknown):
```unknown
$ docker run --mount type=volume,src=<volume-name>,dst=<mount-path>
$ docker run --volume <volume-name>:<mount-path>
```

---

## Legacy versions

**URL:** https://docs.docker.com/compose/compose-file/compose-file-v3/

**Contents:**
- Legacy versions

The legacy versions of the Compose file reference has moved to the V1 branch of the Compose repository. They are no longer being actively maintained.

The latest and recommended version of the Compose file format is defined by the Compose Specification. This format merges the 2.x and 3.x versions and is implemented by Compose 1.27.0+. For more information, see the History and development of Docker Compose.

---
