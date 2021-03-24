# Parches en Electron

Electron está construido sobre dos grandes proyectos principales: Chromium y Node.js. Cada uno de estos proyectos tienen también varias de sus propias dependencias. Hacemos todo lo posible para utilizar estas dependencias exactamente como están, pero a veces no podemos lograr nuestros objetivos sin parchear esas dependencias en upstream para que se ajusten a nuestros casos de uso.

## Justificación del parche

Cada parche en Electron es una carga de mantenimiento. Cuando el código original cambia, los parches pueden romperse, a veces incluso sin conflicto de parches o error de compilación. Es un esfuerzo continuo para mantener nuestro parche actualizado y efectivo. Así que nos esforzamos por mantener el número de parches como mínimo. Para ello, cada parche debe describir su razón de existencia en su mensaje de commit. Esa razón debe ser unas de las siguientes:

1. El parche es temporal, y está pensado para ser (o ha sido) confirmado a upstream o de lo contrario eventualmente eliminado. Incluye un enlace a un upstream PR o review de código si está disponible o a un procedimiento para verificar si el parche aún es necesario en una fecha posterior.
2. The patch allows the code to compile in the Electron environment, but cannot be upstreamed because it's Electron-specific (e.g. patching out references to Chrome's `Profile`). Include reasoning about why the change cannot be implemented without a patch (e.g. by subclassing or copying the code).
3. The patch makes Electron-specific changes in functionality which are fundamentally incompatible with upstream.

In general, all the upstream projects we work with are friendly folks and are often happy to accept refactorings that allow the code in question to be compatible with both Electron and the upstream project. (See e.g. [this](https://chromium-review.googlesource.com/c/chromium/src/+/1637040) change in Chromium, which allowed us to remove a patch that did the same thing, or [this](https://github.com/nodejs/node/pull/22110) change in Node, which was a no-op for Node but fixed a bug in Electron.) **Deberíamos enfocarnos en los cambios siempre que podamos y evitar los parches de duración indefinida**.

## Sistema de parches

If you find yourself in the unfortunate position of having to make a change which can only be made through patching an upstream project, you'll need to know how to manage patches in Electron.

All patches to upstream projects in Electron are contained in the `patches/` directory. Each subdirectory of `patches/` contains several patch files, along with a `.patches` file which lists the order in which the patches should be applied. Think of these files as making up a series of git commits that are applied on top of the upstream project after we check it out.

```text
patches
├── config.json   <-- this describes which patchset directory is applied to what project
├── chromium
│   ├── .patches
│   ├── accelerator.patch
│   ├── add_contentgpuclient_precreatemessageloop_callback.patch
│   ⋮
├── node
│   ├── .patches
│   ├── add_openssl_is_boringssl_guard_to_oaep_hash_check.patch
│   ├── build_add_gn_build_files.patch
│   ⋮
⋮
```

To help manage these patch sets, we provide two tools: `git-import-patches` and `git-export-patches`. `git-import-patches` importa un conjunto de archivos de parche en un repositorio git aplicando cada parche en el orden correcto y creando un commit para cada parche. `git-export-patches` hace el reverso; exporta una serie de commits git en un repositorio a un conjunto de ficheros en un directorio y un arvhivo `.patches` adjunto.

> Nota al margen: la razón por la que usamos un archivo `.patches` para mantener el orden de los parches aplicados, en lugar de anteponer un número como `001-` a cada archivo, es porque reduce los conflictos relacionados al orden de parches. It prevents the situation where two PRs both add a patch at the end of the series with the same numbering and end up both getting merged resulting in a duplicate identifier, and it also reduces churn when a patch is added or deleted in the middle of the series.

### Uso

#### Agregar un nuevo parche
```bash
$ cd src/third_party/electron_node
$ vim some/code/file.cc
$ git commit
$ ../../electron/script/git-export-patches -o ../../electron/patches/node
```

> **NOTA**: `git-export-patches` ignora cualquier archivo sin confirmar, así que debes crear un commit si quieres que tus cambios sean exportados. The subject line of the commit message will be used to derive the patch file name, and the body of the commit message should include the reason for the patch's existence.

La reexportación de parches ocasionalmente causará que cambien los shasums en parches no relacionados. Esto es generalmente inofensivo y puede ser ignorado (pero sigue adelante y agrega esos cambios a tu PR, esto evitará que aparezcan para otras personas).

#### Editar un parche existente
```bash
$ cd src/v8
$ vim some/code/file.cc
$ git log
# Find the commit sha of the patch you want to edit.
$ git commit --fixup [COMMIT_SHA]
$ git rebase --autosquash -i [COMMIT_SHA]^
$ ../electron/script/git-export-patches -o ../electron/patches/v8
```

#### Eliminar un parche
```bash
$ vim src/electron/patches/node/.patches
# Delete the line with the name of the patch you want to remove
$ cd src/third_party/electron_node
$ git reset --hard refs/patches/upstream-head
$ ../../electron/script/git-import-patches ../../electron/patches/node
$ ../../electron/script/git-export-patches -o ../../electron/patches/node
```

Note that `git-import-patches` will mark the commit that was `HEAD` when it was run as `refs/patches/upstream-head`. This lets you keep track of which commits are from Electron patches (those that come after `refs/patches/upstream-head`) and which commits are in upstream (those before `refs/patches/upstream-head`).

#### Resolviendo conflictos
When updating an upstream dependency, patches may fail to apply cleanly. A menudo, el conflicto puede ser resolvido automáticamente mediante git con un fusión de tres vías. Puedes indicar a `git-import-patches` que use el algoritmo de fusión de 3 vías pasando el argumento `-3`:

```bash
$ cd src/third_party/electron_node
# If the patch application failed midway through, you can reset it with:
$ git am --abort
# And then retry with 3-way merge:
$ ../../electron/script/git-import-patches -3 ../../electron/patches/node
```

If `git-import-patches -3` encounters a merge conflict that it can't resolve automatically, it will pause and allow you to resolve the conflict manually. Once you have resolved the conflict, `git add` the resolved files and continue to apply the rest of the patches by running `git am --continue`.