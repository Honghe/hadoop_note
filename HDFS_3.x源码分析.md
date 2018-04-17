---
typora-root-url: image
typora-copy-images-to: image
---

# DFSClient

## DFSInputStream

Hadoop v3删除了buffersize，说是在此没用了，那buffersize这个功能由谁在哪接任？

https://github.com/apache/hadoop/commit/80d7d183cd4052d6e6d412ff6588d26471c85d6d

https://github.com/apache/hadoop/commit/80d7d183cd4052d6e6d412ff6588d26471c85d6d

```java
-  DFSInputStream(DFSClient dfsClient, String src, int buffersize, boolean verifyChecksum
+  DFSInputStream(DFSClient dfsClient, String src, boolean verifyChecksum
                  ) throws IOException, UnresolvedLinkException {
     this.dfsClient = dfsClient;
     this.verifyChecksum = verifyChecksum;
-    this.buffersize = buffersize;
     this.src = src;
     this.cachingStrategy =
         dfsClient.getDefaultReadCachingStrategy();
```

