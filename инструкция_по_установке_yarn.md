# BigData - инструкция по установке YARN на кластер

---

## 1. Архитектура YARN

- **ResourceManager (RM):** отвечает за управление ресурсами кластера, планирование заданий и распределение ресурсов. Запускается на мастер-ноде (team-76-nn).
- **NodeManager (NM):** отвечает за запуск контейнеров на узлах, мониторинг использования ресурсов и передачу статистики RM. Запускается на всех нодах кластера (team-76-nn, team-76-dn-00, team-76-dn-01).
- **History Server (HS):** хранит историю выполненных MapReduce заданий (JobHistory) и предоставляет веб-интерфейс для их просмотра. Запускается на мастер-ноде (team-76-nn).

---

## 2. Настройка конфигурационных файлов

### 2.1. Файл `yarn-site.xml`

На мастер ноде (team-76-nn) от имени пользователя **hadoopuser** откройте файл:

```bash
vim $HADOOP_HOME/etc/hadoop/yarn-site.xml
```

Замените его содержимое или добавьте следующий блок:

```xml
<configuration>
  <!-- Настройки ResourceManager -->
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>team-76-nn</value>
  </property>
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>team-76-nn:8050</value>
  </property>
  <property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>team-76-nn:8030</value>
  </property>
  <property>
    <name>yarn.resourcemanager.resource-tracker.address</name>
    <value>team-76-nn:8031</value>
  </property>
  <property>
    <name>yarn.resourcemanager.admin.address</name>
    <value>team-76-nn:8033</value>
  </property>

  <!-- Веб-интерфейс ResourceManager -->
  <property>
    <name>yarn.resourcemanager.webapp.address</name>
    <value>team-76-nn:8088</value>
  </property>

  <!-- Настройки NodeManager -->
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
  <property>
    <name>yarn.nodemanager.env-whitelist</name>
    <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
  </property>
</configuration>

```

> **Важно:** Используйте имена хостов, указанные в файле `/etc/hosts`. Таким образом, ResourceManager будет слушать на `team-76-nn`, а все ноды смогут корректно с ним взаимодействовать.

### 2.2. Файл `mapred-site.xml`

Для настройки History Server и параметров MapReduce (JobHistory) необходимо сконфигурировать файл `mapred-site.xml`.

1. Если файла `mapred-site.xml` ещё нет, скопируйте шаблон:
   ```bash
   cp $HADOOP_HOME/etc/hadoop/mapred-site.xml.template $HADOOP_HOME/etc/hadoop/mapred-site.xml
   ```

2. Отредактируйте файл:
   ```bash
   vim $HADOOP_HOME/etc/hadoop/mapred-site.xml
   ```

3. Добавьте (или измените) следующие свойства:

   ```xml
    <configuration>
    <!-- Использование YARN в качестве фреймворка для MapReduce -->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>

    <!-- Задаем classpath для MapReduce приложений -->
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_HOME/share/hadoop/mapreduce/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>

    <!-- Настройки History Server для хранения и отображения истории заданий -->
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>team-76-nn:10020</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>team-76-nn:19888</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.done-dir</name>
        <value>/home/hadoopuser/hadoop-3.4.1/dfs/data/history/done</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.intermediate-done-dir</name>
        <value>/home/hadoopuser/hadoop-3.4.1/dfs/data/history/intermediate</value>
    </property>
    </configuration>
   ```

4. Создайте указанные директории для History Server (на мастер-ноде, team-76-nn):
   ```bash
   mkdir -p /home/hadoopuser/hadoop-3.4.1/dfs/data/history/done
   mkdir -p /home/hadoopuser/hadoop-3.4.1/dfs/data/history/intermediate
   ```

---

## 3. Распространение конфигурации

Чтобы все ноды имели идентичные настройки, скопируйте файлы `yarn-site.xml` и `mapred-site.xml` с мастера (team-76-nn) на DataNode-ы (team-76-dn-00 и team-76-dn-01):

