---
author: Marc Falzon
layout: post
title: "My First Terraform Provider"
twitter: falzm
keywords: terraform,hashicorp,golang
---

The [Terraform](https://www.terraform.io/) Open Source community is very active, and has produced over the years a wide variety of *providers*. But what happens when you have a specific use case for which there are no existing provider yet? This article will illustrate how to create your very own Terraform provider.

Terraform is a plugin-based software written in [Go](https://golang.org/): the [core](https://github.com/hashicorp/terraform) is essentially responsible for the CLI interface, configuration file parsing and building a graph-based representation of the resources described in the configuration files. The actual provisioning work is actually performed by *[providers](https://github.com/terraform-providers/)* – external plugins with which the Terraform main processes interacts through standard interfaces.

For internal educational purposes at D2SI, we decided to create a simple yet functional plugin to introduce our fellow teammates to Terraform provider plugin development. Although HashiCorp already provides [a very good documentation](https://www.terraform.io/docs/plugins/basics.html) on this topic on the Terraform website, we've decided to open-source the code of our demo provider plugin and explain in this article how it works in a very practical way.

The demo provider plugin we've developed is named [filesystem](https://github.com/d2si-oss/terraform-provider-filesystem): it is able to provision single files and directories local to the host where Terraform is applied, similar to the official [local_file](https://www.terraform.io/docs/providers/local/r/file.html) provider.

## Plugin Structure

As mentioned before, our provider plugin is purposefully trivial. It is composed of the following files:

* `main.go`: plugin binary main entry point (i.e. the `main` function)
* `filesystem/`: Go package containing the actual provisioning-related provider code
  * `provider.go`: [ResourceProvider](https://godoc.org/github.com/hashicorp/terraform/terraform#ResourceProvider) interface implementation
  * `resource_filesystem_directory.go`: *filesystem_directory* provider resource type-related code
  * `resource_filesystem_file.go`: *filesystem_file* provider resource type-related code

Let's now dive into the actual code implementing our provider plugin.

*Note: this article has been written based on the version [v0.1.0](https://github.com/d2si-oss/terraform-provider-filesystem/tree/v0.1.0) of the provider, the actual open source code is subject to changes.*

### main.go

The entry point code is quite straightforward: after importing the standard `terraform/plugin` and our own `filesystem` packages, we instantiate the plugin listener pointing to our `filesystem.Provider()` initialization function (implemented in the `filesystem/provider.go` file):

```golang
package main

import (
    "github.com/d2si-oss/terraform-provider-filesystem/filesystem"
    "github.com/hashicorp/terraform/plugin"
)

func main() {
    plugin.Serve(&plugin.ServeOpts{
        ProviderFunc: filesystem.Provider})
}
```

### filesystem/provider.go

The `filesystem/` package directory contains the heavy-lifting code. The `filesystem/provider.go` starts with the declaration of a `filesystemProvider` structure type that will be used across the package to carry our internal-usage resources, such as a logging handler for debugging purposes:

```golang
package filesystem

import (
    ...

    "github.com/facette/logger"
    "github.com/hashicorp/terraform/helper/schema"
    "github.com/hashicorp/terraform/terraform"
)

type filesystemProvider struct {
    log *logger.Logger
}
```

The main attraction of this file is obviously the `Provider()` function, which returns a `schema.Provider` structure pointer implementing the `terraform.ResourceProvider` interface required by the Terraform core.

```golang
func Provider() terraform.ResourceProvider {
    return &schema.Provider{
        Schema: map[string]*schema.Schema{
            "debug": {
                Type:        schema.TypeBool,
                Description: fmt.Sprintf("Enable provider debug logging (logs to file %s)", providerLogFile),
                Optional:    true,
                Default:     false,
            },
        },

        ResourcesMap: map[string]*schema.Resource{
            "filesystem_directory": resourceDirectory(),
            "filesystem_file":      resourceFile(),
        },

        ConfigureFunc: config,
    }
}
```

The `schema.Provider` structure we're initializing and returning from this function contains 3 elements:

* `Schema` describes the provider's global configuration settings, which in our case is only one: `debug`, a boolean flag that controls whether the debug logging is enabled or not.
* `ResourcesMap` is a mapping of the provider's resources types to the function that will handle its provisioning. Our provisioner exposes 2 resources types: *filesystem_directory* and *filesystem_file*.
* `ConfigureFunc` is an optional function pointer allowing us to perform preliminary provider configuration, and which return value will be passed as metadata to each of the provider resources's [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) functions we'll cover soon. Here, we actually set it to the package-internal `config()` function.

The `config()` function evaluates the value of the `debug` configuration setting, and initializes the logging handler accordingly:

```golang
func config(d *schema.ResourceData) (interface{}, error) {
	var (
		p            filesystemProvider
		loggerConfig logger.FileConfig
		err          error
	)

	if d.Get("debug").(bool) {
		loggerConfig = logger.FileConfig{
			Level: "debug",
			Path:  providerLogFile,
		}
	}

	if p.log, err = logger.NewLogger(loggerConfig); err != nil {
		return nil, fmt.Errorf("unable to init provider debug logger: %s", err)
	}

	return p, nil
}
```

We're now getting to the interesting part of this journey: the actual resource provisioning implementation.

### filesystem/resource\_filesystem\_file.go

The first resource type to present is *filesystem_file*, which is described by the following `schema.Resource` structure returned by our `resourceFile()` function:

```golang
func resourceFile() *schema.Resource {
    return &schema.Resource{
        Schema: map[string]*schema.Schema{
            "path": {
                Type:        schema.TypeString,
                Description: "Path to the file to be created",
                Required:    true,
                ForceNew:    true,
            },
            "user": {
                Type:        schema.TypeString,
                Description: "File owner user name (default: current user)",
                Optional:    true,
                ForceNew:    false,
                DefaultFunc: func() (interface{}, error) {
                    currentUser, err := user.Current()
                    if err != nil {
                        return nil, fmt.Errorf("unable to lookup current user name: %s", err)
                    }
                    return currentUser.Username, nil
                },
            },
            "group": {
                Type:        schema.TypeString,
                Description: "File owner group name",
                Optional:    true,
                ForceNew:    false,
                DefaultFunc: func() (interface{}, error) {
                    currentUser, err := user.Current()
                    if err != nil {
                        return nil, fmt.Errorf("unable to lookup current user name: %s", err)
                    }

                    currentGroup, err := user.LookupGroupId(currentUser.Gid)
                    if err != nil {
                        return nil, fmt.Errorf("unable to lookup current user group name: %s", err)
                    }
                    return currentGroup.Name, nil
                },
            "mode": {
                Type:        schema.TypeString,
                Description: "Permissions to apply to file (in octal representation, e.g. 0644)",
                Optional:    true,
                Default:     "0644",
                ForceNew:    false,
                ValidateFunc: func(i interface{}, k string) (ws []string, errors []error) {
                    if _, err := strconv.ParseUint(i.(string), 8, 32); err != nil {
                        errors = append(errors, fmt.Errorf("%q: invalid value", k))
                    }
                    return
                },
            },
            "content": {
                Type:        schema.TypeString,
                Description: "File content",
                Optional:    true,
                Default:     "",
                ForceNew:    false,
                StateFunc: func(v interface{}) string {
                    return hash(v.(string))
                },
            },
        },

        Create: resourceFilesystemFileCreate,
        Read:   resourceFilesystemFileRead,
        Update: resourceFilesystemFileUpdate,
        Delete: resourceFilesystemFileDelete,
    }
}
```

A *filesystem_file* resource features the following attributes (described in the `Resource.Schema` element):

* `path` (required – type string): the path to the file to be created. Note the `ForceNew` flag set to `true`: changing this attribute value for an existing resource will result in the destruction then creation of a new resource.
* `user` (type string): an optional file owner user name, defaults to the current user running Terraform if not specified.
* `group` (type string): an optional file owner group name, defaults to the primary group of the current user running Terraform if not specified.
* `mode` (type string): optional UNIX permissions to apply to the file in octal representation, defaults to `"0644"` if not specified.
* `content` (type string): optional content to write to the file.

The second part of the resource description is a set of CRUD functions that will do the actual filesystem operations: the `Create` operation will be performed by the `resourceFilesystemFileCreate()` function, the `Read` operation will be performed by the `resourceFilesystemFileRead()` function, and so on...

Before digging into the functions called to perform the CRUD operations let's do a quick focus on notable resource attributes definitions.

Instead of a static default value, the `user`/`group` resource attributes use the `DefaultFunc` hook allowing the dynamic computation of the default value. If the user doesn't provide values for either of these attributes, the provider will look up the current user running Terraform and use its shell user name and group name as default values.

The `mode` attribute features a particular `ValidateFunc` hook that performs a simple sanity checking on the value provided by the user at parsing time, so that Terraform can return a potential syntax warning or error before actually trying to use the value during the provisioning – and crash during the `terraform apply` phase. This check is required because the value type for the file UNIX permissions is a string, as we need to keep the the leading `0` in `0644` when converting it to the Go's standard library [os.FileMode](https://godoc.org/os#FileMode): if we'd used a integer type for this attribute, parsing will strip the leading 0.

The `content` attribute also has a particularity regarding the use of the `StateFunc` hook: it allows us to change the value that will actually be stored in the Terraform [state](https://www.terraform.io/docs/state/index.html). As the file content may be of a significant length, instead of storing this attribute value as-is we hash it using the simple function `hash()` defined in the file `provider.go`, which returns the SHA-256 hash of a string:

```golang
func hash(s string) string {
    sha := sha256.Sum256([]byte(s))
    return hex.EncodeToString(sha[:])
}
```

This way, instead of storing the raw file content in the we only store the *hash* of the file content, which is sufficient to track content changes.

When applying a Terraform configuration the first operation to be performed is `Create`, which will call our `resourceFilesystemFileCreate()` function:

```golang
func resourceFilesystemFileCreate(d *schema.ResourceData, meta interface{}) error {
    p := meta.(filesystemProvider)

    p.log.Debug("calling resourceFilesystemFileCreate()")
```

This function – as for all other CRUD-related functions – is passed two arguments:

* a `*schema.ResourceData` containing the *filesystem_file* resource attributes parsed a the configuration file
* an "opaque" `meta` value, which in our case is actually the return value from the earlier execution of the `config()` function: we can safely cast this `interface{}` to the `filesystemProvider` type

```golang
    fileMode, _ := strconv.ParseUint(d.Get("mode").(string), 8, 32)
    d.Set("mode", fmt.Sprintf("%#o", os.FileMode(fileMode)))

    file, err := os.OpenFile(d.Get("path").(string), os.O_RDWR|os.O_CREATE, os.FileMode(fileMode))
    if err != nil {
        return err
    }
    defer file.Close()

    if _, err := file.WriteString(d.Get("content").(string)); err != nil {
        return err
    }

    u, err := user.Lookup(d.Get("user").(string))
    if err != nil {
        return fmt.Errorf("unable to lookup file owner user information: %s", err)
    }
    uid, _ := strconv.Atoi(u.Uid)

    g, err := user.LookupGroup(d.Get("group").(string))
    if err != nil {
        return fmt.Errorf("unable to lookup file owner group information: %s", err)
    }
    gid, _ := strconv.Atoi(g.Gid)

    if err := file.Chown(uid, gid); err != nil {
        return fmt.Errorf("unable to change file user/group: %s", err)
    }
```

At this point, we've created the file at the desired path, written its content if any and set its user/group/mode attributes according to the configuration values looked up using the `d.Get(<attribute name>)` method, and set the matching state values to be stored using the `d.Set(<attribute name>, <attribute  value>)` method.

For safety, we use the file path hashed value as resource unique identifier passed to the `d.SetId()` method:

```golang
    d.SetId(hash(file.Name()))

    return nil
}
```

The second operation to be implemented is `Read` – which calls our `resourceFilesystemFileRead()` function. It is used by Terraform before any other operation if a state exists for the resource, or in the case of imports.

```golang
func resourceFilesystemFileRead(d *schema.ResourceData, meta interface{}) error {
    p := meta.(filesystemProvider)

    p.log.Debug("calling resourceFilesystemFileRead()")

    fileInfo, err := os.Stat(d.Get("path").(string))
    if err != nil {
        if os.IsNotExist(err) {
            d.SetId("")
            return nil
        }

        return err
    }
    d.Set("mode", fmt.Sprintf("%#o", fileInfo.Mode()))
```

We start by looking up the file attributes using the system call `os.Stat()`: if an error indicating that it doesn't exist is returned, we unset its Terraform unique resource identifier to notify Terraform that the resource doesn't exist anymore and that it should be created; else, we set the current file `mode` attribute value to the actual one reported by the file system.

```golang
    fileContent, err := ioutil.ReadFile(d.Get("path").(string))
    if err != nil {
        return err
    }
    d.Set("content", hash(string(fileContent)))
```

Next, we read the file content and set the `content` resource attribute value to the hash of the content.

```golang
    u, err := user.LookupId(fmt.Sprintf("%d", fileInfo.Sys().(*syscall.Stat_t).Uid))
    if err != nil {
        return fmt.Errorf("unable to lookup file owner user information: %s", err)
    }
    d.Set("user", u.Username)

    g, err := user.LookupGroupId(fmt.Sprintf("%d", fileInfo.Sys().(*syscall.Stat_t).Gid))
    if err != nil {
        return fmt.Errorf("unable to lookup file owner group information: %s", err)
    }
    d.Set("group", g.Name)

    return nil
}
```

Finally, we look up the user and group file ownership and set the corresponding resource attributes as well.

The next operation implemented is `Update`, calling our function `resourceFilesystemFileUpdate()`:

```golang
func resourceFilesystemFileUpdate(d *schema.ResourceData, meta interface{}) error {
    p := meta.(filesystemProvider)

    p.log.Debug("calling resourceFilesystemFileUpdate()")

    file, err := os.OpenFile(d.Get("path").(string), os.O_RDWR, 0666)
    if err != nil {
        return err
    }
    defer file.Close()

    if d.HasChange("mode") {
        fileMode, _ := strconv.ParseUint(d.Get("mode").(string), 8, 32)

        if err := file.Chmod(os.FileMode(fileMode)); err != nil {
            return err
        }
    }

    if d.HasChange("user") || d.HasChange("group") {
        u, err := user.Lookup(d.Get("user").(string))
        if err != nil {
            return fmt.Errorf("unable to lookup file owner user information: %s", err)
        }
        uid, _ := strconv.Atoi(u.Uid)

        g, err := user.LookupGroup(d.Get("group").(string))
        if err != nil {
            return fmt.Errorf("unable to lookup file owner group information: %s", err)
        }
        gid, _ := strconv.Atoi(g.Gid)

        if err := file.Chown(uid, gid); err != nil {
            return fmt.Errorf("unable to change file user/group: %s", err)
        }
    }

    if d.HasChange("content") {
        if err := file.Truncate(0); err != nil {
            return err
        }

        if _, err := file.WriteString(d.Get("content").(string)); err != nil {
            return err
        }
    }

    return nil
}
```

This function applies changes to resources attributes selectively if they have been detected using the `d.HasChange(<attribute name>)` method: it avoids re-applying file attributes systematically.

Finally, the last operation implemented is `Delete`, calling our `resourceFilesystemFileDelete()` function. It is by far the simplest of all, as it just deletes the file from disk:

```golang
func resourceFilesystemFileDelete(d *schema.ResourceData, meta interface{}) error {
    p := meta.(filesystemProvider)

    p.log.Debug("calling resourceFilesystemFileDelete()")

    return os.Remove(d.Get("path").(string))
}
```

That's about it: those 4 basic functions provide the core functionality for managing our `filesystem_file` resource.

### filesystem/resource\_filesystem\_directory.go

As the code for managing the `filesystem_directory` resource is functionally very similar to the code for managing the `filesystem_file` resource – some interesting code refactoring could be done here, actually –, in this section we'll only focus on the few minor differences.

```golang
func resourceDirectory() *schema.Resource {
    return &schema.Resource{
        Schema: map[string]*schema.Schema{
            ...
            "mode": {
                Type:        schema.TypeString,
                Description: "Permissions to apply to directory (in octal representation, e.g. 0755)",
                Optional:    true,
                Default:     "0755",
                ForceNew:    false,
                ValidateFunc: func(i interface{}, k string) (ws []string, errors []error) {
                    if _, err := strconv.ParseUint(i.(string), 8, 32); err != nil {
                        errors = append(errors, fmt.Errorf("%q: invalid value", k))
                    }
                    return
                },
                StateFunc: func(v interface{}) string {
                    // We serialize the permissions including 'directory mode' (e.g. `020000000755`) otherwise
                    // the internal format will always be found different from the state format (`0755`)
                    permBits, _ := strconv.ParseUint(v.(string), 8, 32)
                    return fmt.Sprintf("%#o", os.ModeDir|os.FileMode(permBits))
                },
            },
            "create_parents": {
                Type:        schema.TypeBool,
                Description: "Create parent directories as needed",
                Optional:    true,
                Default:     false,
                ForceNew:    false,
            },
        },

        Create: resourceFilesystemDirectoryCreate,
        Read:   resourceFilesystemDirectoryRead,
        Update: resourceFilesystemDirectoryUpdate,
        Delete: resourceFilesystemDirectoryDelete,
    }
}
```

The first difference at resource schema level is the `mode` attribute: besides obvious different default permissions for a directory (`0755` instead of `0644`), we had to specify a custom `StateFunc` to serialize the permissions including the Go standard library [os.ModeDir](https://godoc.org/os#ModeDir) bit right now, as when we'll later read the directory mode using the `os.Stat()` function the internal format (e.g. `020000000755`) would always be found different from the state format (e.g. `0755`), thus triggering a resource change even if the permissions didn't actually change on disk.

Also, instead of a `content` attribute the `filesystem_directory` resource features a `create_parents` attribute, which is a boolean flag controlling whether the provider should create missing parent directories to the target directory. It is used in the `resourceFilesystemDirectoryCreate()` function called by the `Create` operation:

```golang
func resourceFilesystemDirectoryCreate(d *schema.ResourceData, meta interface{}) error {

   ...

    if d.Get("create_parents").(bool) {
        if err := os.MkdirAll(d.Get("path").(string), os.FileMode(dirMode)); err != nil {
            return err
        }
    } else {
        if err := os.Mkdir(d.Get("path").(string), os.FileMode(dirMode)); err != nil {
            return err
        }
    }

   ...

}
```

As mentioned earlier, the code implementing the `filesystem_directory` is very similar to its `file` counterpart so we'll skip the details and move on the fun part: using our own provider.

## Building & Installing

Before being able to use our *filesystem* provider with actual Terraform configuration, we have to build the plugin binary and install it.

A GNU *Makefile* is provided with the code, so the build step is trivial:

```shell
$ make build
==> Checking that code complies with gofmt requirements...
go install
```

If everything went well, this should result with a `terraform-provider-filesystem` binary in your `$GOPATH/bin` directory:

```shell
$ ls -l $GOPATH/bin/terraform-provider-filesystem
-rwxr-xr-x 1 marc staff 18888500 Feb 23 11:57 /Users/marc/.go/bin/terraform-provider-filesystem
```

## Usage

This is it: we're ready to use our provider in a standard Terraform setup. Our test configuration is the following:

```hcl
## test.tf

provider "filesystem" {
  debug = true
}

resource "filesystem_directory" "test" {
  path = "/tmp/test/dir"
  user = "marc"
  group = "admin"
  mode = "0750"
}

resource "filesystem_file" "test" {
  path = "${filesystem_directory.test.path}/file"
  content = <<EOF

                              /\__/\
                             /`    '\
                           === 0  0 ===
                             \  --  /
                            /        \
                           /          \
                           |           |
                           \  ||  ||  /
                            \_oo__oo_/#######o

EOF
}
```

If everything goes according to the (Terraform) plan, we should end up with two resources:

* a directory `/tmp/test/dir` owned by user `marc` and group `admin` with permissions `0750`
* a file `/tmp/test/dir/file` with no specific user/group ownership or permissions, and a beautiful ASCII-art portrait of a cat for content

First, we initialize the Terraform environment giving Terraform the path for our newly created terraform-provider-filesystem binary:

```
$ terraform init -plugin-dir=$GOPATH/bin

Initializing provider plugins...

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Next, we run the `terraform plan` command to ensure that the resources will be created according to the configuration we've just described:

```
$ terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + filesystem_directory.test
      id:             <computed>
      create_parents: "false"
      group:          "admin"
      mode:           "020000000750"
      path:           "/tmp/test/dir"
      user:           "marc"

  + filesystem_file.test
      id:             <computed>
      content:        "491b3d14819e554aaa2c950d459c3e55b99690c2b132d9c681722135458416cf"
      group:          "staff"
      mode:           "0644"
      path:           "/tmp/test/dir/file"
      user:           "marc"


Plan: 2 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

Looks good: we can see in the command output that the `filesystem_directory.test` resource will be created with attributes specified in the configuration, and that the `mode` attribute has actually been altered by the provider as a result of the resource attribute custom `StateFunc` function described earlier. As expected, the `filesystem_file.test` resource's `content` attribute value has also been altered by the provider, with the actual content replaced by its hash; also, the `user`/`group` attributes have been dynamically set to my current shell user's because we haven't specified any in the resource configuration.

Now is the moment of truth, let's actually apply the Terraform configuration:

```
$ terraform apply -auto-approve
filesystem_directory.test: Creating...
  create_parents: "" => "false"
  group:          "" => "admin"
  mode:           "" => "020000000750"
  path:           "" => "/tmp/test/dir"
  user:           "" => "marc"

Error: Error applying plan:

1 error(s) occurred:

* filesystem_directory.test: 1 error(s) occurred:

* filesystem_directory.test: mkdir /tmp/test/dir: no such file or directory

Terraform does not automatically rollback in the face of errors.
Instead, your Terraform state file has been partially updated with
any resources that successfully completed. Please address the error
above and apply again to incrementally change your infrastructure.

```

Well, this is embarrassing... but was actually expected. This is a perfect case for using the `create_parents` attribute of the `filesystem_directory` resource, as we've instructed Terraform to create a directory `/tmp/test/dir` but the parent directory `/tmp/test` doesn't exist. Our updated resource configuration is now:

```hcl
resource "filesystem_directory" "test" {
  path = "/tmp/test/dir"
  user = "marc"
  group = "admin"
  mode = "0750"
  create_parents = true
}
```

Let's try to apply Terraform now:

```
$ terraform apply -auto-approve
filesystem_directory.test: Creating...
  create_parents: "" => "true"
  group:          "" => "admin"
  mode:           "" => "020000000750"
  path:           "" => "/tmp/test/dir"
  user:           "" => "marc"
filesystem_directory.test: Creation complete after 0s (ID: f5aee067c62015c4883439792bf8d42139f0ee72f2802ee0d55af7bb6a676f21)
filesystem_file.test: Creating...
  content: "" => "491b3d14819e554aaa2c950d459c3e55b99690c2b132d9c681722135458416cf"
  group:   "" => "staff"
  mode:    "" => "0644"
  path:    "" => "/tmp/test/dir/file"
  user:    "" => "marc"
filesystem_file.test: Creation complete after 0s (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

Looks like it worked! To ensure it actually did, let's inspect the resources that have been created on disk:

```
$ tree /tmp/test
/tmp/test
└── dir
    └── file

$ stat /tmp/test/dir /tmp/test/dir/file
  File: /tmp/test/dir
  Size: 96            Blocks: 0          IO Block: 4194304 directory
Device: 1000004h/16777220d    Inode: 8831273     Links: 3
Access: (0750/drwxr-x---)  Uid: (  501/    marc)   Gid: (   80/   admin)
Access: 2018-02-23 14:48:16.866065822 +0100
Modify: 2018-02-23 14:48:16.869521077 +0100
Change: 2018-02-23 14:48:16.869521077 +0100
 Birth: 2018-02-23 14:48:16.866065822 +0100
  File: /tmp/test/dir/file
  Size: 362           Blocks: 8          IO Block: 4194304 regular file
Device: 1000004h/16777220d    Inode: 8831275     Links: 1
Access: (0644/-rw-r--r--)  Uid: (  501/    marc)   Gid: (   20/   staff)
Access: 2018-02-23 14:48:16.869512243 +0100
Modify: 2018-02-23 14:48:16.869603014 +0100
Change: 2018-02-23 14:48:16.869641454 +0100
 Birth: 2018-02-23 14:48:16.869512243 +0100
 
$ cat /tmp/test/dir/file

                              /\__/\
                             /`    '\
                           === 0  0 ===
                             \  --  /
                            /        \
                           /          \
                           |           |
                           \  ||  ||  /
                            \_oo__oo_/#######o

```

Success: the directory and the file have indeed been created according to our Terraform configuration.

As we've enabled debug logging by specifying `debug = true` in the provider configuration, let's have a look at the output `terraform-provider-filesystem.log` log file:

```
$ cat terraform-provider-filesystem.log
2018/02/23 14:47:56.305299 DEBUG: calling resourceFilesystemDirectoryCreate()
2018/02/23 14:48:16.865864 DEBUG: calling resourceFilesystemDirectoryCreate()
2018/02/23 14:48:16.869287 DEBUG: calling resourceFilesystemFileCreate()
```

We can see our first directory resource creation unsuccessful attempt, then the successful one immediately followed by the file resource creation.

Let's now update our Terraform configuration to specify a different group for the file resource:

```hcl
resource "filesystem_file" "test" {
  path = "${filesystem_directory.test.path}/file"
  group = "admin"
  content = <<EOF

                              /\__/\
                             /`    '\
                           === 0  0 ===
                             \  --  /
                            /        \
                           /          \
                           |           |
                           \  ||  ||  /
                            \_oo__oo_/#######o

EOF
}
```

We run `terraform plan` to see if Terraform detects the change:

```
$ terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

filesystem_directory.test: Refreshing state... (ID: f5aee067c62015c4883439792bf8d42139f0ee72f2802ee0d55af7bb6a676f21)
filesystem_file.test: Refreshing state... (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  ~ filesystem_file.test
      group: "staff" => "admin"


Plan: 0 to add, 1 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

It did, so we apply the new configuration and check the file to see it its group owner reflects the change:

```
$ terraform apply -auto-approve
filesystem_directory.test: Refreshing state... (ID: f5aee067c62015c4883439792bf8d42139f0ee72f2802ee0d55af7bb6a676f21)
filesystem_file.test: Refreshing state... (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)
filesystem_file.test: Modifying... (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)
  group: "staff" => "admin"
filesystem_file.test: Modifications complete after 0s (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.

$ stat /tmp/test/dir/file
  File: /tmp/test/dir/file
  Size: 362           Blocks: 8          IO Block: 4194304 regular file
Device: 1000004h/16777220d    Inode: 8831275     Links: 1
Access: (0644/-rw-r--r--)  Uid: (  501/    marc)   Gid: (   80/   admin)
Access: 2018-02-23 14:54:06.310992197 +0100
Modify: 2018-02-23 14:48:16.869603014 +0100
Change: 2018-02-23 14:54:06.310991959 +0100
 Birth: 2018-02-23 14:48:16.869512243 +0100
```

As you can see, the resource update went successfully; the log file also contains traces of the `terraform plan` and then the update function call:

```
2018/02/23 14:51:57.531546 DEBUG: calling resourceFilesystemDirectoryRead()
2018/02/23 14:51:57.534078 DEBUG: calling resourceFilesystemFileRead()
2018/02/23 14:54:06.298849 DEBUG: calling resourceFilesystemDirectoryRead()
2018/02/23 14:54:06.300984 DEBUG: calling resourceFilesystemFileRead()
2018/02/23 14:54:06.310831 DEBUG: calling resourceFilesystemFileUpdate()
```

If we modify the file state outside of Terraform and then apply Terraform again, it should pick up the changes according to its internal state:

```
$ echo > /tmp/test/dir/file

$ terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

filesystem_directory.test: Refreshing state... (ID: f5aee067c62015c4883439792bf8d42139f0ee72f2802ee0d55af7bb6a676f21)
filesystem_file.test: Refreshing state... (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  ~ filesystem_file.test
      content: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855" => "491b3d14819e554aaa2c950d459c3e55b99690c2b132d9c681722135458416cf"


Plan: 0 to add, 1 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.

```

As our provider stores the file content hash as `content` attribute value, executing `terraform plan` calls our `resourceFilesystemFileRead()` function that loads the current file content on disk then hashes it, and Terraform detects the change. We just have to apply Terraform again to get our file state according out the configuration:

```
$ terraform apply -auto-approve
filesystem_directory.test: Refreshing state... (ID: f5aee067c62015c4883439792bf8d42139f0ee72f2802ee0d55af7bb6a676f21)
filesystem_file.test: Refreshing state... (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)
filesystem_file.test: Modifying... (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)
  content: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855" => "491b3d14819e554aaa2c950d459c3e55b99690c2b132d9c681722135458416cf"
filesystem_file.test: Modifications complete after 0s (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.

$ cat /tmp/test/dir/file

                              /\__/\
                             /`    '\
                           === 0  0 ===
                             \  --  /
                            /        \
                           /          \
                           |           |
                           \  ||  ||  /
                            \_oo__oo_/#######o
```

We're about done, time to finish the experiment by running `terraform destroy` and see if our provider is able to destroy the resources it created:

```
$ terraform destroy -force
filesystem_directory.test: Refreshing state... (ID: f5aee067c62015c4883439792bf8d42139f0ee72f2802ee0d55af7bb6a676f21)
filesystem_file.test: Refreshing state... (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)
filesystem_file.test: Destroying... (ID: ea77e763acccecfd63e89cef535a799dfee4f9a67e606f512a03f568b5e419d2)
filesystem_file.test: Destruction complete after 0s
filesystem_directory.test: Destroying... (ID: f5aee067c62015c4883439792bf8d42139f0ee72f2802ee0d55af7bb6a676f21)
filesystem_directory.test: Destruction complete after 0s

Destroy complete! Resources: 2 destroyed.

$ ls /tmp/test
$
```

Mission complete: our first Terraform provider is ready.