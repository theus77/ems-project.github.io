# Storages

There are many types of file storages supported such as regular file system, asset saved in database, S3 and so forth. It's also possible to extend this list.

To configure the storages of your solution you should  configure the bundle parameter `ems_common.storages` (or via the environment (.env) `EMS_STORAGES` variable if you are using elasticms or the skeleton). This variable must contain a JSON array listing all storage services that you want ordered. I.e.:

```yaml
EMS_STORAGES='[{"type":"fs","path":"./var/assets"},{"type":"fs","path":"./var/assets2"}]'
```

All items must at least contain a `type` attribute. This attribute specifies the type of the storage adapter to instanciate.
 
The other attributes depend on the type of service chosen.

## Priorities and usages

### Read priorities

When the system is looking for a file, based on a hash, the system is looking to the adapters in their order in the `EMS_STORAGES` environment variable. You should think about this when you are defining your storage services. In terms of storage's performances and probablity to find files by sources.  

### Write usages

Each storage adapter can be defined with a `usage` attribute. This attribute can have one of the following option:
 - `cache` : refers to files generated by the storage manager
 - `config` : refers to files generated by the `ems_asset_path`  
 - `asset` : refers to files uploaded in the admin interface
 - `backup` : refers to files synchronized by the ems:asset:synchronize` command
 - `external` : refers to files handled by an external data source. I.e. the S3 admin storage from a skeleton.
 
 So, if a want to upload an asset, it will be saved on cache, config and cache storages, And config will be saved on cache and config storages. Externals storages act like a read-only storage.
 
For a regular website (1 elasticms and 1 skeleton) a best practice is to defined 4 storages:
 - A `config` storage for the skeleton : typically in an empty directory or in the docker images. As `config` and `cache` files doesn't have to be persisted.
 - A `config` storage for the admin : typically in an empty directory or in the docker images. As `config` and `cache` files doesn't have to be persisted.
 - An S3 bucket:
    - with the attribute `external` in the skeleton config
    - with the attribute `asset` in the elasticms config
 - A SFTP `backup` in the elasticms config and schedule a `ems:asset:synchronize`
 
 With this configuration you won't fill you storages with useless files.
 
 If you have a performant enough backup storage, you can just define it as `asset`. No need to schedule the `ems:asset:synchronize` commend. But a backup storage should never be configured as external (i.e. on a Skeleton).  

### Hot synchronize limit

An `hot-synchronize-limit` attribute can be specified. It's a file size attribute (`10M`,`1G`) that force synchronize missing assets (smaller than the limit) on more prioritized storage services. It's useful if you intend to use a volatile storage with high performances or if you interconnect environments or in development combine with a http storage service. 


## Cleaning useless assets
With the time the size of your storages can increase. For security reason, it's not possible to cleaned them from the application. But, there is a procedure.

With the EMSCoreBundle command `ems:asset:clean` you can remove from the database all files references that are not used by at least one revision. Typically, it cleans files that have been uploaded in a document but the or document has never been finalized, or the file has been replaced. 
 
Now if you define a new storage you'll be able to synchronize it with `ems:asset:synchronize`. This command is only synchronizing the referenced files.

Finally, it's time to remove your old storage from the `EMS_STORAGES` config. And, if you are sure this storage isn't used by another elasticms, you can drop it.
 
## Existing type of storages services

### File system
This service can be instantiated as many as you want and will use a regular folder to save/read assets. 
 - `type` (mandatory): `"fs"`
 - `usage` Default value: `"cache"`
 - `path` (mandatory): Path where to save assets
 
 Example:
 ```yaml
[
  {
    "type": "fs",
    "path": "/var/lib/ems"
  }
]
```
 
### Entity
This will save/read assets in the default relational database.
 - `type` (mandatory): `"db"`
 - `usage` Default value: `"config"`
 - `usage` Default value: `"cache"`
 
 Example:
 ```yaml
[
  {
    "type": "db"
  }
]
```
 
### HTTP
They will instantiate an HTTP service to read/save assets, typically an elasticms.
 - `type` (mandatory): `"html"`
 - `base-url` (mandatory): The base url (with scheme, protocol, ...) of your service i.e. `http://my-website.eu/admin`
 - `get-url` (optional): the relative url where to get asset by with a file's hash. Default value `/public/file/`
 - `auth-key` (optional): the authentication key to use in order to save asset. If not define the service will be read only.
 - `usage` Default value: `"backup"`, `external` if the `auth-key` is not defined
 
 Example:
 ```yaml
[
  {
    "type": "html",
    "base-url": "http://my-website.eu/admin",
    "auth-key": "MY-AUTH-KEY"
  }
]
```

 ### S3
The will instantiate a S3 client service to read/save assets in a S3 (or a s3-like i.e. minio) bucket. .
 - `type` (mandatory): `"s3"`
 - `credentials` (mandatory): S3 credential object
 - `bucket` (mandatory): Name of the bucket to use
 - `usage` Default value: `"cache"`
 - `upload-folder` Default value: `null`. As S3 doesn't support appending chunks the performances of this services decrease with large files. You may be interested to upload files in a local folder first. That folder must be shared between your instances or at least sessions must associate to one instance (i.e. via a sticky session cookie).  
 
 Example:
 ```yaml
[
  {
    "type": "s3",
    "bucket": "mybucket",
    "credentials": {
      "version": "2006-03-01",
      "credentials": {
        "key": "accesskey",
        "secret": "secretkey"
      },
      "region": "us-east-1",
      "endpoint": "http://localhost:9000",
      "use_path_style_endpoint": true
    }
  }
]
```


 ### SFTP
They will instantiate a SFTP client service to read/save assets on an SSH server. See the [PHP ssh2_auth_pubkey_file function documentation](https://www.php.net/manual/en/function.ssh2-auth-pubkey-file.php).
 - `type` (mandatory): `"s3"`.
 - `host` (mandatory): Host name or IP.
 - `path` (mandatory): Path to locate assets.
 - `username` (mandatory): User's login.
 - `public-key-file` (mandatory): Path to a public key. The public key file needs to be in OpenSSH's format. It should look something like: `ssh-rsa AAAAB3NzaC1yc2EAAA....NX6sqSnHA8= rsa-key-20121110`
 - `private-key-file` (mandatory): Path to a private key. 
 - `password-phrase` (optional): If `private-key-file` is encrypted, the passphrase must be provided.
 - `usage` Default value: `"backup"`
 
 Example:
 ```yaml
[
  {
    "type": "sftp",
    "host": "my-server.local",
    "path": "/var/lib/ems",
    "username": "user",
    "public-key-file": "/home/user/.ssh/id_rsa",
    "private-key-file": "/home/user/.ssh/id_rsa.pub",
    "password-phrase": "my-secret-key"
  }
]
```

## Extend with a specific storage type
If you want to implement your own storage service here is a quick tutorial.

Register a Symfony service implementing the `EMS\CommonBundle\Storage\Factory\StorageFactoryInterface` interface. The service must be tagged with `ems_common.storage.factory`:
```xml
    <service id="app.storage.ftp_factory" class="App\Storage\FtpFactory">
        <argument type="service" id="logger" />
        <tag name="`ems_common.storage.factory`" alias="ftp"/>
    </service>
```

This interface has 2 function to implement:
 - `public function createService(array $parameters): ?StorageInterface;` : This should return an instance of a class implementing the `EMS\CommonBundle\Storage\Service\StorageInterface`
 - `public function getStorageType(): string;`: this function should just return the string that you want to register as type for your storage service. I.e. `ftp`.