```bash
scp $HADOOP_HOME/etc/hadoop/yarn-site.xml hadoopuser@team-76-dn-00:$HADOOP_HOME/etc/hadoop/
scp $HADOOP_HOME/etc/hadoop/yarn-site.xml hadoopuser@team-76-dn-01:$HADOOP_HOME/etc/hadoop/

scp $HADOOP_HOME/etc/hadoop/mapred-site.xml hadoopuser@team-76-dn-00:$HADOOP_HOME/etc/hadoop/
scp $HADOOP_HOME/etc/hadoop/mapred-site.xml hadoopuser@team-76-dn-01:$HADOOP_HOME/etc/hadoop/
```

---

## 4. Запуск YARN демонов

### 4.1. Запуск ResourceManager и NodeManager

На мастер-ноде (team-76-nn) от имени **hadoopuser** выполните:
```bash
$HADOOP_HOME/sbin/start-yarn.sh
```
Эта команда запустит:
- **ResourceManager** на team-76-nn.
- **NodeManager** на team-76-nn (при условии, что он настроен в файле workers, как и ранее).

### 4.2. Запуск History Server

Запустите History Server (JobHistory Server) на мастере:
```bash
$HADOOP_HOME/sbin/mr-jobhistory-daemon.sh start historyserver
```

---

## 5. Проверка запущенных процессов

На каждой ноде выполните команду:
```bash
jps
```

**Ожидаемые процессы:**

- **team-76-nn (мастер):**
  - `ResourceManager`
  - `NodeManager`
  - `SecondaryNameNode` (и другие HDFS демоны, если запущены)
  - `JobHistoryServer`
  
- **team-76-dn-00 и team-76-dn-01 (DataNode):**
  - `NodeManager`
  - `DataNode`

---

## 6. Публикация веб-интерфейсов демонов YARN

Чтобы веб-интерфейсы были доступны извне:

1. **Убедитесь, что в конфигурационных файлах используются публичные имена нод**  
   (например, `team-76-nn` вместо `localhost`). Это гарантирует, что службы будут слушать на нужном IP.

2. **Откройте необходимые порты в firewall (если используется).**

Например, если на ваших нодах используется **ufw**, выполните:

- На мастере (team-76-nn):
  ```bash
  sudo ufw allow 8088/tcp    # ResourceManager web-интерфейс
  sudo ufw allow 19888/tcp   # History Server web-интерфейс
  sudo ufw allow 8042/tcp    # Если на мастере запущен NodeManager
  ```
- На DataNode-ах (team-76-dn-00 и team-76-dn-01):
  ```bash
  sudo ufw allow 8042/tcp    # NodeManager web-интерфейс
  ```

Если используется другой firewall, откройте соответствующие порты:
- **ResourceManager:** 8088
- **NodeManager:** 8042
- **History Server:** 19888

---

## 7. Проверка веб-интерфейсов

Откройте веб-браузер и проверьте доступность следующих адресов:

- **ResourceManager:**  
  [http://team-76-nn:8088](http://team-76-nn:8088)

- **History Server:**  
  [http://team-76-nn:19888](http://team-76-nn:19888)

- **NodeManager:**  
  Например, для team-76-dn-00: [http://team-76-dn-00:8042](http://team-76-dn-00:8042)  
  и для team-76-dn-01: [http://team-76-dn-01:8042](http://team-76-dn-01:8042)

Если какая-либо страница не открывается:
- Проверьте, что соответствующий демон запущен (используйте `jps`).
- Убедитесь, что файлы конфигурации правильно настроены и содержат публичные имена.
- Проверьте, что firewall не блокирует доступ к указанным портам.

---

## 8. Логи и отладка

Логи YARN демонов располагаются в каталоге:
```bash
$HADOOP_HOME/logs
```

Примеры команд для просмотра логов:

- **ResourceManager:**
  ```bash
  tail -n 50 hadoop-hadoopuser-resourcemanager-team-76-nn.log
  ```

- **NodeManager (например, на team-76-dn-00):**
  ```bash
  tail -n 50 hadoop-hadoopuser-nodemanager-team-76-dn-00.log
  ```

- **History Server:**
  ```bash
  tail -n 50 hadoop-hadoopuser-jobhistoryserver-team-76-nn.log
  ```

Для мониторинга в реальном времени используйте `tail -f`.
