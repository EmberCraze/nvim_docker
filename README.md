# nvim_docker

### TODO
- [x] Add git support inside container
- [ ] Allow container to start debugger inside another container
- [ ] Allow container to start other containers in the system

This repo describes the process of setting up nvim inside a docker invironment and allowing it to have both lsp and debugging capabilities

### Requirements
- Your own Astronvim user config

### Instructions
1. Place docker file inside desired python repository
2. Modify the docker file to be as close to your project setup as possible
3. Update the links to configs in docker file to point to your own config repo
4. Start your environtment
5. Ssh into your environtment and install gdb and debugpy
6. Before running the debugger, we need to add the following to the docker compose service that we want to debug:
    ```
    cap_add: 
        - SYS_PTRACE
    ```
6. Attach debugpy to the correct process with this command:
    ```
     python -m debugpy --listen 0.0.0.0:5678 --pid 19 --configure-subProcess False
    ```
7. Build the docker environment
    ```
    cd ~/your_project_path
    docker build -f DockerfileNvimAstro -t django_dev_env .
    ```
8. Run you docker environment
    ```
    docker run -it --network=host -e DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix -v ~/path_to_your_project:/home/dev/you_project_name django_dev_env:latest
    ```

Note: if you want to iterate over your astronvim user config without having to rerun the whole docker file everytime, you can run it with this command to partially invalidate the cache
```
docker build --build-arg CACHE_DATE=$(date) -f DockerfileNvimAstro -t django_dev_env .
```

For the debugging to work in nvim, the astronvim user config would need to have a file called `setup_handlers.lua` in the path `user/mason-nvim-dap/`

The file should contain the follow lua code:
```lua
local debug_port = 5678
local host = "0.0.0.0"

return {
  python = function(source_name)
    local dap = require("dap")
    dap.adapters.python = {
      type = "server",
      host = host,
      port = tonumber(debug_port),
    }

    dap.configurations.python = {
      {
        type = "python",
        request = "attach",
        connect = {
          port = tonumber(debug_port),
          host = host,
        },
        mode = "remote",
        name = "Remote Attached Debugger",
        cwd = vim.fn.getcwd(),
        pathMappings = {
            {
              localRoot = vim.fn.getcwd(), -- Wherever your Python code lives locally.
              remoteRoot = "/app", -- Wherever your Python code lives in the container.
            };
        };
      },
    }
  end,
}
```

Run `xhost local:root` on host to share clippboard
