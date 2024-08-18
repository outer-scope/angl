# **angl**

`angl` (short for disp**A**tchi**NG** uti**L**ity; derived from a german word: Angeln,) is a framework for dispatching and hooking shell scripts. It is a fun, opiniated, amateur hobby-project that exercizes software design and architecture. This project is substracted from [`virt-aid`](https://github.com/outer-scope/virt-aid).

## **1. Disclaimer**

Due to the unserious nature of this project, neither stability, security nor performance will be guaranteed.

## **2. Features & Usage**

### **2.1. Modular Design**

`angl` allows shell scripts to be implemented in a modular fashion to adhere to single responsibility principle. Scripts that perform as a single unified feature are grouped together to form a `module`.  Dispatchers are therefore introduced to handle centralized module calling mechanism, to reduce cohesion between them and to enforce extendability.

Imagine the following scenario:
- A script that mounts a block device, called repo, and
- A script that handles bootstraping, calles autostart.

Using `angl`, calling, or dispatching, these scripts can be denoted by:

```
# Calling repo.
dispatch repo

# Calling autostart.
dispatch autostart
```

### **2.2. Dispatching**

Dispatchers are utility scripts that call a module. A module can also call other modules by invoking the dispatch. Dispatchers would resemble, or be comparable to, an event bus, where a call is dispatched and all dependent modules subscribing to the event will respond.

`angl` `dispatch`:
1. Dispatches a command,
2. Hooks some subscribers to that command, and
3. If necessary, gives back (or resume) control to the initial `dispatch` caller.


To allow these mechanics, `angl` devises the following folder structure:

```
───[angl_root_folder]
   ┗──dispatch 
```

Revisiting our earlier examples, the structure would look like the following:

```
───[angl_root_folder]
   ├──[autostart]         : A module folder, named autostart, that contain autostart scripts.
   |  ┗──... 
   ├──[repo]              : A module folder, named repo, that contain repo scripts.
   |  ┗──... 
   ┗──dispatch 
```

For now, the `autostart` and `repo` modules would have a minimal structure like follows:

```
───[module_name]
   ┗──main                : A main entry for the module, so one can do: dispatch module_name 
                            instead of dispatch module_name/main 
```

### **2.3. Hooking**

From previous section, we learn that `angl` `dispatch` can also hook subscribers. So, let's say we extend our features:

- `repo` mounts a block device during host boot (`autostart`.)
- `autostart` dispatches

    ```
    dispatch autostart
    ```
    during boot using a `systemd` service.

If we were to write 

```
dispatch repo
```

within `dispatch autostart/main` script, we contaminate `autostart` with `repo` logic. Instead of doing this, we 'hook' all modules that subscribe to the event `autostart`. `repo` structure would now look like:

```
───[repo]
   ├──[autostart]
   |  ┗──main             : repo module subscribes to autostart/main execution event. 
   |                        Here lies the mounting logic.
   ┗──main 
```

Here is the complete structure as of now:

```
───[angl_root_folder]
   ├──[repo]
   |  ┗──[autostart]
   |     ┗──main 
   ┗──dispatch 
```

> **`?` Why is there no `autostart` module?**
>
> `dispatch` will look into modules for scripts to execute. But, if there is no need for a module to have executable scripts, that module doesn't have to exist. The 'hooking' mechanism of `dispatch` is to look for an existing event structure within modules and execute those. In our example, `autostart` doesn't have a main logic, and is therefore eliminated from the structure. All modules that subscribe to `autostart` event will be hooked nevertheless.

Let's extend the requirements evenmore:

- We have an 'installer' that installs:
    - `autostart` systemd service, and
    - registers `repo`'s block device in a config.
- We also have an 'uninstaller' that, when dispatched:
    - Removes `autostart` systemd service


Here is the complete structure as of the newest requirements:

```
───[angl_root_folder]
   ├──[autostart]
   |  ├──[install]
   |  |  ├──autostart.service      : A systemd service that executes `dispatch autostart` 
   |  |  ┗──main                   : Copies the service file into /etc/systemd and enables it.
   |  ┗──[uninstall]               
   |     ┗──main                   : Disables the service and removes the service file from /etc/systemd. 
   ├──[repo]
   |  ├──[autostart]
   |  |  ┗──main                   : Read the repo location config file in order to be able to mount the device.
   |  ┗──[install]
   |     ┗──main                   : Asks user to locate the block device and saves it into a config file. 
   ┗──dispatch 
```

From the above, we can:

```
# Install the features. The repo module will ask you to locate the block device.
dispatch install

# Uninstall the features.
dispatch uninstall
```

### **2.4. Resuming**

Let's say we want to dispose all installation files since we no longer need them post-install. `angl` introduces `dispatch` resuming ability. `dispatch` looks for a `resume-module` script under `module` and executes it after hooks complete.

```
───[angl_root_folder]
   ├──[autostart]
   |  ├──[install]
   |  |  ├──autostart.service       
   |  |  ┗──main                   
   |  ┗──[uninstall]               
   |     ┗──main                    
   ├──[install]
   |     ┗──resume-main              : Dispatch will give control back to this script after `dispatch install`.
   |                                   This script will remove all install/* files under each module.
   ├──[repo]
   |  ├──[autostart]
   |  |  ┗──main                   
   |  ┗──[install]
   |     ┗──main                    
   ┗──dispatch 
```

### **2.5. Dispatchers**

`angl` dispatchers are:
- `dispatch`
    - `dispatch` is an intermediary between modules. It also provides the ability to cascade calls, which can be seen effectively as a hooking mechanism. 
    ```
    angl/dispatch module-name
    # or
    angl/dispatch module-name/some-script
    ```
    > **⚠️ Warning**  
    > Due to the design, circular dispatch call is possible.
- `help`
    - A utility to `cat` a module's README.md file.
    ```
    angl/help module-name
    ```
- `helpers`
    - A utility to gather module's `helpers` file, which contains helper/util functions.
    ```
    source angl/helpers module-name
    ```
- `query`
    - A utility to query a module's shared variables.
    ```
    source angl/query module-name
    ```

### **2.6. `ANGL_PATH`, `SHARED_VAR`, and Dispatch Parameters**

Hooked modules utilize shared variables to execute command or to 'communicate' with each other. These are:

- `ANGL_PATH`
    - The `angl` location. So the dispatched script and the hooked modules can call

        ```
        $ANGL_PATH/dispatch module_name
        ```
        instead of
        ```
        /path/to/angl/dispatch module_name
        ```

- `SHARED_VAR`
    - A shared variable for dispatched script and the hooked modules to use to communicate between each other.
        ```
        # For example, set SHARED_VAR from some module to pass on to other modules.
        read guest_name
        SHARED_VAR=$guest_name
        ```

For the time being, hooked script will share the same parameter as the dispatched main executable script.

```
# If we dispatched some_module with
dispatch some_module param1 param2

# All the subsequent hooked scripts will receive param1 and param2 as well.
# If subsequent hooked scripts need to use some other variable, we use SHARED_VAR for this purpose.
```
