## Java in Docker

Исследование Java in Docker за авторством [jeckep][https://github.com/jeckep]

Problem:
Текущие настройки памяти java не позволяют надежно оценить потребляемую память, из за чего периодически docker убивает контейнер по OOM. ??? действительно ли ос убивает процесс? выяснить у админов и написать тест
Надо разобраться как настроить сервис, чтобы было возможно примерно рассчитать максимальный объем потребляемой памяти, и как этот объем будет изменяться при нагрузке
Investigation
0) http://trustmeiamadeveloper.com/2016/03/18/where-is-my-memory-java/
1) джава в контейнере не видит ограничений на память для контейнера
https://dzone.com/articles/java-and-memory-limits-in-containers-lxc-docker-an
2) Утилита для помогающая определить необходимые параметры запуска jvm: https://github.com/cloudfoundry/java-buildpack-memory-calculator
3) Еще инфа https://habrahabr.ru/company/ruvds/blog/324756/
https://habrahabr.ru/post/324918/
4) про CompressedClassSpaceSize: http://stackoverflow.com/questions/31075761/java-8-reserves-minimum-1g-for-metaspace-despite-maxmetaspacesize
5)
-XX:InitialCodeCacheSize=64M -XX:CodeCacheExpansionSize=1M -XX:CodeCacheMinimumFreeSpace=1M -XX:ReservedCodeCacheSize=200M
-XX:MinMetaspaceExpansion=1M -XX:MaxMetaspaceExpansion=8M -XX:MaxMetaspaceSize=200M
-XX:MaxDirectMemorySize=96M
-XX:CompressedClassSpaceSize=256M
-Xss1024K
This is an example from an app with -Xmx1445M -Xms1445M . The total consumption goes to about 2048M with these values in one case.
https://plumbr.eu/blog/memory-leaks/why-does-my-java-process-consume-more-memory-than-xmx
6) В контейнере от fabric8io никаких магических параметров не проставляется, только ставится Xmx = лимит на контейнер / 2, и число тредов под сборщик мусора равное числу выделенных процессоров (-XX:ParallelGCThreads=$
{core_limit} -XX:ConcGCThreads=${core_limit}
-Djava.util.concurrent.ForkJoinPool.common.parallelism=$
{core_limit}
)
7) http://blog.sokolenko.me/2014/11/javavm-options-production.html
8) описание опций java - https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html
Инструменты для анализа:
jps - инфо по процессам java
vmmap PID - инфо по сегментам памяти выделенной любому процессу(не только java)
jmap -heap PID - инфо по памяти java процесса
jcmd PID VM.native_memory summary(нужно при запуске процесса выставить параметр jvm: -XX:NativeMemoryTracking=summary)
Варианты решения проблемы:
1) не задавать никаких параметров для java процесса и дать jvm самой их выставить. В этом случае jvm в контейнере должна видеть ограничения по памяти контейнера, а не хоста. Хз как сделать, то ли это баг jvm, то ли это баг докера (это должны поправить в jdk9 http://hg.openjdk.java.net/jdk9/jdk9/hotspot/rev/5f1d1df0ea49)
1.1) выкинуть контейнеры
2) поставить все возможные параметры java процесса ограничивающие потребляемую память. Нужно определить по какой формуле от этих параметров высчитывается общее количество потребляемой памяти. Также нужно учитывать максимальное кол-во тредов создаваемых сервером приложений и -Xss параметр 
3) Сконфигурить своп и дать дохера памяти процессу
4) Использовать эксперементальные фичи или ждать java 10 (http://bugs.java.com/view_bug.do?bug_id=8170888)
Resolution(keycloak)
1) Поставить ограничение на количество тредов в jbos
2) обязательно выставить Xmx, Xss, XX:MaxMetaspaceSize, XX:CompressedClassSpaceSize(обязательно), XX:ReservedCodeCacheSize, XX:MaxDirectMemorySize
3) в качестве ограничения памяти на контейнер выставить Xmx + max(XX:MaxMetaspaceSize, XX:CompressedClassSpaceSize) + Xss*NumberOfThreads + MaxDirectMemorySize + N(под всякую ебалу: JIT code cache, classloaders, Socket Buffers (receive/send), JNI, GC ). N ~ 200M(чтоб наверняка хватило)
4) добавить -XX:NativeMemoryTracking=summary для точной статистики изпользования памяти ( note that enabling this will cause you a 5-10% overhead) http://hirt.se/blog/?p=401, можно еще добавить -XX:+PrintNMTStatistics -XX:+UnlockDiagnosticVMOptions чтобы при смерте процес вывел информацию о памяти
Native Memory Tracking
JVM option: -XX:NativeMemoryTracking=summary
How to get from java process: jcmd PID VM.native_memory summary
Section	Restriction option
Java Heap	-Xmx
Class	-XX:MaxMetaspaceSize, -XX:CompressedClassSpaceSize
Thread	-Xss
Code	-XX:ReservedCodeCacheSize
GC
Compiler
Internal
Symbol
Native Memory Tracking
Arena Chunk
