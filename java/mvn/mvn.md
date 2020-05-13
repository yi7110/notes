# 手动向mvn本地仓库添加com.alipay alipay-sdk 3.0.0 jar包  
```java 
mvn install:install-file -DgroupId=com.alipay -DartifactId=alipay-sdk -Dversion=3.0.0 -Dpackaging=jar -Dfile=d:/alipay-sdk-java-3.0.0.jar
```
> groupId:组名 artifactId:别名 version:版本号 file:文件路径 