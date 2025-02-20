---
title: "JetBrains Gateway x Intellij Idea: File size exceeds configured limit"
date: 2025-02-20T18:19:00+08:00
tags: [jetbrains]
---

When there's a really large code file in the project and JetBrains' IDE may complains at the top of editor:

>File size exceeds configured limit (2560000). Code insight features not available.

Follow the steps below to solve the issue.

1. Edit custom properties on the remote machine (instead of your local client):
   Help -> Edit Custom Properties... (On Host)

<img src="https://s2.loli.net/2025/02/20/PEWGqdoSt97nTxB.png" alt="Edit Custom Properties (On Host)" style="zoom: 33%;" />

2. Add the following lines to the properties:

```properties
# Maximum file size (kilobytes) IDE should provide code assistance for.
idea.max.intellisense.filesize=60000

# Maximum file size (kilobytes) IDE is able to open.
idea.max.content.load.filesize=60000
```

3. The configuration in the **cache** still needs to be modified to take effect.
   Find `idea.properties` in the cache folder. On my machine, the location of the cached file is `$HOME/.cache/JetBrains/RemoteDev/dist/951f5b99b8987_ideaIU-2024.2.5/bin/idea.properties`.
4. Edit `idea.max.intellisense.filesize` and `idea.max.content.load.filesize` to let it finally work.

```properties
#---------------------------------------------------------------------
# Maximum file size (in KiB) IDE should provide code assistance for.
# The larger file is the slower its editor works and higher overall system memory requirements are
# if code assistance is enabled. Remove this property or set to very large number if you need
# code assistance for any files available regardless of their size.
#---------------------------------------------------------------------
idea.max.intellisense.filesize=60000

#---------------------------------------------------------------------
# Maximum file size (in KiB) the IDE is able to open.
#---------------------------------------------------------------------
idea.max.content.load.filesize=60000
```

5. Restart IDE.

