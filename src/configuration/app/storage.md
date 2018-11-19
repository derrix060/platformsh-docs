---
search:
    keywords: ['.platform.app.yaml', 'disk', 'mounts']
---

# Storage

The built file system image that results from your build process is mounted read-only.  That means it cannot be edited in production, even by accident.

Many applications still require the ability to write and store files, however.  For that, applications can specify one or more mount points, that is, directories that will be mounted from a writable network file system cluster.  They may be mounted anywhere within the file system of your application.  If the specified directory already exists the contents of it will be masked by the writable mount and inaccessible at runtime.

## Disk

The `disk` key is required, and defines the size of the persistent disk of the application (in MB).  Its minimum value is 256 MB and a validation error will occur if you try to set it lower.

## Mounts

The `mounts` key is an object whose keys are paths relative to the root of the application (that is, where the `.platform.app.yaml` file lives), and values are a 2-line mount definition.

This section is optional; if your application does not need writable local file storage simply omit the `mounts` section and set `disk` to a value to the minimum of 256.

Note that whether a mounted directory is web-accessible or not depends on the configuration of the `web.locations` block in `.platform.app.yaml`.  It may be accessible, or not, and have different rules for different file types as desired.

## Basic mounts

The following block defines a single writable directory, `web/uploads`:

```yaml
mounts:
    'web/uploads':
        source: local
        source_path: uploads
```

The `source` specifies where the writable mount is.  `source_path` specifies the subdirectory from within the source that the mount should point at.  It is often easiest to have it match the name of the mount point itself but that is not required.

### `local` mounts

The `local` source indicates that the mount point will point to a local directory on the application container.  The `source_path` is then a subpath of that.  That means they may overlap.  `local` mounts are not shared between different application containers or workers.

Be aware that the entire `local` space for a single app container is a common directory, and the directory is not wiped.  That means if you create a mount point with a `source_path` of "uploads", then write files there, then remove the mount point, the files will still exist on disk indefinitely until manually removed.

### `service` mounts

A `service` mount refers to a `network-storage` service, as defined in `services.yaml`.  They function in essentially the same way as a `local` mount, with two important differences:

1) The disk size of the `service` mount is controlled in `services.yaml`; it is separate from the value of the `disk` key in `.platform.app.yaml`.
2) Multiple application containers may refer to the same `service` mount and share files.

A `service` mount works like so:

```yaml
mounts:
  'web/uploads':
    source: service
    service: files
    source_path: uploads
```

This assumes that a `network-storage` service named `files` has already been defined.  See the [Network Storage](configuration/services/network-storage.md) page for more details and examples.

## Multi-instance disk mounts

If you have multiple application instances defined (using both `web` and `workers`), each instance will have its own `local` disk mounts.  That's the case even if they are named the same, and even if there is only a single top-level mounts directive.  In that case, every instance will have an identical configuration, but separate, independent file spaces.

If you want to have multiple application instances share file storage, you will need to use a `service` mount.

## How do I set up both public and private file uploads?

The following example sets up two file mounts.  One is mounted at `/private` within the application container, the other at `/web/uploads`.  The two file mounts together have a limit of 1024 MB of storage.

```yaml
disk: 1024

mounts:
    'private':
        source: local
        source_path: private
    'web/uploads':
        source: local
        source_path: uploads
```

Then in the `web.locations` block, you'd specify that the `web/uploads` path is accessible.  For example, this fragment would specify the `/web` path as the docroot but provide a more locked-down access to the `/web/uploads` path.

```yaml
web:
    locations:
        '/':
            # The public directory of the application relative to its root.
            root: 'web'
            # The front-controller script which determines where to send
            # non-static requests.
            passthru: '/app.php'
        # Allow uploaded files to be served, but do not run scripts.
        '/web/uploads':
            root: 'web/uploads'
            expires: 300s
            scripts: false
            allow: true
```

See the [web locations](/configuration/app/web.md) documentation for more details.

## How do I setup overlapping mount paths?

While not recommended it is possible to setup multiple mount points whose source paths overlap.  Consider the following example:

```yaml
mounts:
    'private':
        source: local
        source_path: stuff
    'secret':
        source: local
        source_path:  stuff/secret
```

In this configuration, there will be two mount points as seen from the application: `~/private` and `~/secret`.  However, the `secret` mount will point to a directory that is also under the mount point for `private`.  That is, the `secret` path and the `private/secret` path will be the exact same directory.

Although this configuration won't cause any technical issues, it may be quite confusing so is generally not recommended.

## Checking the size of mounts

You can use standard commands such as `df -ah` to find the total disk usage of mounts (which are usually all on the same filesystem) and `du -hs /path/to/dir` to check the size of individual directories.

The CLI provides a command that combines these checks:

```
$ platform mount:size
Checking disk usage for all mounts of the application 'app'...
+-------------------------+-----------+---------+-----------+-----------+----------+
| Mount(s)                | Size(s)   | Disk    | Used      | Available | Capacity |
+-------------------------+-----------+---------+-----------+-----------+----------+
| private                 | 55.2 MiB  | 1.9 GiB | 301.5 MiB | 1.6 GiB   | 15.5%    |
| tmp                     | 34.1 MiB  |         |           |           |          |
| web/sites/default/files | 212.2 MiB |         |           |           |          |
+-------------------------+-----------+---------+-----------+-----------+----------+
```
