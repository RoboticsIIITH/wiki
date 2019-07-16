# Working with servers

## Working with `modules`

Using environment modules is a common practise to maintain different versions of the same software without any conflicts on the server.

- You can use `module avail` to list the available modules on the server.
- Use `module load <module>` to load a module.
- Use `module list` to see the modules that you have loaded so far.
- Use `module unload <module>` to remove a loaded module.

### CUDA

Different versions of CUDA are available on the servers. You can load your preferred CUDA version using `modules`.

For example, if you want to load `cuda/9.0`, then

```bash
mrd@neon:~$ module avail
--------------------------- /opt/Modules/modulefiles ---------------------------
blender/2.79b  cudnn/5.1-cuda-8.0   gcc/5.5          torch
cuda/9.0       cudnn/7.3-cuda-9.0   matlab/R2018b
cuda/10.0      cudnn/7.4-cuda-10.0  matlab/RRC2018b
```
This will return the available modules on the server.

```bash
mrd@neon:~$ module load cuda/9.0
```

If you have loaded `cuda/9.0` and want to change to `cuda/10.0`,
```bash
mrd@neon:~$ module unload cuda/9.0
mrd@neon:~$ module load cuda/10.0
```

**Note**: Loading just `cuda` or just `cudnn` is not sufficient. Both of them should be loaded independently.

Please drop a mail to rrc-sysads@lists.iiit.ac.in in case you need a different CUDA version. We will add it for you on the server.
